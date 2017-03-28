---
layout: post
title: "compojure-api에서 path-params을 alias 심볼로 binding하기"
date:   2017-03-28 13:00:00 +0900
author: eunmin
---

[compojure-api](https://github.com/metosin/compojure-api)에서 `path-params`를 심볼에
바인딩하는 방법은 아래와 같습니다.

```clojure
(GET "/plus/:x/:y" []
  :path-params [x :- Long, y :- Long]
  (ok {:result (+ x y)}))
```

위 예제는 `/plus/:x/:y` 형식으로 들어온 요청에 `:x`와 `:y`자리에 각각 `x`와 `y` 심볼로 바인딩합니다.
그런데 `x` 또는 `y`가 아닌 다른 심볼에 바인딩하려면 어떻게 할까요?

`compojure-api`는 `*-params`의 restructuring에 [plumbing](https://github.com/plumatic/plumbing)이란
라이브러리를 사용합니다. `plumbing` 라이브러리는 `:as`와 같은 alias binding 기능이 있는데 이 문법을
사용해서 바인딩하면 파라미터 이름과 다른 이름의 심볼로 바인딩 할 수 있습니다.

```clojure
(GET "/plus/:x/:y" []
  :path-params [[x :as i] :- Long, [y :as j] :- Long]
  (ok {:result (+ i j)}))
```
