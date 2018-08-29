---
layout: post
current: post
cover:  assets/images/bg-codes.jpg
navigation: True
title: Travis-CI를 이용한 Jekyll 컴파일 및 배포
date: 2018-08-26 14:27:11 +0900
tags: [development]
class: post-template
subclass: 'post'
author: hwasub
---

정적 사이트 생성기(Static Site Generator)인 [Jekyll](https://jekyllrb.com/)을 이용하면 Markdown, HTML 등 다양한 Markup으로 만들어 둔 컨텐츠를 간단하게 컴파일하여 하나의 완성된 정적 사이트로 컴파일할 수 있다. 이렇게 생성된 정적 사이트는 DB나 스토리지 등 백엔드 자원을 사용하지 않으므로 웹서버만 있다면 간단하게 배포할 수 있으며, [GitHub Pages](https://pages.github.com/)와 같은 정적 사이트 호스팅 서비스를 이용하면 서버 구축에 대해 신경쓰지 않아도 호스팅이 가능하다.

특히 Jekyll은 GitHub Pages에서 기본 내장하고 있으므로 소스파일만 Push하면 자동으로 Deploy가 가능하다. 다만 GitHub Pages에 포함된 Jekyll은 safe mode에서 작동하여 다양한 플러그인을 활용하는 데 제한이 있다. 이러한 제한을 해제하기 위해 [Travis CI](https://www.travis-ci.org/)를 이용하여 GitHub 저장소에 Push된 소스파일을 자동으로 컴파일하여 다시 GitHub Pages에 배포하는 과정에 대해 알아보고자 한다.

# Travis CI

Travis CI는 이름에서 알 수 있다시피 Continuous Integration을 위한 툴이다. 이는 자동화된 테스트를 통해 일부 코드의 수정이 전체 프로젝트에 미치는 영향을 검토하여 소프트웨어의 전체적인 품질 관리를 꾀하는 것이자, 나아가 이러한 자동화된 테스트 이후 배포까지 자동으로 수행하여 배포 사이클을 줄이는 데 기여할 수 있다. Amazon의 설명에 따르면,

> 지속적 통합에서는 개발자가 Git와 같은 버전 관리 제어 시스템을 사용하여 공유된 리포지토리에 빈번하게 커밋하게 됩니다. 각 커밋에 앞서, 개발자는 통합 전에 추가 검증 계층으로써 코드에 로컬 유닛 테스트를 수행할 수 있습니다. 지속적 통합 서비스는 새로운 코드 변화에 대한 유닛 테스트를 자동으로 구축하고 실행하여 즉시 모든 오류를 표면화합니다. [Amazon, 지속적 통합이란 무엇입니까?](https://aws.amazon.com/ko/devops/continuous-integration/)

다만, 여기서는 이러한 거창한 작업보다는 단순한 빌드 및 빌드 후 배포 자동화 도구로서 활용할 예정이다.

# 파일 구조
{% highlight console %}
$ tree
.
├── _config.yml
├── .travis.yml
├── deploy.sh
├── Gemfile
├── Gemfile.lock
├── LICENSE
├── README.md
├── source
│   ├── ....
└── travisGemfile
{% endhighlight %}

이 글에서는 `.travis.yml`과 `deploy.sh` 파일의 구조에 대해 알아볼 것이다. Jekyll 사용에 대해서는 별도로 설명하지 않겠다.

## .travis.yml

{% highlight yaml linenos %}
language: ruby

cache:
  - bundler

gemfile: travisGemfile

before_script:
- chmod +x deploy.sh
- git config credential.helper "store --file=.git/credentials"
- echo "https://${GH_TOKEN}:@github.com" > .git/credentials

script: "./deploy.sh"

env:
  global:
    - NOKOGIRI_USE_SYSTEM_LIBRARIES=true
    - HTML_FOLDER="./output/"
    - secure: ...

branches:
  only:
  - master

notifications:
  email: false
{% endhighlight %}

기본적으로 Travis CI는 빌드 캐시를 지원한다. Travis에서는 모든 빌드 작업이 개별 VM 위에서 실행되고, 빌드 완료 후에는 해당 VM은 파기된다. 따라서 `bundler install`이나 `npm install`과 같은 패키지 매니저의 의존성 설치 작업을 실행할 경우 해당 패키지 설치 시간에 상당한 시간이 소요된다. 간단한 페이지를 컴파일하는 데 수 분이 걸릴 필요는 없다.

따라서 캐시 기능을 사용하자.

{% highlight yaml %}
cache:
  - bundler
{% endhighlight %}

해당 명령어는 Travis에 cache를 사용하라는 것과, bundler를 사용할 것임을 알려주고 있다. 해당 캐시 명령어가 있을 경우 Travis는 빌드 완료 후 `vendors/bundle` 디렉토리를 캐시한다 (따라서 해당 디렉토리 안에 민감한 정보가 저장되게 해서는 안 된다. Pull request도 동일한 캐시를 사용하여 빌드하므로, PR 권한이 있는 사용자가 해당 디렉토리에 접근할 수 있게 되는 결과를 낳는다). 

주의할 것은,

{% highlight yaml %}
install: bundler install
{% endhighlight %}

와 같이 별도의 install 명령어를 지정하는 경우, Travis가 cache할 수 있는 디렉토리에 vendor 파일이 저장되도록 하여야 한다. bundler install은 기본적으로 `~/.rvm/gems/ruby-x.x.x/gems/` 디렉토리에 패키지를 설치하므로, Travis가 캐시를 할 수 없다. `bundle install --path=vendor/bundle`와 같이 설치 위치를 지정하여야 한다. 물론 build 명령어를 별도로 지정하지 않으면 Travis가 적절하게 bundle 명령어를 실행하여 주므로, bundler 실행 시 별도의 옵션이 필요 없으면 지정하지 않아도 괜찮다.

{% highlight yaml %}
env:
  global:
    - NOKOGIRI_USE_SYSTEM_LIBRARIES=true
    - HTML_FOLDER="./output/"
    - secure: ...
{% endhighlight %}

이는 Travis가 빌드를 실행하는 환경에서 global하게 활용할 수 있는 변수이다. 주의할 점은, GitHub 토큰과 같이 중요한 정보는 `secure` 에다가 별도로 암호화하여 지정해야 한다. 그렇지 않으면 해당 저장소 접근 권한이나 Travis의 Build Log 접근 권한이 있는 사람 모두에게 해당 정보가 공개된다. secure에 지정하는 변수는 Travis가 발급한 공개키로 암호화되어 다른 사람(암호화한 본인도)은 해독할 수 없다.

Travis의 ruby CLI를 이용하여 암호화할 수도 있고, [한 개인이 만든 간단한 웹 도구](http://rkh.github.io/travis-encrypt/public/index.html)를 이용하여도 괜찮다. CLI를 이용한 암호화에 대해서는 [Travis CI Document - Encryption keys](https://docs.travis-ci.com/user/encryption-keys/)를 참고한다.

본 예제에서는 GitHub Repository에 push할 수 있는 권한이 있는 토큰(`$GH_TOKEN`)과 GitHub Repository 정보(`$GH_REF`)를 암호화하였다.

{% highlight yaml %}
before_script:
  - chmod +x deploy.sh
  - git config credential.helper "store --file=.git/credentials"
  - echo "https://${GH_TOKEN}:@github.com" > .git/credentials

script: "./deploy.sh"
{% endhighlight %}

Build를 위한 script는 script 변수에 설정한다. before_script 는 script를 실행하기 전에 실행되는 명령어들로, 해당 명령어에서 에러가 발생해도 build fail이 이루어지지 않으므로 환경설정 등에 활용할 수 있다.

여기서는 GitHub에 push하기 위한 토큰을 빌드 서버의 git credential helper에 저장하고 deploy.sh를 실행하도록 하였다.

## deploy.sh

{% highlight sh linenos %}
#!/usr/bin/env bash
set -e

# master's sha
export GIT_SHA=`git rev-parse --short HEAD`

# git clone from gh-pages
git clone --depth=2 --branch=gh-pages https://${GH_TOKEN}@github.com/${GH_REF} ${HTML_FOLDER}

# build
bundle exec jekyll build

# change workdir
cd ${HTML_FOLDER}

# config
git config --global user.email "lee@hwasub.com"
git config --global user.name "Hwasub Lee"

# git commit
git add --all
git commit -m "Deploy master:${GIT_SHA} on GitHub Pages"
git push --force --quiet "https://${GH_TOKEN}@github.com/${GH_REF}" gh-pages:gh-pages
{% endhighlight %}

이는 (1) GitHub Repository에서 `gh-pages` branch를 받아와서 (2) 빌드 서버에서 Jekyll build를 진행하고 (3) 빌드 서버 git에서 commit 한 후 (4) 다시 본래 GitHub Repository의 `gh-pages` branch에 push 하는 script이다.

필자는 `gh-pages` branch의 변경사항을 추적하기 위해 기존 source를 받아온 후 build를 진행하였으나, 이러한 변경사항 추적이 필요 없는 경우 받아 오는 과정 없이 `git init`를 진행하고 `git push --force ... master:gh-pages` 명령어를 이용해 강제로 push하여도 무방하다.

# Log

이제 GitHub의 소스 저장소의 master branch에 Jekyll 소스를 push 하면

![Build Log](/assets/images/2018/0826-build-log.jpg "Bulid Log")
Travis에서 Build가 실행되고,

![Push Log](/assets/images/2018/0826-push-log.jpg "Push Log")
GitHub gh-pages branch에 정상적으로 push된다.
