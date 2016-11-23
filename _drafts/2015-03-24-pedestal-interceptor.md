---
layout: post
title: 
date: 2015-03-24 15:09:10 +0900
author: Eunmin Kim
---

Pedestal은 요청과 응답을 순서대로 처리할 수 있는 interceptor라는 기능을 제공합니다. Interceptor는 전체 요청에 적용하거나 특정한 라우팅에 적용할 수 있습니다. Inteceptor는 요청이 들어올 때 처리되는 `:enter` 스테이지, 응답이 나갈 때 처리되는 `:leave` 스테이지, 에러가 발생한 경우 처리되는 `:error` 스테이지가 있습니다. 처리하고 싶은 스테이지만 구현해주면 되고 처리하지 않을 스테이지는 없어도 됩니다. 아래는 간단한 interceptor의 예입니다.

```clojure
(def sample-interceptor
  (io.pedestal.interceptor/interceptor
    {:enter
     (fn [ctx]
       (println "enter")
       ctx)
     :leave
     (fn [ctx]
       (println "leave")
       ctx)
     :error
     (fn [ctx e]
       (println "error")
       ctx)}))
```

이 interceptor를 적용하고 핸들러에게 요청을 보내보면 `enter`, `leave`가 순서대로 출력됩니다.

만약 핸들러에서 Exception이 발생한다면 `enter`, `error`가 순서대로 출력됩니다.

각 스테이지의 함수는 첫번째 인자로 context를 받습니다. context에는 여러가지 정보가 있는 맵으로 구성되어 있고 함수는 context를 리턴해줘야합니다.

context에서 중요한 키는 `:request`와 `:response` 입니다.

`:request`과 `:response`는 Ring Spec 형식의 맵입니다.

요청이 오면 `:request`에는 요청 맵 값이 있고 `:response`는 `nil` 값이 들어 있습니다.

스테이지 함수에서 context를 리턴해주기 전에 `:response`와 `:request`를 변경해 다음에 실행될 핸들러나 interceptor에게 전달 할 수 있습니다.

예를 들어 context에 있는 `:request` 값이 잘못되었다면 `:response`에 에러 응답을 내려 주는 interceptor를 만들 수 있습니다.

```clojure
:enter
(fn [ctx]
  (if (valid-request? (:request ctx))
    ctx
    (assoc ctx :response {:status 400 :body "Bad Request" :headers {}})))
```

또 `:request` `:uri`의 확장자를 보고 `:response`에 알맞은 `Content-Type`을 넣어주는 inteceptor도 만들 수 있습니다.

```clojure
:leave
(fn [ctx]
  (assoc-in ctx [:response :headers "Content-type"]
    (mime/ext-mime-type (get-in ctx [:request :uri]))))
```

### Interceptor 처리 순서

Interceptor는 여러 개를 적용할 수 있고 적용한 순서대로 실행됩니다.

예를 들어 `[interceptor1 interceptor2 interceptor3]` 순서를 가지는 interceptor가 적용되어 있다면 요청이 왔을 때,

`interceptor1 :enter` -> `interceptor2 :enter` -> `interceptor3 :enter` -> 핸들러 -> `interceptor3 :leave` -> `interceptor2 :leave` -> `interceptor1 :leave`

순서대로 실행되게 됩니다.

사실 핸들러(`defhandler`)도 interceptor로 구현 되어 있습니다. 위에서 처럼 모든 interceptor가 적용된 순서대로 `:enter` 스테이지가 실행되고 반대 순서대로 `:leave` 스테이지가 실행됩니다.

여기서 주의해야할 점은 어떤 인터셉터에서든 context에 `:response` 값이 설정되면 다음 인터셉터로 전달 되지 않고 역순으로 `:leave` 스테이지가 실행됩니다.

예를 들어  `interceptor2`의 `:enter` 스테이지에서 `:response` 값을 설정하면 `interceptor3`의 `:enter` 스테이지가 불리지 않고 `interceptor2`의 `:leave` 스테이지가 불리고 그 다음은 `interceptor1`의 `:leave` 스테이지가 불립니다.

만약 interceptor(핸들러도 interceptor)에서 Exception이 발생하면 `:error` 스테이지가 실행 됩니다.

`:error` 스테이지는 `:leave` 스테이지와 비슷한 실행 순서를 가집니다.

만약 `interceptor2`의 `:enter` 스테이지에서 Exception이 발생하면 역시 `interceptor3`의 `:enter`가 불리지 않고 `interceptor2`의 `:error`가 불립니다. 그리고 `interceptor1`의 `:leave`가 불립니다.

여기서 중요한 점은 `interceptor1`의 `:error`가 불리지 않는다는 점입니다. `:error` 스테이지는 예외를 올바르게 처리했을 것으로 가정하기 때문에 다음부터는 `:leave` 스테이지가 불립니다.

만약 `interceptor1`에도 `:error` 스테이지가 불리게 하고 싶다면 `interceptor2`의 `:error`에서 Exception을 전달(`throw`)하면 됩니다.

또 `interceptor2`에 `:error` 스테이지가 없다면 `interceptor1`이 `:error`를 처리하게 됩니다.
