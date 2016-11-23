---
layout: post
title:
date: 2015-02-09 15:09:10 +0900
author: Eunmin Kim
---

Pedestal interceptor는 Ring middleware와 비슷한 일을 하는 기능이다.

Pedestal interceptor가 좋은 점은 interceptor가 한 쓰레드에서 동작하지 않아도 된다는 점이다. 다른 쓰레드에서 interceptor를 pause하거나 resume 할 수 도 있다.

또 Pedestal interceptor는 Ring middleware처럼 middleware에서 다음 middleware을 호출하지 않아도 되기 때문에 비동기로 동작하게 만들 수 도 있다.

자세한 내용은 문서(https://github.com/pedestal/pedestal/blob/master/guides/documentation/service-interceptors.md)를 참조하고 여기서는 간단한 예제로 맛을 보자.

Interceptor는 `defbefore`, `defafter`, `defaround` 3개의 매크로를 제공하고 각각 request 핸들러가 실행 전, 실행 후, 실행 전 후에 추가 기능을 넣을 수 있다. 또 이 3개의 매크로를 사용하기 편하게 만든 `defon-request`, `defon-response`, `defmiddleware`가 있다. 실행 시점은 앞에 말한 3개의 매크로와 같지만 넘어오는 파라미터가 다르다. 예제를 보자.

```Clojure
(require '[io.pedestal.interceptor :as interceptor]
         '[ring.util.response :as ring-resp])

(interceptor/defon-request request-interceptor
  [request]
  (assoc request :user {:id "123" :name "todd"}))

(interceptor/defon-response response-interceptor
  [response]
  (ring-resp/content-type response "application/json"))

(defroutes routes
  [[["/interceptor" {:get #(ring-resp/response "I'm Pedestal")} ^:interceptors [requeset-interceptor response-interceptor]]]])
```

위 예제는 간단한 `on-request`와 `on-response` interceptor 예제다. `on-request` 예제는 요청을 처리하기 전에 request 맵에 `:user`키를 추가하는 예이다. `on-response` 예제는 응답을 주기 전에 response 해더에 `content-type`을 `application/json`으로 설정하는 예제이다.
