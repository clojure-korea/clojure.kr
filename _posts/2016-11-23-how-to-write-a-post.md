---
layout: post
title: how to write a post
date:   2016-11-22 15:09:10 +0900
author: minsun
---

## 마크다운 파일 생성하기

- post를 하려면 반드시 다은과 같은 네이밍 규칙을 지켜야한다.
- 리포 홈에서, `_posts` 하위에 `YYYY-MM-DD-name-of-post.md`의 파일을 만든다.
  - 예:  `_posts/2016-11-23-how-to-write-a-post.md`

### jekyll-compose 를 이용해 생성하기

- 현재 repo를 다운로드 한후 `$ bundle`명령어를 날리면 jekyll-compose 설치된다.
- 다음 명령어로 `_posts/2016-11-23-my-new-post.md ` 가 생성됨
```
$ bundle exec jekyll post "My New Post"
```
- 파일 명은 반드시 영어로 한다.
- https://github.com/jekyll/jekyll-compose 에 기타 사용법이 있다. (ex: draft생성)

## 마크다운 작성시에

위의 gem으로 파일을 생성하면, 다음과 같은 기본 레이아웃으로 마크다운이 생성된다.

```
layout: post
title: how to write a post
```

date와 author를 추가합니다.

```
layout: post
title: how to write a
date: 2016-11-22 15:09:10 +0900
author: minsun
```

title은 바꿔도 좋습니다.

```
layout: post
title: 포스팅 하는 법
date: 2016-11-22 15:09:10 +0900
author: minsun
```

깃헙 마크다운을 따릅니다, 변경사항이 필요하면 요청해주세요.
계층은 `##`입니다.

## 로컬에서 실행해보기

```
$ jekyll s
```

- 미리 볼 수 있습니다.

## publish

### 커밋 & 푸시

- 잠시 후 반영 됩니다.


### organization 소속이 아닐 경우

- pull-req를 날려주세요.
