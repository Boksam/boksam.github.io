---
title: "Make an blog using Jekyll & Github Pages"
date: 2024-06-30
categories:
  - blog
tags:
  - jekyll
  - minimal-mistakes
---

# Overview
### Motivation
I used to have my blog on Velog, but I decided to switch to Jekyll because I wanted to have more control and customization options. Jekyll allows me to create a unique and cool blog that perfectly suits my needs.

### Select Theme
I selected the **minimal-mistakes** theme because I prefer a simple design and minimalism. This theme offers a cool and clean look that perfectly aligns with my preferences. There were many other themes available, but after careful consideration, I decided that **minimal-mistakes** was the best fit for my blog.

[Top 10 Jekyll Theme](https://jekyll-themes.com/blog/top-jekyll-themes)

# Installation
### 1. Install Ruby
I'm using mac now, so I installed Ruby using `brew`.

[Ruby Installation Guide](https://mac.install.guide/ruby/13)
![Ruby Version](/assets/images/diary/2024-06-30-blog-creation/ruby-install.png)
### 2. Make a repository for github page.
Repository name should be like `{username}.github.io`
![Repository Image](/assets/images/diary/2024-06-30-blog-creation/repo.png)

### 3. Clone theme and connect to my repository
Clone `minimal-mistakes` repository to my local.
```
git clone https://github.com/mmistakes/minimal-mistakes.git
```
Connect to my repository.
```
git remote remove origin
git remote add origin {repository address}
```

### 4. Remove unnecessary files & directories
Refer to Quick Start Guide for minimal-mistakes, unnecessary files are like below
- .editorconfig
- .gitattributes
- .github
- /docs
- /test
- CHANGELOG.md
- minimal-mistakes-jekyll.gemspec
- README.md
- screenshot-layouts.png
- screenshot.png

# Execute in Local Environment
### Dependency Settings
Before execute in local environment, change `Gemfile` like below to set **dependencies**.
```
source "https://rubygems.org"

gem "jekyll"
gem "minimal-mistakes-jekyll"

group :jekyll_plugins do
end
```

### Install Bundle
```
bundle install
```
![Bundle Install](/assets/images/diary/2024-06-30-blog-creation/bundle-install.png)

### Execute
```
bundle exec jekyll serve
```
![Local Execution](/assets/images/diary/2024-06-30-blog-creation/local-execute.png)
![Local Execution Result](/assets/images/diary/2024-06-30-blog-creation/result.png)

# Feelings after Installation
After successfully installing the minimal-mistakes theme and setting up my blog, I feel excited and motivated to start customizing it. The installation process was smooth and the theme looks great. I can't wait to explore all the customization options and make my blog truly unique.

I plan to continuously customize my blog to make it reflect my personal style and preferences. I will experiment with different layouts, colors, and features to create a blog that stands out. Stay tuned for updates as I embark on this exciting journey of blog customization!
