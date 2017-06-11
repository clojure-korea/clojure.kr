---
layout: post
title: "Async Ring 미들웨어 패턴"
date:   2017-06-11 16:30:00 +0900
author: eunmin
---

최근 Ring 라이브러리 1.6이 발표되면서 Ring Spec이 1.4 버전으로 업데이트되었습니다. 비동기 핸들러를
지원하는 내용이 추가되었는데요. 비동기 핸들러는 아래 예제 코드처럼 생겼습니다.

```clojure
(defn handler [request respond raise]
  (respond (ok "Hello World")))
```

Ring 핸들러는 `request` 맵을 인자로 받아서 `response` 맵을 리턴하도록 되어 있었는데요. 비동기
Ring 핸들러는 `request` 맵과 추가로 응답을 인자로 받는 함수와 에러를 인자로 받는 함수를 인자로 받게
되어 있습니다. 그래서 응답을 하는 곳에 두 번째 인자인 함수를 `response` 맵과 함께 적어 줘야 합니다.

비동기 핸들러가 위와 같이 동기 핸들러랑 다른 모양을 하고 있으므로 미들웨어를 만들 때도 다르게 만들어야
합니다. 아래 코드는 동기 핸들러에서 수행 시간을 로그에 쓰는 미들웨어 예제입니다.

```clojure
(defn wrap-time [handler]
  (fn [request]
    (let [start-time (System/currentTimeMillis)
          response (handler request)
          end-time (System/currentTimeMillis)
          ptime (- end-time start-time)]
      (log/info "time:" ptime)
      response)))
```

핸들러 함수를 수행하기 전과 후에 각각 현재 시각을 기록하고 그 차이를 로그에 써줍니다. 비동기 핸들러를
사용할 때는 핸들러의 리턴 값이 `response` 맵이 아니기 때문에 조심해서 만들어야 합니다. 아래 코드는
비동기 핸들러를 사용하는 경우 수행 시간을 로그에 쓰는 예제입니다.

```clojure
(defn wrap-time-for-async-handler [handler]
  (fn [request respond raise]
    (let [start-time (System/currentTimeMillis)]
      (handler request
        (fn [response]
          (respond response)
          (let [end-time (System/currentTimeMillis)
                ptime (- end-time start-time)]
            (log/info "time:" ptime)))
        raise))))
```

`handler` 함수가 적용되기 전에 시작 시각을 기록하고 응답을 하는 `respond` 함수가 적용된 이후에
종료 시각을 기록해서 수행 시간을 로그에 써줍니다. 보통 Ring 미들웨어는 동기 비동기 모두 지원하도록
만들기 때문에 여러 개의 인자를 같은 함수로 정의합니다.

```clojure
(defn wrap-time [handler]
  (fn
    ([request]
      (let [start-time (System/currentTimeMillis)
            response (handler request)
            end-time (System/currentTimeMillis)
            ptime (- end-time start-time)]
        (log/info "time:" ptime)
        response)))
    ([request respond raise]
      (let [start-time (System/currentTimeMillis)]
        (handler request
          (fn [response]
            (respond response)
            (let [end-time (System/currentTimeMillis)
                  ptime (- end-time start-time)]
              (log/info "time:" ptime)))
          raise))))
```
