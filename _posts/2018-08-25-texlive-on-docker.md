---
layout: post
current: post
cover:  assets/images/bg-do-what-is-great.jpg
navigation: True
title: Docker and TeXLive
date: 2018-08-25 21:51:17 +0900
tags: [development]
class: post-template
subclass: 'post'
author: hwasub
---

TeX은 굉장히 편리한 도구이지만, 컴파일 환경을 구축하기에는 꽤 많은 품이 든다. 물론 Ubuntu 등 대부분의 배포판에서 TeX Live를 제공하므로 조금이나마 수고를 줄일 수 있겠으나, 최신 버전을 사용하는 데 한계가 있고 결정적으로 TeX Live 자체적인 패키지 관리자인 `tlmgr` 사용이 어려워 CTAN의 최신 패키지 활용이 어렵다.

그래서 TeX Live를 통채로 Dockerize한 패키지를 하나 만들어 두었다.

{% highlight console %}
$ docker pull minisena/docker-texlive-alpine
$ docker run -v `pwd`:/home --rm -it minisena/docker-texlive-alpine
...
# xelatex file
...
# exit
$ 
{% endhighlight %}

 * Dockerfile: <https://github.com/hwasub/docker-texlive-alpine>
 * Docker Hub: <https://hub.docker.com/r/minisena/docker-texlive-alpine>