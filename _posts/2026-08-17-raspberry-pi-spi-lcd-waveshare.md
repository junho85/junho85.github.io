---
layout: post
title: "라즈베리파이에 3.5인치 SPI LCD 붙이기 - 검색 결과가 다 안 통하는 이유"
date: 2026-08-17
tags: ["RaspberryPi", "SPI", "LCD", "waveshare", "ILI9486", "framebuffer", "ADS7846", "touch", "maker"]
image: /assets/images/2026-08-17-waveshare-35-lcd-board.jpg
---

집에 놀고 있던 라즈베리파이 3에 서랍에 있던 3.5인치 SPI LCD를 붙여봤습니다. 결과적으로 **`config.txt`에 한 줄만 넣으면 되는 일**이었는데, 그 한 줄을 찾기까지 검색 결과가 하나도 안 통했습니다.

이 글은 그 과정에서 걸린 함정 다섯 개의 기록입니다. 세 개는 제 코드 문제였고, 두 개는 **"인터넷의 안내가 지금은 유효하지 않은"** 종류였습니다.

![Waveshare 3.5inch RPi LCD (A) V3](/assets/images/2026-08-17-waveshare-35-lcd-board.jpg)

## 준비물과 환경

- **Raspberry Pi 3 Model B Rev 1.2** (1GB)
- **Waveshare(SpotPear) 3.5inch RPi LCD (A) V3** — ILI9486 컨트롤러, ADS7846 호환 저항막 터치
- Raspbian GNU/Linux 13 (trixie), 커널 6.18

![Raspberry Pi 3 Model B와 Waveshare 3.5인치 LCD](/assets/images/2026-08-17-waveshare-35-lcd-setup.jpg)

### 장착할 때 주의 — 헤더가 26핀입니다

이 LCD의 헤더는 **26핀(2×13)** 이라 Pi의 40핀 헤더 중 **1~26번에만** 꽂힙니다. 짧으니까 **어느 쪽으로도 꽂히는데**, 반드시 **1번 핀 쪽(microSD·전원 방향)에 끝을 맞춰야** 합니다. USB 쪽으로 밀어 꽂으면 GPIO 번호가 전부 어긋납니다.

전원도 미리 챙기는 게 좋습니다. LCD는 백라이트로 전류를 더 먹기 때문에, 저전압 경고가 있는 상태라면 화면이 안 나올 때 **드라이버 문제인지 전원 문제인지 구분이 안 됩니다.** 먼저 확인하세요.

```bash
vcgencmd get_throttled     # 0x0 이어야 정상
```

⚠️ 이 LCD는 **HDMI가 아니라 SPI** 방식입니다. HDMI 디스플레이처럼 꽂으면 알아서 되는 물건이 아니라, 커널에 "이 SPI 장치가 화면이다"라고 알려주는 device tree 오버레이가 반드시 필요합니다.

## 흰 화면은 고장이 아닙니다

꽂고 부팅하면 **하얀 화면**만 나옵니다. 처음엔 불량인가 했는데 아니었습니다.

이 제품은 **백라이트가 GPIO 제어 없이 상시 ON**입니다. 즉 백라이트는 켜졌는데 컨트롤러에 데이터가 안 들어간 상태고, 이게 **드라이버 미설정의 정상 증상**입니다. 흰 화면이 보이면 오히려 전원과 백라이트는 살아있다는 신호입니다.

## 함정 1. `waveshare35a` 오버레이가 이제 없습니다

검색하면 거의 모든 글이 이걸 시킵니다.

```
dtoverlay=waveshare35a
```

그런데 **최신 Raspberry Pi OS에는 이 오버레이 파일이 없습니다.** 직접 확인해보면 알 수 있습니다.

```bash
ls /boot/firmware/overlays/ | grep -i waveshare
# vc4-kms-dsi-waveshare-800x480.dtbo
# vc4-kms-dsi-waveshare-panel.dtbo
# waveshare-can-fd-hat-mode-a.dtbo
# ... waveshare35a.dtbo 는 없다
```

범용 `fbtft` 오버레이의 지원 목록에도 `waveshare32b`, `waveshare22`는 있지만 **35a는 빠져 있습니다.**

