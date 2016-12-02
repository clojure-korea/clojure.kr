---
layout: post
title: Leiningen Trampoline
date: 2015-03-24 00:00:50 +0900
author: Eunmin Kim
---

`lein run` 등으로 뭔가 동작시키면 `lein` 쉘 프로세스와 `leiningen.core.main` java 프로세스와 실제 어플리케이션 프로세스가 생성 된다.

`lein` 쉘 프로세스와 `leiningen.core.main` 프로세스는 프로덕션 환경에서는 불필요한 경우가 많기 때문에 `lein trampoline run`으로 실행하면 실제 어플리케이션 프로세스만 실행할 수 있다.

동작 방식에 대한 설명은 아래 블로그를 참조한다.
http://www.flyingmachinestudios.com/programming/lein-trampoline/
