---
layout: post
title: "[아라크네 튜토리얼] 3. 컴포넌트와 의존성 주입"
date:   2017-02-28 13:00:00 +0900
author: eunmin
---

# 컴포넌트와 의존성 주입

이 글은 아라크네 공식 문서에 있는 [Createing a Project](http://docs.arachne-framework.org/tutorials/creating-a-project/)를 정리한 버전입니다.

이번 장도 [HTTP 요청 처리하기](http://clojure.kr/arachne-tutorials-handling-http-requests)에서
만든 예제 프로젝트에 덧붙여 진행합니다.

## 컴포넌트 만들기

`http://robohash.org`에서 제공하는 랜덤 이미지를 보여주는 HTTP 핸들러를 만들어 보려고 하는데요.
먼저 기능을 가지고 있는 `VisualHash` 컴포넌트를 만들어 봅시다.
`myproj.visual-hash` 네임스페이스에 다음과 같이 컴포넌트를 만듭니다.

```clojure
(ns myproj.visual-hash
  (:import [java.net URL]))

(defprotocol VisualHash
  (vhash [this s] "Given a string, return an image (as an InputStream)"))

(defrecord RoboHash []
  VisualHash
  (vhash [this s]
    (let [url (URL. (str "https://robohash.org/" s))]
      (.openStream url))))

(defn new-robohash
  "Constructor function for a RoboHash"
  []
  (->RoboHash))
```

컴포넌트 라이프사이클에서 특별히 처리할 일이 없기 때문에 `com.stuartsierra.component/Lifecycle`는
구현하지 않았습니다.

## 설정하기

`config/myproj.clj` 파일에 컴포넌트와 핸들러 앤드포인트를 정의합니다.

```clojure
(a/component :myproj/robohash 'myproj.visual-hash/new-robohash)

(h/endpoint :get "/robot/:name" (h/handler 'myproj.core/robot
                                  {:hash-component :myproj/robohash}))
```

핸들러에 `:hash-component`라는 키로 `:myproj/robohash`를 사용한다고 정의했습니다.

이제 `myproj.core/robot` 핸들러를 만들어 봅시다.

## 핸들러 만들기

`myproj.core` 네임스페이스에 다음과 같이 `robot` 핸들러를 만듭니다.

```clojure
(defn robot
  [req]
  (let [name (get-in req [:path-params :name])
        c (:hash-component req)]
    {:status 200
     :headers {"Content-Type" "image/png"}
     :body (vhash c name)}))
```

이제 서버를 실행하고 `http://localhost:8080/robot/yourname`으로 접속해 봅시다.

핸들러에서 `req`에 설정에서 추가한 `:hash-component`키로 컴포넌트를 사용할 수 있는 것을 볼 수 있습니다.

## 의존성 주입

이미지를 캐시하는 컴포넌트를 만들어 보면서 컴포넌트와 컴포넌트 간 의존성을 설정하는 방법을 알아봅시다.

먼저 이미지를 `myproj.visual-hash` 네임스페이스에 캐싱하는 컴포넌트를 다음과 같이 만듭니다.

```clojure
(ns myproj.visual-hash
  (:require [arachne.log :as log]
            [com.stuartsierra.component :as component])
  (:import [java.net URL]
           [java.io InputStream ByteArrayInputStream]
           [org.apache.commons.io IOUtils]))

(defrecord CachingVisualHash [delegate cache]
  component/Lifecycle
  (start [this]
    (assoc this :cache (atom {})))
  (stop [this]
    (dissoc this :cache))
  VisualHash
  (vhash [this key]
    (if-let [bytes (get @cache key)]
      (ByteArrayInputStream. bytes)
      (let [bytes (IOUtils/toByteArray (vhash delegate key))]
        (log/info :msg "CachingVisualHash cache miss" :key key)
        (swap! cache assoc key bytes)
        (ByteArrayInputStream. bytes)))))

(defn new-cache
  "Constructor function for a CachingVisualHash"
  []
  (map->CachingVisualHash {}))
```

위 컴포넌트는 `delegate`라는 컴포넌트의 `vhash` 함수를 사용합니다. `delegate` 컴포넌트는 먼저 만든
`RoboHash` 컴포넌트가 되고 설정파일에서 의존성을 설정합니다.


## 설정 변경하기

`config/myproj.clj`에 새 컴포넌트를 정의하고 `myproj.core/robot` 핸들러가 새 컴포넌트를 사용하도록 합니다.

```clojure
(a/component :myproj/hashcache 'myproj.visual-hash/new-cache {:delegate :myproj/robohash})

(h/endpoint :get "/robot/:name" (h/handler 'myproj.core/robot
                                   {:hash-component :myproj/hashcache}))
```

`component` 함수의 마지막 인자로 `delegate` 컴포넌트를 정의한 것을 볼 수 있습니다.

이제 서버를 실행하면 같은 url의 경우 두번째 부터는 빠르게 이미지가 표시되는 것을 볼 수 있습니다.
