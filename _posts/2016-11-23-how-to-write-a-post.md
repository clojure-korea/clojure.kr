---
layout: post
title: clojure.kr 블로그에 포스팅 하는법
date: 2016-11-22 15:09:10 +0900
author: dalzony
---

## github

-  [clojure-korea github] fork!

## 마크다운 파일 생성하기

- post를 하려면 반드시 다은과 같은 네이밍 규칙을 지켜야한다.
- 리포 홈에서, `_posts` 하위에 `YYYY-MM-DD-name-of-post.md`의 파일을 만든다.
  - 예:  `_posts/2016-11-23-how-to-write-a-post.md`
- 파일 명은 반드시 영어로 한다.

## 마크다운 룰

다음 내용은 반드시 넣어야합니다.

- layout
- title
- date
- author

### 기본 레이아웃

```
---
layout: post
title: how to write a post
---
```

### date와 author 추가

- date와 author를 추가합니다.
- author는 개인 github id로 작성합니다.
- 참고: title을 한글로 바꿔도 좋습니다.

```
---
layout: post
title: 포스팅 하는 법
date: 2016-11-22 15:09:10 +0900
author: dalzony
---
```

### 최종 모습 예

```
---
layout: post
title: 포스팅 하는 법
date: 2016-11-22 15:09:10 +0900
author: dalzony
---
```

### 기타

- 깃헙 마크다운을 따릅니다, 변경사항이 필요하면 요청해주세요.
- 계층의 시작은 `##`입니다.

## 로컬에서 실행해보기

```
$ jekyll s
```

- 미리 볼 수 있습니다.

## publish 하는 법

### 커밋 & 푸시

- 잠시 후 반영 됩니다.


### organization 소속이 아닐 경우

- pull-req를 날려주세요.
- author에 관한 기타 문의는 clojure.kr@gmail.com로 해주세요.
  - 또는 clojure-korea.slack.com의 `#clojurekr` 채널

[clojure-korea github]: https://github.com/clojure-korea/clojure.kr
