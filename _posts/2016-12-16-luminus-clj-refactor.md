---
layout: post
title: Luminus에서 clj-refactor 설정하기
date:   2016-12-16 16:50:00 +0900
author: eunmin
---

[Luminus](http://www.luminusweb.net/)는 `lein run`을 실행하면 `7000` 포트로 `nrepl`이 실행됩니다.
사용하는 에디터에서 `nrepl`에 연결해 개발하면 편리한데요. Emacs를 사용하는 경우 `cider`와
[clj-refactor](https://github.com/clojure-emacs/clj-refactor.el)를 많이 사용합니다.
`cider`의 경우는 최근에 Luminus 템플릿에 `+cider` 옵션이 추가되어서 `nrepl`이 실행될 때 `cider`
`nrepl-middleware`를 추가해주는 코드가 `core.clj`에 포함되는데요. 아쉽게도 `clj-refactor`에서 사용하는
`nrepl-refactor`는 자동으로 추가되는 기능은 없습니다. 그래서 `nrepl-refactor`를 따로 추가해줘야
`clj-refactor`를 제대로 쓸 수 있습니다. Luminus에 `clj-refactor`를 추가하려면 다음과 같이 해주면
됩니다.

## project.clj에 plugin 추가하기

`project.clj` 파일을 열어 `:plugins`에 `nrepl-refactor` 플러그인을 추가합니다.

```clojure
[refactor-nrepl "버전"]
```

## nrepl-refactor 미들웨어 추가하기

`core.clj` 파일을 열어 `Mount`로 정의된 `repl-server` 상태 `:start` 부분을 다음과 같이 고칩니다.

```clojure
(when-let [nrepl-port (env :nrepl-port)]
  (repl/start {:port nrepl-port
               :handler (apply nrepl-server/default-handler
                 (cons #'wrap-refactor (map resolve cider-middleware)))))
```

위 코드를 쓰려면 아래 처럼 필요한 것들을 `require`해 추가해 줍니다.

```clojure
[cider.nrepl :refer [cider-middleware]]
[clojure.tools.nrepl.server :as nrepl-server]
[refactor-nrepl.middleware :refer [wrap-refactor]]
```

## cider 연결하기

`lein run`으로 프로젝트를 실행해 놓고 Emacs에서 `cider-connect`로 `Host:localhost`, `Port:7000`으로
접속하면 nrepl에 접속되고 `clj-refactor`를 사용할 수 있습니다.
