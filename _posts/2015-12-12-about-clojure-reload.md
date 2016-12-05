---
layout: post
title: Clojure Reload에 대해서
date: 2015-02-09 13:09:10 +0900
author: eunmin
---

클로저 REPL을 처음 사용할 때 클로저 코드를 다시 불러오는 문제에 혼란을 겪는 경우가 많이 있어 예를 들어 설명해보려고 한다. 스크립트 형태의 언어들은 코드가 수정되면 실행할 때마다 기본적으로 새로 변경된 내용을 반영해 실행하지만 클로저는 기본적으로 실행할 때마다 새로 변경된 내용을 반영해 주지 않는다. 아래 설명되어 있는 예제를 따라서 해보면 클로저에서 다시 코드를 불러오는 방법에 대해 이해할 수 있을 것이다.

먼저 `lein new clojure-reload`라고 입력해 연습용 프로젝트를 만들어 보자.

프로젝트가 생성되면 `clojure-reload.core/foo`라는 샘플 함수가 생기는데 일단 REPL을 실행해서 샘플 함수를 실행해 보자.

```clojure
lein repl
nREPL server started on port 54998 on host 127.0.0.1 - nrepl://127.0.0.1:54998
REPL-y 0.3.7, nREPL 0.2.12
Clojure 1.7.0
Java HotSpot(TM) 64-Bit Server VM 1.8.0_65-b17
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

user=> (require '[clojure-reload.core :refer :all])
nil
user=> (foo "AH!!")
AH!! Hello, World!
nil
user=>
```

REPL의 기본 네임스페이스에서 `clojure-reload.core`에 있는 `foo` 함수를 편리하게 사용하기 위해 `require`에 `:refer :all`을 추가해줬다.

REPL을 닫지 말고 `src/clojure_reload/core.clj` 파일을 열어 `foo` 함수를 다음과 같이 고쳐보자.

```clojure
(ns clojure-reload.core)

(defn foo
  "I don't do a whole lot."
  [x]
  (println x "안녕?"))
```

그리고 아까 열어둔 REPL에서 `foo` 함수를 불러보자.

```clojure
user=> (foo "AH!!")
AH!! Hello, World!
nil
```

분명 파일을 고쳤는데도 고친 내용은 반영되지 않았다. 그럼 다시 `require`를 해보고 실행해보자.

```clojure
user=> (require '[clojure-reload.core :refer :all])
nil
user=> (foo "AH!!")
AH!! Hello, World!
nil
```

역시 변화가 없다. 수정된 내용을 반영하기 위해서는 `require`에서 제공하는 `:reload`라는 플래그를 추가해줘야 한다. `:reload` 플래그를 추가하고 다시 실행해보자.

```clojure
user=> (require '[clojure-reload.core :refer :all] :reload)
nil
user=> (foo "AH!!")
AH!! 안녕?
nil
```

고친 `foo` 함수 내용이 잘 반영되었다.

#### 사용할 네임스페이스에서 의존하고 있는 다른 네임스페이스는?

만약 `clojure-reload.core/foo` 함수에서 다른 네임스페이스의 값을 사용하고 다른 네임스페이스의 값이 변경되었을 때도 잘 동작할까? 한번 확인해보자.

이번에는 출력 형식을 정형화 해주는 `clojure-reload.util/log`라는 함수를 만들어보자. `src/clojure_reload/util.clj` 파일을 새로 만들고 다음과 같이 `log` 함수 값을 정의해보자.

```clojure
(ns clojure-reload.util)

(defn log [level message]
  (println
    (str "[" level "]")
    message))
```

다음은 `clojure-reload.core/foo` 함수를 방금 만든 `log` 함수를 사용하도록 바꿔보자.

```clojure
(ns clojure-reload.core
  (:require [clojure-reload.util :refer [log]]))

(defn foo
  [x]
  (log :debug x))
```

그리고 REPL에서 `clojure-reload.core`를 `:reload`해서 실행해보자.

```clojure
user=> (require '[clojure-reload.core :refer :all] :reload)
nil
user=> (foo "하하하하!")
[:debug] 하하하하!
nil
```

잘 동작하는 것을 볼 수 있다. 이제는 원래 확인해보려던 것을 해보자.

`level`이 키워드 형식이라 보기 싫기 때문에 키워드 값에 `name` 함수를 적용해 콜론을 없애고 출력해보자. `clojure-reload.util/log`를 다음과 같이 고치고 REPL에서 실행해보자.

```clojure
(ns clojure-reload.util)

(defn log [level message]
  (println
    (str "[" (name level) "]")
    message))
```

