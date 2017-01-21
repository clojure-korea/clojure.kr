---
layout: post
title: "[아라크네 튜토리얼] 1. 프로젝트 만들기"
date:   2017-01-21 15:00:00 +0900
author: eunmin
---

## 프로젝트 만들기

이 글은 아라크네 공식 문서에 있는 [Createing a Project](http://docs.arachne-framework.org/tutorials/creating-a-project/)를 정리한 버전입니다.

### 프로젝트 만들기

아래와 같이 leiningen 프로젝트를 생성합니다.

```bash
lein new myproj
```

`project.clj`에 다음과 같이 의존성이 있는 파일을 추가합니다.

```clojure
(defproject myproj "0.1.0-SNAPSHOT"
  :dependencies [[org.clojure/clojure "1.9.0-alpha14"]
                 [org.arachne-framework/arachne-core "0.1.0-master-0081-0ab2073"]
                 [datascript "0.15.5"]
                 [ch.qos.logback/logback-classic "1.1.3"]]
  :repositories [["arachne-dev"
                  "http://maven.arachne-framework.org/artifactory/arachne-dev"]])
```

### 컴포넌트 만들기

아라크네는 Stuart Sierra가 만든 Component 라이브러리를 사용합니다.

`src/myproj/core.clj`를 열고 컴포넌트를 생성합니다.

```clojure
(ns myproj.core
  (:require [com.stuartsierra.component :as c]
            [arachne.log :as log]))

(defrecord Widget []
  c/Lifecycle
  (start [this]
    (log/info :msg "Hello, world!")
    this)
  (stop [this]
    (log/info :msg "Goodnight!")
    this))
```

다음으로 컴포넌트를 생성하는 생성 함수를 만듭니다.

```clojure
(defn make-widget
  "Constructor for a Widget"
  []
  (->Widget))
```

### 설정 스크립트 만들기

아라크네는 어플리케이션 설정을 위한 DSL을 제공하고 있습니다.

`config/myproj.clj` 파일을 아래와 같이 생성합니다.

```clojure
(require '[arachne.core.dsl :as a])

(a/component :myproj/widget-1 'myproj.core/make-widget)

(a/runtime :myproj/runtime [:myproj/widget-1])
```

`arachne.core.dsl/component` 함수는 컴포넌트를 정의합니다. 첫번째 인자는 컴포넌트의 아이디로 `:myproj/widget-1`로 어디서나 참조할 수 있습니다. 두번째 인자는 컴포넌트 생성 함수를 넘깁니다.
세번째 인자로 컴포넌트의 의존성을 정의 할 수 있는데 이번 예제에서는 사용하지 않습니다.

`arachne.core.dsl/runtime` 함수는 아라크네 실행 환경으로 같은 설정으로 여러개의 실행 환경으로
실행할 수 있습니다. 여기에는 인자로 런타임 이름과 사용할 컴포넌트를 지정합니다.

### 어플리케이션 메타데이터

아라크네 메타데이터는 어플리케이션 이름, 설정 파일의 위치, 의존하는 아라크네 모듈등을 정의합니다.

`/resources` 아래 `arachne.edn` 파일로 생성합니다.

```clojure
[{:arachne/name :myproj/app
  :arachne/inits ["config/myproj.clj"]
  :arachne/dependencies [:org.arachne-framework/arachne-core]}]
```

- `:arachne/name`은 어플리케이션 이름입니다.
- `:arachne/inits`는 초기화 할 때 사용하는 백터입니다. 설정 파일의 위치를 지정합니다.
- `:arachne/dependencies`는 아라크네 모듈을 정의합니다.

이제 아라크네를 실행할 준비가 되었습니다. 위에 만든 파일은 아래와 같은 구조를 가집니다.

```
myproj
├── config
│   └── myproj.clj
├── project.clj
├── resources
│   └── arachne.edn
└── src
    └── myproj
        └── core.clj
```

### 어플리케이션 실행하기

아라크네 어플리케이션은 두가지 방법으로 실행할 수 있습니다.

- REPL에서 실행하기 (개발 환경에서)
- 커맨드라인에서 실행하기 (프로덕션 환경에서)

#### REPL에서 실행하기

REPL을 열고 아래 두 네임스페이스를 `require` 해줍니다.

```clojure
(require '[arachne.core :as arachne])
(require '[com.stuartsierra.component :as c])
```

다음은 설정 값을 준비합니다.

```clojure
(def cfg (arachne/config :myproj/app))
;; => #'user/cfg
```

다음은 생성한 설정 값으로 런타임 값을 만듭니다.

```clojure
(def rt (arachne/runtime cfg :myproj/runtime))
```

이제 아래와 같이 런타임을 실행합니다.

```clojure
(def started-rt (c/start rt))
=> #'user/started-rt
```

실행하면 로그가 아래와 같이 두 줄 출력 됩니다.

```
14:44:07.081 [nREPL-worker-3] INFO  arachne.core.runtime - {:msg "Starting Arachne runtime", :line nil}
14:44:07.090 [nREPL-worker-3] INFO  myproj.core - {:msg "Hello, world!", :line nil}
```

`stop` 함수로 어플리케이션을 종료 할 수 있습니다.

```clojure
(c/stop started-rt)
14:46:38.596 [nREPL-worker-4] INFO  arachne.core.runtime - {:msg "Stopping Arachne runtime", :line nil}
14:46:38.597 [nREPL-worker-4] INFO  myproj.core - {:msg "Goodnight!", :line nil}
```

#### 커맨드라인에서 실행하기

아라크네는 `arachne.run` 네임스페이스에 `-main` 함수를 정의하고 있기 때문에 `project.clj`에
`arachne.run` 네임스페이스를 `:main` 키에 지정해주고 실행하면 됩니다.

```
:main arachne.core
```

실행 할때는 `lein run`에 인자 두개를 넣어 실행합니다. 첫번째는 어플리케이션 이름이고 두번째는 실행할 런타임 이름입니다.

```bash
lein run :myproj/app :myproj/runtime
14:51:23.810 [main] INFO  arachne.run - {:msg "Launching Arachne application", :name ":myproj/app", :runtime ":myproj/runtime", :line nil}
14:51:25.295 [main] INFO  arachne.run - {:cfg "cfg", :line nil}
14:51:25.319 [main] INFO  arachne.core.runtime - {:msg "Starting Arachne runtime", :line nil}
14:51:25.325 [main] INFO  myproj.core - {:msg "Hello, world!", :line nil}
```

어플리케이션은 자동으로 종료되지 않기 때문에 종료하려면 수동으로 종료 해줘야합니다.

### 에러 리포팅

아라크네는 에러 리포팅 툴이 있습니다. 확인해보기 위해 아래 처럼 에러를 만들어 봅시다.

```
(a/component :widget-1 'myproj.core/make-widget)
```

다음에 REPL에서 설정을 만들어봅시다.

```
(require '[arachne.core :as arachne])
(require '[com.stuartsierra.component :as c])

(def cfg (arachne/config :myproj/app))

CompilerException arachne.ArachneException: Error initializing module `:myproj/app` (type = :arachne.core.module/error-in-initializer), compiling:(form-init2498552204824241129.clj:1:10)
```

위와 같이 에러가 나긴 하지만 에러를 확인하기 어렵습니다. `*e`를 출력해보면 아래와 같습니다.

```clojure
arachne.ArachneException: Error initializing module `:myproj/app` (type = :arachne.core.module/error-in-initializer), compiling:(form-init2498552204824241129.clj:1:10)
	at clojure.lang.Compiler$InvokeExpr.eval(Compiler.java:3657)
	at clojure.lang.Compiler$DefExpr.eval(Compiler.java:451)
	at clojure.lang.Compiler.eval(Compiler.java:6983)
	at clojure.lang.Compiler.eval(Compiler.java:6941)
	at clojure.core$eval.invokeStatic(core.clj:3187)
	at clojure.core$eval.invoke(core.clj:3183)
	at clojure.main$repl$read_eval_print__9983$fn__9986.invoke(main.clj:242)
	at clojure.main$repl$read_eval_print__9983.invoke(main.clj:242)
	at clojure.main$repl$fn__9992.invoke(main.clj:260)
	at clojure.main$repl.invokeStatic(main.clj:260)
	at clojure.main$repl.doInvoke(main.clj:176)
	at clojure.lang.RestFn.invoke(RestFn.java:1523)
	at clojure.tools.nrepl.middleware.interruptible_eval$evaluate$fn__4650.invoke(interruptible_eval.clj:87)
	at clojure.lang.AFn.applyToHelper(AFn.java:152)
	at clojure.lang.AFn.applyTo(AFn.java:144)
	at clojure.core$apply.invokeStatic(core.clj:657)
	at clojure.core$with_bindings_STAR_.invokeStatic(core.clj:1963)
	at clojure.core$with_bindings_STAR_.doInvoke(core.clj:1963)
	at clojure.lang.RestFn.invoke(RestFn.java:425)
	at clojure.tools.nrepl.middleware.interruptible_eval$evaluate.invokeStatic(interruptible_eval.clj:85)
	at clojure.tools.nrepl.middleware.interruptible_eval$evaluate.invoke(interruptible_eval.clj:55)
	at clojure.tools.nrepl.middleware.interruptible_eval$interruptible_eval$fn__4695$fn__4698.invoke(interruptible_eval.clj:222)
	at clojure.tools.nrepl.middleware.interruptible_eval$run_next$fn__4690.invoke(interruptible_eval.clj:190)
	at clojure.lang.AFn.run(AFn.java:22)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
```

아라크네는 디버깅을 도와주기 위해 `(arachne.error/explain)`로 마지막 예외를 보기 좋게 출력해 줍니다.
