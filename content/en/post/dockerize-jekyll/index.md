+++
title = "🐳 How I Dockerized My GitHub Pages Jekyll Site — The Clean Setup That Works"
date = 2025-06-03
tags = ["jekyll", "docker", "github-pages", "devops"]
categories = ["DevOps"]
draft = false
+++

## 😩 The Problem

Setting up Jekyll with Docker sounds easy, but I ran into:

- platform issues (arm64 vs amd64) - I use Apple Silicon Macbook (M1)
- `bundle install` headaches

Since I was building this for my personal GitHub Pages site, I also had to make sure it stays compatible with GitHub Pages gem versions while being easy to develop locally.

## 🛠 My Clean Solution

I ended up building this Docker setup. It works for me at last.

### ✅ Key Features

- Works on both Apple Silicon (M1/M2) and Intel
- Clean rebuildable image
- Proper file sync with volume mounts
- Compatible with GitHub Pages Jekyll version
- Supports livereload (optional)

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

Gemfile.lock is also in the repo.

---

🙏 God bless Docker. May your builds be fast, your volumes mount correctly, and your ports never conflict. Good luck!


## 📦 Full repo

👉 [GitHub Repo Link] (https://github.com/namikimlab/namikimlab.github.io/tree/main)




