+++
title = "🐳 GitHub Pages용 Jekyll 블로그를 Docker로 세팅하기 — 삽질 끝에 완성한 깔끔한 구성"
date = 2025-06-3T12:00:00+09:00
tags = ["jekyll", "docker", "github-pages", "devops"]
categories = ["DevOps"]
draft = false
+++

## 😩 문제

Jekyll을 Docker로 세팅하는 건 쉬워 보였지만, 실제로는 여러 문제를 겪었습니다:

- 플랫폼 이슈 (arm64 vs amd64) - Apple Silicon Macbook (M1)을 사용중
- `bundle install`에서 발생하는 오류들

개인 GitHub Pages 사이트용으로 만들고 있었기 때문에, GitHub Pages에서 사용하는 gem 버전과 호환되면서도 로컬에서 개발하기 쉽게 유지해야 했습니다.

## 🛠 나의 깔끔한 해결책

결국 이 Docker 세팅을 만들게 되었습니다. 저에게는 잘 작동합니다.


### ✅ Key Features

- Apple Silicon (M1/M2)와 Intel 모두에서 동작
- 깔끔하게 다시 빌드 가능한 이미지
- 볼륨 마운트를 통한 안정적인 파일 동기화
- GitHub Pages의 Jekyll 버전과 호환
- (선택 사항) livereload 지원

## 🐳 The Dockerfile

```dockerfile
FROM ruby:3.2.3-slim

RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev nodejs npm

RUN gem install bundler -v 2.6.9

WORKDIR /srv/jekyll

COPY Gemfile* ./

RUN bundle install

COPY . .

EXPOSE 4000

CMD ["bundle", "exec", "jekyll", "serve", "--host", "0.0.0.0", "--force_polling", "--livereload"]
```

## 🐳 docker-compose.yml

```yaml
services:
  site:
    image: my-jekyll
    platform: linux/arm64
    command: bundle exec jekyll serve --host 0.0.0.0 --force_polling --livereload
    ports:
      - "4000:4000"
      - "35729:35729"
    volumes:
      - .:/srv/jekyll
      - ./vendor/bundle:/usr/local/bundle
    working_dir: /srv/jekyll
    environment:
      - JEKYLL_ENV=development
```

## 💎 Gemfile
```ruby
source "https://rubygems.org"

gem "jekyll", "~> 4.3.3"
gem "csv", "~> 3.3.5"
gem "base64", "~> 0.2.0"
gem "logger", "~> 1.6.0"

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.17.0"
  gem "jekyll-seo-tag", "~> 2.8.0"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Lock `http_parser.rb` gem to `v0.6.x` on JRuby builds since newer versions of the gem
# do not have a Java counterpart.
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

# Add webrick as it's no longer bundled with Ruby
gem "webrick", "~> 1.8" 
```

Gemfile.lock도 레포에 포함되어 있습니다.

---

🙏 Docker에게 신의 가호가 있기를. 빌드는 빠르게, 볼륨은 정확히 마운트되고, 포트 충돌은 절대 발생하지 않기를. 행운을 빕니다!


## 📦 전체 레포

👉 [GitHub Repo Link] (https://github.com/namikimlab/namikimlab.github.io/tree/main)