```bash
awk '/^Name:[[:space:]]*fbtft$/,/^Name:[[:space:]]*fe-pi/' /boot/firmware/overlays/README | grep waveshare
#         waveshare32b            Waveshare 3.2
#         waveshare22             Waveshare 2.2
```

그리고 벤더가 안내하는 `LCD-show` 스크립트는 **`config.txt`를 통째로 덮어쓰고** 옛 커널·X11 시절 가정을 밀어넣습니다. 게다가 그 스크립트가 기대하는 오버레이 자체가 없으니, 되돌리기 어려운 상태만 만들 위험이 큽니다. **실행하지 않는 쪽을 권합니다.**

## 정답은 `piscreen` 한 줄이었습니다

`overlays/` 목록을 뒤지다 **`piscreen.dtbo`** 를 발견했습니다. OzzMaker PiScreen용이지만 **ILI9486 + 저항막 터치**라는 조합이 같습니다.

핀이 맞는지는 추측하지 않고 커널 소스에서 확인했습니다.

```bash
# raspberrypi/linux 의 arch/arm/boot/dts/overlays/piscreen-overlay.dts
reset-gpios = <&gpio 25 GPIO_ACTIVE_LOW>;
dc-gpios    = <&gpio 24 GPIO_ACTIVE_HIGH>;
led-gpios   = <&gpio 22 GPIO_ACTIVE_HIGH>;
interrupts  = <17 2>;                      # 터치 IRQ = GPIO17
compatible  = "ilitek,ili9486";
spi-max-frequency = <24000000>;            # 표시 CS0, 터치 CS1은 2MHz
```

**RST=25 / DC=24 / BL=22 / 터치IRQ=17.** Waveshare 3.5(A) V3와 그대로 일치했습니다.

설정은 두 줄입니다.

```bash
sudo nano /boot/firmware/config.txt
```

```
dtparam=spi=on
dtoverlay=piscreen
```

⚠️ 경로가 `/boot/config.txt`가 아니라 **`/boot/firmware/config.txt`** 입니다. 그리고 `dtparam=spi=on`은 대개 `#`로 주석 처리된 채 이미 들어있으니 `#`만 지우면 됩니다.

재부팅하면 드라이버가 붙습니다.

```bash
dmesg | grep -i ili9486
# fb_ili9486 spi0.0: fbtft_property_value: rotate = 270
# graphics fb0: fb_ili9486 frame buffer, 480x320, 300 KiB video memory, fps=33, spi0.0 at 24 MHz

cat /proc/bus/input/devices | grep -A2 ADS7846
# N: Name="ADS7846 Touchscreen"
```

참고로 오버레이를 적용하면 **`/dev/spidev0.0`, `/dev/spidev0.1`이 사라집니다.** 이건 정상입니다 — raw SPI 장치를 드라이버가 인수한 것입니다.

## 함정 2. `/dev/fb1`이라는 가정이 깨집니다

인터넷 안내는 대체로 "LCD는 `/dev/fb1`이니 여기에 쓰면 된다"고 합니다. 저도 그렇게 스크립트를 짰고 잘 돌았습니다. **그런데 재부팅하니 아무것도 안 나왔습니다.**

```
1차 부팅: [ 9.891] vc4-drm ... fb0: vc4drmfb     →  LCD는 fb1
2차 부팅:  (vc4 프레임버퍼 등록 안 됨)          →  LCD가 fb0
```

프레임버퍼 번호는 **등록 순서로 정해지고**, vc4 KMS와 SPI 드라이버의 프로브 타이밍 경합에 따라 갈립니다. 증상이 고약한 이유는 **하드코딩한 스크립트가 에러 없이 조용히 실패**한다는 점입니다. 장치는 멀쩡한데 화면만 죽은 것처럼 보입니다.

번호가 아니라 **드라이버 이름으로 찾아야** 합니다.

```python
import glob, os

def find_fb(driver="fb_ili9486"):
    for path in sorted(glob.glob("/sys/class/graphics/fb*")):
        if open(f"{path}/name").read().strip() == driver:
            w, h = open(f"{path}/virtual_size").read().strip().split(",")
            return "/dev/" + os.path.basename(path), int(w), int(h)
    raise SystemExit(f"{driver} 프레임버퍼 없음")
```

