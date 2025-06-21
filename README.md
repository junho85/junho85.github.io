# junho85.github.io

개발자 김준호의 GitHub Pages 블로그

- GitHub Pages: https://pages.github.com/
- [Setting up a GitHub Pages site with Jekyll](https://docs.github.com/en/github/working-with-github-pages/setting-up-a-github-pages-site-with-jekyll)

## Jekyll 설치

https://jekyllrb.com/docs/installation/macos/

```bash
bundle install
bundle exec jekyll serve
```

## Docker로 Jekyll 실행하기

```bash
docker-compose up
```

브라우저에서 http://localhost:4000 으로 접속하여 확인할 수 있습니다.

### 백그라운드 실행

```bash
docker-compose up -d
```

### 중지하기

```bash
docker-compose down
```

## 배포 방법

GitHub Pages는 자동으로 배포됩니다. 다음 단계를 따르세요:

1. **변경사항 커밋하기**
```bash
git add .
git commit -m "Update site"
```

2. **GitHub에 푸시하기**
```bash
git push origin master
```

3. **배포 확인**
- 몇 분 후 https://junho85.github.io 에서 변경사항을 확인할 수 있습니다
- GitHub 저장소의 Settings > Pages 에서 배포 상태를 확인할 수 있습니다

### 배포 전 로컬 테스트

배포하기 전에 로컬에서 빌드가 제대로 되는지 확인하세요:

```bash
# Docker 사용
docker-compose up

# 또는 Jekyll 직접 사용
bundle exec jekyll build
bundle exec jekyll serve
```