```clojure
user=> (require '[clojure-reload.core :refer :all] :reload)
nil
user=> (foo "하하하하!")
[:debug] 하하하하!
nil
```

`:reload`를 했음에도 고친 내용이 반영되지 않고 그대로 `:debug`라고 출력되었다.

`require`에 나열된 네임스페이스들만 `:reload`가 적용되기 때문에 `require`에 없는 `clojure-reload.util` 네임스페이스는 `:reload` 되지 않았다.

변경된 값을 반영하기 위해 `clojure-reload.util`을 `:reload`하고 다시 실행해보자. `clojure-reload.util`은 여기서 직접 사용하지 않기 때문에 `:refer :all`은 안해도 된다.

```clojure
user=> (require 'clojure-reload.util :reload)
nil
user=> (foo "하하하하!")
[debug] 하하하하!
nil
```

이제 변경된 내용이 잘 반영된 것을 볼 수 있다. 어떤 네임스페이스가 변경되었다면 해당 네임스페이스를 모두 `:reload` 해야 사용 할 수 있다는 것을 알 수 있다.
하지만 `require`는 `:reload-all`이라는 어떤 네임스페이스를 `:reload` 할때 사용하는 네임스페이스를 모두 `:reload` 해주는 `:reload-all`이라는 플래그를 제공한다.

`:reload-all`을 확인해보기 위해서 `clojure-reload.util/log` 함수를 고쳐보자. 이번에는 소문자로 표시되고 있는 level을 대문자로 출력해보자.

```clojure
(ns clojure-reload.util
  (:require [clojure.string :refer [upper-case]]))

(defn log [level message]
  (println
    (str "[" (upper-case (name level)) "]")
    message))
```

```clojure
user=> (require '[clojure-reload.core :refer :all] :reload-all)
nil
user=> (foo "하하하하!")
[DEBUG] 하하하하!
nil
```

잘 반영된 것을 볼 수 있다. `:reload-all`은 `clojure-reload.core`에서 `clojure-reload.util`을 사용하고 있다는 것을 알고 두 네임스페이스 모두 `:reload` 해준다.

#### Reload가 실패한다면?

만약 구문 오류로 `:reload`가 실패하면 어떻게 될까? `clojure-reload.util/log`함수를 일부로 에러가 나도록 고쳐보자.

```clojure
(ns clojure-reload.util
  (:require [clojure.string :refer [upper-case]]))

(defn log [level message]
  (println- ;; 없는 함수
    (str "[" (upper-case (name level)) "]")
    message))
```

```clojure
user=> (require '[clojure-reload.core :refer :all] :reload-all)

CompilerException java.lang.RuntimeException: Unable to resolve symbol: println- in this context, compiling:(clojure_reload/util.clj:5:3)
user=> (foo "하하하하!")
[DEBUG] 하하하하!
nil
```

오류가 발생하면 `require`에서 에러가 나게되고 실행해보면 이전에 불러왔던 기능이 동작하게 된다.

### Reload 관련 라이브러리

#### org.clojure/tools.namespace

https://github.com/clojure/tools.namespace

어떤 파일에 네임스페이스를 가져오거나 하는 등의 네임스페이스 관련 함수들을 제공하는 tools.namespace 라이브러리는 `refresh`, `refresh-all` 함수를 제공한다. 파일의 lastModified를 캐싱해서 변경된 파일을 찾아 `:reload` 해주는 기능을 한다. 자세한 사용법은 위 링크에 가면 예제가 있다.

#### weavejester/ns-tracker

https://github.com/weavejester/ns-tracker

tools.namespace 처럼 lastModified를 캐싱해서 변경된 네임스페이스를 리스트로 리턴해준다. `:reload`를 직접하지 않고 변경된 네임스페이스만 리스트로 주기 때문에 `:reload`를 직접해줘야하지만 다양하게 활용 할 수 있어 개발용 라이브러리들에 많이 사용되고 있다.

### 사용 예

#### ring-devel

ring.middleware.reload/wrap-reload

ring 미들웨어로 핸들러를 실행할 때마다 ns-tracker로 변경된 네임스페이스를 가져와 reload 한 후에 핸들러를 실행해주는 미들웨어다. ring 개발환경에서 사용된다.

#### io.pedestal/pedestal.service-tools

io.pedestal.service-tools.dev/watch

pedestal에서 제공하고 있는 라이브러리로 `watach` 함수를 실행하면 쓰레드가 뜨고 0.5초마다 ns-tracker를 실행해서 주기적으로 변경된 네임스페이스를 reload 해준다.
