---
layout: post
title: "[아라크네 튜토리얼] 2. HTTP 요청 처리하기"
date:   2017-01-21 16:00:00 +0900
author: eunmin
---

## HTTP 요청 처리하기

이 글은 아라크네 공식 문서에 있는 [Handling HTTP Requests] (http://docs.arachne-framework.org/tutorials/http-requests/)를 정리한 버전입니다.

이번 장은 [프로젝트 만들기](http://clojure.kr/arachne-tutorials-creating-a-project)에서
만든 예제 프로젝트에 Pedestal 모듈을 추가해서 진행합니다.
이번에 설명할 내용은 HTTP 요청을 처리하는 함수를 만들고 서버를 구성해서 실행하는 방법에 대해 알아보겠습니다.

### Pedestal 모듈 사용하기

아라크네는 [Pedestal](http://pedestal.io/) HTTP 서버를 사용합니다.

`project.clj`에  `arachne-core` 대신 `arachne-pedestal`로 바꿉니다.

```clojure
:dependencies [[org.clojure/clojure "1.9.0-alpha14"]
               [org.arachne-framework/arachne-pedestal "0.1.0-master-0036-59ecd65"]
               [datascript "0.15.5"]
               [ch.qos.logback/logback-classic "1.1.3"]]
```

그리고 `arachne.edn` 파일도 `arachne-core` 모듈 대신 `arachne-pedestal`을 사용하도록 바꿉니다.

```clojure
[{:arachne/name :myproj/app
  :arachne/inits ["config/myproj.clj"]
  :arachne/dependencies [:org.arachne-framework/arachne-pedestal]}]
```

### 핸들러 함수 만들기

아라크네도 Ring 스펙에 맞는 핸들러 함수를 사용합니다. `src/myproj/core.clj`에 아래와 같은 핸들러를 정의합니다.

```clojure
(defn hello-handler
  [req]
  {:status 200
   :body "Hello, world!"})
```

"Hello, world!"를 출력하는 핸들러 입니다.

### 서버 설정하기

먼저 작성한 `config.clj`에 아래 처럼 핸들러를 정의합니다.

```clojure
(require '[arachne.http.dsl :as h])

(h/handler :myproj/hello 'myproj.core/hello-handler)
```

다음은 아래와 같이 서버와 핸들러에 매핑할 라우팅을 정의합니다.

```clojure
(require '[arachne.pedestal.dsl :as p])

(p/server :myproj/server 8080

  (h/endpoint :get "/" :myproj/hello)

  )
```

완성된 `config.clj`는 아래와 같습니다.

```clojure
(require '[arachne.core.dsl :as a])
(require '[arachne.http.dsl :as h])
(require '[arachne.pedestal.dsl :as p])

(a/component :myproj/widget-1 'myproj.core/make-widget)

(a/runtime :myproj/runtime [:myproj/server])

(h/handler :myproj/hello 'myproj.core/hello-handler)

(p/server :myproj/server 8080

  (h/endpoint :get "/" :myproj/hello)

  )
```

### 서버 실행하기

서버를 실행하기 전에 로그 파일 설정을 위해 `resources` 아래 `logback.xml` 파일을 만듭니다.

```xml
<configuration debug="false">

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %-5level - %logger{36} %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
      <appender-ref ref="STDOUT"/>
    </root>

</configuration>
```

다음은 커맨드 라인에 아래 처럼 입력해서 실행합니다.

```bash
lein run :myproj/app :myproj/runtime
```

서버가 실행되면 `http://localhost:8080/`에서 Hello, world!가 표시되는 것을 확인 할 수 있습니다.

### Path 파라미터

요청 맵에 `:path-params` 키로 Path 파라미터를 참조할 수 있습니다. 아래는 `:name` Path 파라미터를
사용하는 예제입니다.

```clojure
(defn greeter
  [req]
  (let [name (get-in req [:path-params :name])]
    {:status 200
     :body (if (empty? name)
             "Who's there!?"
             (str "Hello, " name "!"))}))
```

그리고 설정 스크립트에 라우터와 핸들러를 매핑합니다.

```clojure
(h/endpoint :get "/greet/:name" (h/handler 'myproj.core/greeter))
```

위에 정의한 엔드포인트는 아래 처럼 정의 해도 됩니다.

```clojure
(def handler-eid (h/handler 'myproj.core/greeter))
(h/endpoint :get "/greet/:name" handler-eid)
```

```clojure
(h/handler :myproj/greeter 'myproj.core/greeter)
(h/endpoint :get "/greet/:name" :myproj/greeter)
```

```
(h/endpoint :get "/greet/:name" (h/handler :myproj/greeter 'myproj.core/greeter))
```

### 컴포넌트와 런타임

예제에서 먼저 만든 `:myproj/widget-1` 컴포넌트는 실행되지 않았는데 여기에서 사용하지 않기 때문에
런타임에 추가하지 않았습니다. 만약 `:myproj/widget-1` 컴포넌트가 실행되게 하려면 아래 처럼 런타임을
정의하면 됩니다.

```clojure
(a/runtime :myproj/runtime [:myproj/server :myproj/widget-1])
```