해상도도 `virtual_size`에서 읽으면 회전 설정을 바꿔도 안전합니다. 셸에서 확인할 때는 이렇게 합니다.

```bash
grep -l fb_ili9486 /sys/class/graphics/fb*/name
```

### 가장 확실한 동작 검증

드라이버가 붙었는지는 노이즈를 써보면 즉시 압니다. 화면에 지지직 패턴이 뜨면 드라이버와 배선 모두 정상입니다.

```bash
# 307200 = 480 x 320 x 2바이트(RGB565)
sudo sh -c "head -c 307200 /dev/urandom > /dev/fb0"
```

## 프레임버퍼에 그리기 — RGB565 변환이 핵심

fbtft 프레임버퍼는 **16bpp RGB565**입니다. PIL로 그린 RGB 이미지를 그대로 쓰면 안 되고 변환이 필요합니다.

```python
import numpy as np
from PIL import Image, ImageDraw

img = Image.new("RGB", (480, 320), (18, 18, 24))
d = ImageDraw.Draw(img)
d.text((20, 20), "hello", fill=(255, 255, 255))

a = np.asarray(img, dtype=np.uint16)
rgb565 = ((a[:, :, 0] >> 3) << 11) | ((a[:, :, 1] >> 2) << 5) | (a[:, :, 2] >> 3)
with open("/dev/fb0", "wb") as fb:
    fb.write(rgb565.astype("<u2").tobytes())   # little endian
```

PIL과 numpy는 Raspberry Pi OS desktop 이미지에 기본 포함되어 있어 따로 설치할 게 없습니다. 프레임버퍼는 `root:video` 소유라 `sudo`를 쓰거나 사용자를 `video` 그룹에 넣습니다.

### 한글이 네모로 나온다면

DejaVu 폰트에는 **한글 글리프가 없습니다.** 한국 로케일로 설치했다면 `fonts-nanum`이 이미 깔려 있을 겁니다.

```bash
fc-list :lang=ko | head
# /usr/share/fonts/truetype/nanum/NanumGothic.ttf: NanumGothic,나눔고딕
```

```python
FONT = "/usr/share/fonts/truetype/nanum/NanumGothic.ttf"
MONO = "/usr/share/fonts/truetype/nanum/NanumGothicCoding.ttf"  # 숫자 폭 고정
```

HUD처럼 숫자가 실시간으로 바뀌는 자리에는 **고정폭(`NanumGothicCoding`)** 을 쓰면 글자가 흔들리지 않습니다.

## 터치가 훨씬 어려웠습니다

화면은 한 줄로 끝났는데, 터치에서 세 번 연속으로 틀렸습니다. `python3-evdev`가 없는 환경이라 `/dev/input/eventN`을 직접 파싱하기로 한 게 시작이었습니다.

먼저 터치 장치를 찾습니다.

```bash
for d in /sys/class/input/event*; do echo "$(basename $d) -> $(cat $d/device/name)"; done
# event2 -> ADS7846 Touchscreen
```

### 함정 3. `ABS_PRESSURE`가 항상 0입니다

압력으로 "눌림"을 판정했는데 전혀 반응하지 않았습니다. `xohms`(x-plate-ohms) 파라미터를 주지 않으면 ADS7846 드라이버가 **압력을 계속 0으로 보고**합니다. 이벤트 순서가 문제였습니다.

```
BTN_TOUCH=1  →  ABS_X  →  ABS_Y  →  ABS_PRESSURE=0  →  SYN_REPORT
```

`BTN_TOUCH=1`로 잡은 상태를 바로 뒤의 `ABS_PRESSURE=0`이 **"뗌"으로 덮어버립니다.** 눌림 판정은 **`BTN_TOUCH` 하나만 믿어야** 합니다.

### 함정 4. 짧은 탭은 한 번의 `read()`에 통째로 들어옵니다

