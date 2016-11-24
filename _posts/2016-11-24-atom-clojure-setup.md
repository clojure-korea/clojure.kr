---
layout: post
title: Atom으로 Clojure 개발하기
date: 2016-11-24 19:00:00 +0900
author: yd.thx
---

> 참고자료: [Atom Clojure Setup](https://gist.github.com/jasongilman/d1f70507bed021b48625)  
> 누군가의 설정을 그대로 따라 해 봤습니다...

## 준비물
- [java](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
- [leiningen](http://leiningen.org/)
- [atom](https://atom.io/)

## 패키지 설치
- proto-repl
- proto-repl-charts
- ink
- tool-bar
- parinfer
- lisp-paredit
- highlight-selected
- set-syntax

## 패키지 세팅

### [language-clojure](https://gist.github.com/jasongilman/d1f70507bed021b48625#language-clojure)

+ Auto Indent: `✗`
+ Auto Indent On Paste: `✗`
+ Non Word Characters: `` ()"':,;~@#$%^&{}[]` ``
+ Scroll Past End: `✔︎`
+ Tab Length: 1


### [proto-repl](https://gist.github.com/jasongilman/d1f70507bed021b48625#proto-repl)

+ Auto Pretty Print: `✔︎`
+ Auto Scroll: `✔︎`
+ Display Executed Code In Repl: `✔︎`
+ Enable Completions: `✔︎`
+ Prefer Lein: `✔︎`
+ Refreshes... : `✔︎`
+ Show Inline Results: `✔︎`
+ Use Clojure Syntax: `✔︎`

 
### [lisp-paredit](https://gist.github.com/jasongilman/d1f70507bed021b48625#lisp-paredit)
+ Enabled: `✔︎`
+ Strict: `✗`
+ Indentation Forms: `` try, catch, finally, let, are, /^def.*/, fn, cond, if, if-let, for, /when.*/, testing, doseq, dotimes, loop, ns ``
+ Keybindings Enable: `✗`

## Atom 세팅
+ Auto Indent On Paste: `✗`
+ Scroll Past End: `✔︎`
+ (macOS) ~/.atom/ 적용 
    + init.coffee [download](https://gist.githubusercontent.com/jasongilman/d1f70507bed021b48625/raw/08feba06ce68faccfd44e4b7ee683e09879bd2f8/init.coffee) 
    + keymap.cson [download](https://gist.githubusercontent.com/jasongilman/d1f70507bed021b48625/raw/08feba06ce68faccfd44e4b7ee683e09879bd2f8/keymap.cson)

## demo

+ [공식 데모 프로젝트](https://github.com/jasongilman/proto-repl-demo)

### 프로젝트 추가
+ 화면 상단에 툴바가 보입니다.

![init](/assets/atom01.png)

### 단축메뉴

![menu](/assets/atom02.png)
![menu](/assets/atom03.png)

### 'Start REPL from ...' 실행

![repl](/assets/atom04.png)

### 'Execute Block' 실행
+ REPL에 코드를 보내서 실행하고 편집 화면에서도 결과가 표시됩니다.

![exec](/assets/atom05.png)







