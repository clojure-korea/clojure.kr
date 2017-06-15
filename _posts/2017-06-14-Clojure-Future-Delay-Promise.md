---
layout: post
title: "Clojure Future / Delay / Promise"
date:   2017-06-14 22:30:00 +0900
author: storyzero
---

동시성 코드를 읽기 쉽게, 안전하게 작성하기란 어려운 일입니다.
스레드를 직접 다루는 것을 대신해 이를 추상화한 Futures, Delays, 그리고 Promises 을 사용하는 법을 살펴보겠습니다.

### Future

Future 는 작업 정의하고 다른 스레드에 작업 실행을 위임시킵니다. 결과를 바로 응답받지 않아도 된다면 Futures 를 사용할 수 있습니다. future 매크로를 사용해 Future 를 생성할 수 있습니다.

```clojure
user=> (def result (future (apply + (range 100000000))))
#'user/result
#object[clojure.core$future_call$reify__6962 0x551a9ed0 {:status :pending, :val nil}]
```

result 는 1부터 100000000까지의 합을 구하는 작업이 정의된 future 입니다. future 는 생성과 동시에 다른 스레드에서 작업 실행합니다.

```clojure
user=> (realized? result)
false
user=> (realized? result)
true
```

realized? 는 작업을 마쳤는지 확인하는 함수입니다.

```clojure
user=> (deref result)
4999999950000000
user=> @result
4999999950000000
```

작업을 마치면 deref 함수로 result 에서 결과 요구를 합니다. 또한 @result 로 확인할 수 있습니다.

```clojure
user=> (def f (future (Thread/sleep 10) (println "print once") true))
#'user/f
print once
user=> @f
true
user=> @f
true
```

"print once" 는 Future 를 결과 요구를 두번  했으나 한번만 출력되었습니다. Future 는 작업을 한번만 수행하고 결과를 캐시하고 있음을 알 수 있습니다.

```clojure
user=> (def f (future (Thread/sleep 2000) true))
#'user/f
user=> (deref f 1000 false)
false
user=> (deref f 1000 false)
true
```

마지막으로 Future 는 작업이 아직 끝나기 전에 결과 요구를 하면 블록킹됩니다. deref 함수는 timeout 과 timeout 시 리턴할 값을 인자로 전달할 수 있습니다. 그래서 첫 번째 결과 요구는 1초 후 false 를 반환하고, 두 번째 요구는 작업이 완료된 결과 true 를 반환하였습니다.

### Delay

Delay 는 Future 와 같이 작업 정의할 수 있습니다. Future 와 다른 점은 즉시 작업 실행되지 않습니다. delay 매크로를 사용해 Delay 를 생성할 수 있습니다.

```clojure
user=> (def result (delay (println "print once") (apply + (range 100000000))))
#'user/result
user=> result
#object[clojure.lang.Delay 0x15caa6a8 {:status :pending, :val nil}]
```

Future 와 다르게 "print once" 가 아직 출력되지 않았습니다. Delay 는 force 함수를 사용하거나 값을 참조하려고 할 때 작업 실행합니다.

```clojure
user=> (force result)
print once
4999999950000000
user=> @result
4999999950000000
```

Delay 는 Future 와 비슷하지만 작업 실행 시점을 임의로 제어할 수 있습니다.

### Promise

Promise 는 결과 요구만 정의할 수 있습니다. Future 와 Delay 는 작업 정의와 결과 요구가 수반되었으나, Promise 는 오직 결과 요구만 정의합니다. promise 함수를 사용해 Promise 를 생성할 수 있습니다.

```clojure
user=> (def result (promise))
#'user/result
user=> result
#object[clojure.core$promise$reify__7005 0x5fdbe1e {:status :pending, :val nil}]
```

Promise 에는 작업 정의도 하지 않았고 작업 실행도 하지 않았습니다.

```clojure
user=> (deref result 1000 0)
0
user=> (deliver result (apply + (range 100000000)))
#object[clojure.core$promise$reify__7005 0x5fdbe1e {:status :ready, :val 4999999950000000}]
user=> (deref result 1000 0)
4999999950000000
```

deliver 함수를 사용하면 result 에게 결과를 전달할 수 있습니다. 결과가 전달되기 전까지는 result 는 결과 요구하더라도 블록킹만 됩니다.

result 를 첫번째 결과 요구할 때는 timeout 후 0 를 반환했습니다. deliver 함수를 사용해 결과를 전달하니 두번째 요구에는 원하는 결과가 반환되었습니다.

### 활용사례

4명의 사용자가 있다고 가정합시다.

```clojure
user=> (def users
  #_=>   {:lilly {:level 1, :score 20}
  #_=>    :mathew {:level 4, :score 60}
  #_=>    :james {:level 7, :score 90}
  #_=>    :phil {:level 10, :score 120}})
#'user/users
```

그리고 실행시간이 1초 걸리는 사용자 정보를 가져오는 API 가 있다고 합시다.

```clojure
user=> (defn api-call [name]
  #_=>   (Thread/sleep 1000)
  #_=>   (get users name))
#'user/api-call
user=> (time (api-call :lilly))
"Elapsed time: 1003.242726 msecs"
{:level 1, :score 20}
```

사용자 이름 목록을 받아 사용자 정보를 가져오는 함수를 만든다면 아래와 같이 만들 수 있습니다. 하지만 사용자 정보를 가져오는데 사람 수만큼 오래 걸리는 문제가 있습니다.

```clojure
user=> (defn get-users [names]
  #_=>   (let [users (doall (map #(api-call %) names))]
  #_=>     users))
#'user/get-users

user=> (time (get-users `(:lilly :mathew :james :phil)))
"Elapsed time: 4012.425435 msecs"
({:level 1, :score 20} {:level 4, :score 60} {:level 7, :score 90} {:level 10, :score 120})
```

Future 를 사용하면 보다 빠르게 사용자 정보를 가져올 수 있습니다. 사용자별 요청을 병렬로 수행할 수 있기 때문입니다.

```clojure
user=> (defn get-users-concurrency [names]
  #_=>   (let [futures (doall (map #(future (api-call %)) names))]
  #_=>     (map deref futures)))
#'user/get-users-concurrency

user=> (time (get-users-concurrency `(:lilly :mathew :james :phil)))
"Elapsed time: 0.499045 msecs"
({:level 1, :score 20} {:level 4, :score 60} {:level 7, :score 90} {:level 10, :score 120})
```

### Wrap-up

Future, Delay, 그리고 Promise 는 모두 Thread 를 사용하는 법을 추상화한 것 입니다. 덕분에 Thread 나 Runnable 같은 것을 직접 다루지 않아도 동시성 프로그래밍을 할 수 있습니다.

이들은 작업 정의, 작업 실행, 그리고 결과 요구(참조)를 유연하게 제어할 수 있게 도와줍니다. Future 는 작업 정의와 실행이 함께 수반됩니다. 반면 Delay 는 작접 정의와 실행이 분리되어 있습니다. Promise 는 작업 정의와 실행이 없고 결과 요구(참조)만 할 수 있습니다. 이를 표로 정리하면 아래와 같습니다.

| -         |Future|Delay|Promise|
| -------- |:------|:------|:-----------|
| 작업 정의  | O | O | X |
| 작업 실행  | O | X | X |
| 결과 요구  | O | O | O |
