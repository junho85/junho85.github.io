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