`BTN_TOUCH`로 바꿨는데도 안 잡혔습니다. 저항막 터치를 톡 누르면 접촉이 수십 ms뿐이고, 그동안 **`BTN_TOUCH=1`부터 `=0`까지가 한 배치에 함께** 도착합니다. 배치를 다 처리하면 상태는 항상 "뗀 상태"라, `pressed`만 검사하는 코드는 **아무리 눌러도 영원히 못 봅니다.**

눌림 시작을 카운터로 latch해서 소비하는 방식으로 해결했습니다.

```python
if self.pressed and not was:
    self.tap_count += 1        # 눌림 시작을 기록

def take_tap(self):            # 호출자가 소비
    if self.tap_count:
        self.tap_count = 0
        return True
    return False

# 대기 조건은 "누르고 있음" 또는 "미소비 탭"
if touch.pressed or touch.take_tap():
    ...
```

단계가 여러 개인 화면(예: 좌/우 보정)에서는 다음 단계로 넘어갈 때 **남은 latch를 비워야** 합니다. 안 그러면 한 번 누른 게 두 단계를 동시에 통과합니다.

### 함정 5. `struct` 크기를 8바이트로 박으면 깨집니다 — 이게 진짜 원인

앞의 두 수정에도 안 되자 이벤트를 그대로 덤프해봤습니다.

```
batch#001 (3) 61466/27266=648498 ABS_Y=2200 61466/27266=648498
```

**타입·코드가 쓰레기 값입니다.** 구조체 크기가 어긋난 것입니다.

```c
struct input_event {
    struct timeval time;   // long 2개
    __u16 type; __u16 code; __s32 value;
};
```

`struct timeval`은 **long 2개**라서 크기가 userland 비트수를 따릅니다. 32비트는 16바이트, 64비트는 24바이트입니다. 저는 `"qqHHi"`(q = 고정 8바이트, 합 24바이트)로 박아뒀는데, 이 Pi는 32비트였습니다.

```bash
dpkg --print-architecture     # armhf
python3 -c "import platform; print(platform.machine())"   # armv7l
getconf LONG_BIT              # 32
```

**해법은 native long을 쓰는 것입니다.** 32/64비트 양쪽에서 자동으로 맞습니다.

```python
FMT = "llHHi"                  # 32bit → 16바이트, 64bit → 24바이트
SZ = struct.calcsize(FMT)
```

### 아키텍처를 커널로 판단하면 틀립니다

더 근본적인 오진이 있었습니다. 이미지 커널이 `6.18.34+rpt-rpi-v8`이고 **`v8`은 arm64**니까 64비트라고 단정했던 것입니다. 그런데 **Raspberry Pi OS 32-bit는 Pi 3 이상에서 64비트 커널을 로드**합니다. 커널 비트수와 userland 비트수는 별개입니다.

힌트가 하나 더 있었는데 놓쳤습니다. `/etc/os-release`의 이름이 갈립니다.

| userland | PRETTY_NAME |
|---|---|
| 32비트 | **Raspbian** GNU/Linux |
| 64비트 | **Debian** GNU/Linux |

아키텍처는 `uname -m`이 아니라 **userland 기준**(`dpkg --print-architecture`)으로 봐야 합니다.

## 터치 좌표는 회전 때문에 안 맞습니다

`rotate=270`으로 화면이 돌아가 있으니 터치 raw 좌표축과 화면축이 일치하지 않습니다. 값은 0~4095 범위입니다.

```bash
# EVIOCGABS 로 범위 확인
# ABS_X: min=0 max=4095 / ABS_Y: min=0 max=4095
```

어느 축이 화면 가로축인지 문서를 찾는 대신, **좌/우를 한 번씩 눌러보게 해서 직접 알아내는** 보정 단계를 넣었습니다. 더 크게 변한 축이 화면 가로축이고, 두 값의 부호로 방향까지 정해집니다.

![터치 보정 화면](/assets/images/2026-08-17-waveshare-35-lcd-touch-calibration.jpg)

제 경우 `axis=x, lo=774, hi=3400`이 나왔습니다. 결과를 JSON으로 저장해두면 다음 실행부터 건너뛸 수 있습니다.

```python
raw = touch.raw_x if axis == "x" else touch.raw_y
screen_x = min(max((raw - lo) / (hi - lo), 0.0), 1.0) * width
```

