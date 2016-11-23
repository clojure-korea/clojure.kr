---
layout: post
title:
date: 2015-02-09 15:09:10 +0900
author: Eunmin Kim
---

Pedestal을 개발 환경으로 실행할 때 `lein run-dev`와 같이 실행한다.

`run-dev`는 `프로젝트이름.server`에 있는 `run-dev` 함수를 실행하는데 여기에 `io.pedestal.service-tools.dev`에 있는 `watch` 함수를 실행해주면 된다.

```Clojure
(defn watch-for-dev
  []
  (let [service-tools-ns 'io.pedestal.service-tools.dev]
    (do (require service-tools-ns)
        ((ns-resolve service-tools-ns (symbol "watch"))))))

(defn run-dev
  "The entry-point for 'lein run-dev'"
  [& args]
  (watch-for-dev)
  ...
  생략
```

주의해야할 점은 `io.pedestal.service-tools` 디펜던시는 `dev` 환경에만 추가되있기 때문에(project.clj) `io.pedestal.service-tools.dev`를 static하게 require하면 production 환경에서 `io.pedestal.service-tools.dev`를 찾지 못하는 문제가 발생한다.
