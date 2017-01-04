---
layout: post
title: "[아장아장 클로져] 함수를 다른 함수의 인자로 넘겨보자" 
date:   2017-01-04 16:30:00 +0900 
author: dayoungle
---

## [아장아장 클로져] 함수를 다른 함수의 인자로 넘겨보자

아래와 같이 try-catch문이 중복되는 함수 여러 개를 만들었다. 

```clojure
(defn create [param]
  (try
    (wrap-resp (middle-ware1 conn1 :INSERT param))
    (catch Exception e
      (.getMessage e))))

(defn update [param]
  (try
    (wrap-resp (middle-ware2 conn2 :UPDATE param))
    (catch Exception e
      (.getMessage e))))
```

원하는대로 exception이 발생했을 때 오류 메세지가 스트링으로 전달되는 것을 확인했다. 따라서 중복되는 부분인  try-catch 부분을 execute라는 이름의 함수로 추출하고, 각각 다른 종류의 middleware를 사용하는 함수만 execute 함수의 인자로 넘기기로 계획했다. 

```clojure
(defn create [param] 
  (execute (middle-ware1 conn1 :INSERT param)))

(defn execute [f]
  (try
    (f)
    (catch Exception e
      (.getMessage e))))
```
**이렇게\_실행하면\_처음에\_만든\_create\_함수와\_마찬가지로\_string으로\_된\_오류\_메세지가\_올\_줄\_알았어.clj**




그러나 정상 실행이 되지 않았다. 똑같은 함수를 만든 것 같은데 무엇이 문제일까? 쉽게 풀이하자면, 클로져 함수가 안에 있는 소괄호부터 실행되기 때문이다. 바깥에 있는 함수를 실행하는 과정에서 안에 있는 함수를 실행하는 것이 아니다. 

```clojure
(function1 (function2))
```
위의 경우, function1을 수행하기 위해서 스택메모리에 넣고 그 안에 있는 function2 를 인자로 확인하고 function1을 수행하기 위해서  function2를 수행하는 것이 아니다. 
function2를 수행한 후 결과 값이 괄호 밖으로 나오면 그 값을 인자로 사용해서 function1이 수행되는 것. 따라서 위와 같은 경우 수행 순서는 그저 function2-> function1이 되는 것이다.

### 잘 된 예 

```clojure
(defn execute [f & args]
  (try
    (apply f args)
    (catch Exception e
      (.getMessage e))))

(defn create [param]
  (execute middle-ware1 conn1 :INSERT_UMAP param))
```

위의 경우 execute 함수의 인자로 f 와 & args 즉 함수 하나와 가변인자를 받는다. 인자로 받은 middle-ware1 함수와 를 실행하는데 필요한 인자들을 execute 함수로 넘기는 것이다. 수정된 create 함수를 보면 앞서 middle-ware1 함수를 괄호로 싸서 실행시켰던 것에 반해 괄호가 없어지고 middle-ware1이라는 함수와 그 인자들이 execute의 인자로 넘어가는 것을 알 수 있다.

### apply 함수 알아보기 
execute 안에서는 apply가 등장한다. [clojuredocs의 apply](https://clojuredocs.org/clojure.core/apply)에 대한 설명을 보면 함수를 인자로 넘겨야 할 때 apply를 사용하는 방법에 대해서 쉽게 설명되어 있다. 간단히 apply에 대한 설명만 옮기자면 다음과 같다. 

```
Applies fn f to the argument list formed by prepending intervening arguments to args.
사이에 있는(intervening) 인자들을 추가해서(prepending) 만든 리스트 인자를 함수 f에 적용한다
```

apply 함수는 리스트 형태로 전달받은 인자를 인자로 전달받은 함수의 인자로 쓸 수 있도록 한다.
아래의 두 줄은 같은 결과를 가지고 온다. 

```clojure
(apply function1 [params]) 
(function1 params)
```

아직 클로져를 많이 사용해보지 않았지만 apply는 reduce를 비롯해서 많이 쓰이는 함수로 보인다. 그렇게 보이는 이유는 [4clojure](http://www.4clojure.com/)에서 많이 연습시키기 때문이다. clojuredocs의 apply 예제를 보면 이해가 더 쉬우니 참조하면 좋다. 
