오버레이 파라미터로 보정하는 방법도 있습니다.

```
dtoverlay=piscreen,rotate=90            # 화면 방향 (0/90/180/270)
dtoverlay=piscreen,invx,invy,swapxy     # 터치 좌표 반전·교환
dtoverlay=piscreen,speed=16000000       # 화면이 깨지면 SPI 속도를 낮춘다
```

## 콘솔과 그림은 양립하지 않습니다

부팅 메시지와 로그인 콘솔을 LCD로 보내려면 `/boot/firmware/cmdline.txt`(**반드시 한 줄**) 맨 뒤에 추가합니다.

```
fbcon=map:10 fbcon=font:ProFont6x11
```

⚠️ 두 가지를 주의해야 합니다.

- 콘솔이 프레임버퍼를 차지하면 **직접 그리는 프로그램의 출력이 덮입니다.** 둘 중 하나를 골라야 합니다
- **`map:10`은 "fb1"을 가리키는 값**입니다. 앞서 본 것처럼 LCD가 fb0으로 잡히는 부팅에서는 엉뚱한 장치를 지목해 **콘솔이 아무 데도 안 나옵니다**

저는 상태 표시 화면으로 쓰기로 하고 `fbcon` 인자를 뺀 뒤, 스크립트를 systemd 서비스로 등록했습니다.

```ini
# /etc/systemd/system/lcd-status.service
[Unit]
Description=LCD status display
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/USER/lcd-status.py --loop
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now lcd-status
```

## 결과 — 똥피하기 게임

터치가 되니 게임을 만들어봤습니다. 떨어지는 똥을 터치로 좌우 이동해 피하는 게임입니다. **fps는 40 정도** 나왔습니다.

SPI 24MHz로 307KB 프레임을 초당 40번 보내는 건 대역폭상 불가능한데, **fbtft가 변경된 영역만 전송**하고 앱의 `write()`는 커널 버퍼 복사로 끝나기 때문입니다. 실제 화면 갱신은 드라이버의 `fps=33`이 상한입니다. 어쨌든 게임을 돌릴 만한 속도는 나옵니다.

## 정리

| 함정 | 증상 | 해결 |
|---|---|---|
| 흰 화면 | 아무것도 안 나옴 | 고장 아님. 백라이트 상시 ON + 드라이버 미설정 |
| `waveshare35a` 없음 | 오버레이 적용 실패 | **`dtoverlay=piscreen`** |
| `LCD-show` 스크립트 | config.txt 덮어씀 | 실행하지 않기 |
| fb 번호 하드코딩 | 재부팅 후 조용히 실패 | `/sys/class/graphics/fb*/name`으로 탐색 |
| `ABS_PRESSURE`=0 | 터치 무반응 | `BTN_TOUCH`만 신뢰 |
| 짧은 탭 유실 | 톡 눌러도 무반응 | 눌림 시작을 latch |
| `struct` 24바이트 가정 | 이벤트 파싱 깨짐 | **`"llHHi"`**(native long) |
| 커널로 아키텍처 판단 | 32/64비트 오판 | `dpkg --print-architecture` |

가장 값진 교훈은 **"검색 결과와 내 환경의 차이를 먼저 확인하라"** 였습니다. `waveshare35a`도 `/dev/fb1`도 예전엔 맞는 안내였고, 지금 이 이미지에서만 틀립니다. `ls /boot/firmware/overlays/`와 `dmesg` 한 번이면 확인되는 것이었는데, 검색 결과를 그대로 믿고 시작해서 시간을 썼습니다.

터치 쪽 세 함정은 전부 **"하드웨어는 정상인데 내 판정이 틀린"** 경우였습니다. `/proc/interrupts`의 IRQ 카운트가 올라가는 걸 먼저 확인해 하드웨어를 배제한 게 그나마 시간을 아꼈습니다.

```bash
grep ads7846 /proc/interrupts    # 터치할 때 카운트가 오르면 하드웨어 정상
```

추측으로 두 번 고치고 두 번 다 빗나간 뒤에야 이벤트를 그대로 덤프했습니다. **애초에 그것부터 했어야 했습니다.**
