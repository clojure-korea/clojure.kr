---
layout: post
title: Clojure Reload에 대해서 2
date: 2015-12-12 15:09:10 +0900
author: eunmin
---

"Clojure Reload에 대해서"에 이어 두번째로 `:reload`에 대한 내용을 적어본다. 클로저 리로드에 대해서 익숙하지 않다면 먼저 작성한 "Clojure Reload에 대해서"를 보면 이 글을 이해하기 쉬울 것이다.

두번째로 다루는 내용은 다른 네임스페이스에 있는 값을 바인딩해서 사용하고 있는 경우 다른 네임스페이스의 값이 바뀌고 그 네임스페이스가 `:reload`되어도 변경된 내용이 반영되지 않는다는 내용이다. 말로 설명하면 어려우니 역시 예제로 알아보자.

```clojure
(ns value-reload.core
  (:require [value-reload.calc :refer [dollar->won]]))

(defn foo [x]
  (dollar->won x))
```

```clojure
(ns value-reload.calc)

(defn dollar->won [dollar]
  (* dollar 1170))
```

위 예제는 `value-reload.core/foo`에서 `value-reload.calc/dollar->won` 함수를 사용하는 예제다. 달러를 원화로 바꿔주는 예제다.

`lein repl`을 실행해서 동작하는지 확인해보자.

```clojure
user=> (require '[value-reload.core :refer :all])
nil
user=> (foo 1)
1170
```

잘 동작한다. 환율을 변경하고 변경된 환율이 잘 반영되는지 보자.

```clojure
(ns value-reload.calc)

(defn dollar->won [dollar]
  (* dollar 1175))
```

```clojure
user=> (require 'value-reload.calc :reload)
nil
user=> (foo 1)
1175
```

변경된 `value-reload.calc`만 `:reload`하고 다시 실행했다. 예상한데로 변경된 내용이 잘 반영되었다.

그럼 `value-reload.core`에서 사용하는 `dollar->won`을 직접 사용하지 않고 다른 이름으로 바인딩해서 사용해보자.

```clojure
(ns value-reload.core
  (:require [value-reload.calc :refer [dollar->won]]))

(def f dollar->won)

(defn foo [x]
  (f x))
````

```clojure
user=> (require 'value-reload.core :reload)
nil
user=> (foo 1)
1175
```

`value-reload.core`가 변경되었기 때문에 `:reload`해주고 실행했다. 역시 잘 실행된다. 그럼 다시 `value-reload.calc`에 있는 환율값을 바꾸고 실행해보자.

```clojure
(ns value-reload.calc)

(defn dollar->won [dollar]
  (* dollar 1170))
```

```clojure
user=> (require 'value-reload.calc :reload)
nil
user=> (foo 1)
1175
```

고친 `value-reload.calc` 네임스페이스를 `:reload` 했지만 결과는 바뀌지 않았다.

무슨 문제인지 `value-reload.core`를 살펴보자.

```clojure
(ns value-reload.core
  (:require [value-reload.calc :refer [dollar->won]]))

(def f dollar->won)

(defn foo [x]
  (f x))
```

`value-reload.core`가 `require`되면 `def` 구문에서 `f` 심볼과 그 다음에 나오는 값을 연결해 Var를 생성한다. 그 다음에 나오는 값이 `dollar->won` 심볼이기 때문에 `dollar->won`의 값으로 치환한다. 결국 `f`는 `dollar->won`의 함수 값을 가진다.

만약 `value-reload.calc`를 `:reload` 하면 `dollar->won`은 새로운 값이 생기는데 클로저는 내부적으로 인스턴스로 표현된다. 클로저의 함수는 함수 하나 하나마다 클래스로 표현되며 값은 그 클래스의 인스턴스가 된다.

`f`에 연결되어 있는 `dollar->won` 값은 이전에 연결된 인스턴스 값이고 `:reload`해서 새로 만들어진 `dollar->won` 값은 아직 어디에도 연결되지 않았다.

`value-reload.core`를 다시 로드해주면 `def` 구문을 해석하고 다시 `f`값과 새로운 `dollar->won` 값이 연결되 잘 동작한다.

```clojure
user=> (require 'value-reload.core :reload)
nil
user=> (foo 1)
1170
```


```clojure
user=> f
#object[value_reload.calc$dollar__GT_won 0xe187596 "value_reload.calc$dollar__GT_won@e187596"]
user=> value-reload.calc/dollar->won
#object[value_reload.calc$dollar__GT_won 0xe187596 "value_reload.calc$dollar__GT_won@e187596"]
user=> (require 'value-reload.calc :reload)
nil
user=> f
#object[value_reload.calc$dollar__GT_won 0xe187596 "value_reload.calc$dollar__GT_won@e187596"]
user=> value-reload.calc/dollar->won
#object[value_reload.calc$dollar__GT_won 0x7d6d044a "value_reload.calc$dollar__GT_won@7d6d044a"]
user=> (require 'value-reload.core :reload)
nil
user=> f
#object[value_reload.calc$dollar__GT_won 0x7d6d044a "value_reload.calc$dollar__GT_won@7d6d044a"]
user=> value-reload.calc/dollar->won
#object[value_reload.calc$dollar__GT_won 0x7d6d044a "value_reload.calc$dollar__GT_won@7d6d044a"]
```

0xe...으로 시작하는 것이 인스턴스 아이디 값이다. `f`와 `dollar->won` 인스턴스 값의 변화를 살펴보면 위에서 설명한 말을 알 수 있다.

그래서 이렇게 사용하면 변경된 네임스페이스만 `:reload`해서는 원하는데로 동작하지 않을 수 있다.

먼저 소개한 `ns-tracker`, `tools.namespace/refresh`는 이런 문제를 해결하기 위해 네임스페이스 의존 관계를 찾아 모두 `:reload` 해주기 때문에 신경쓰지 않고 사용할 수 있다.

만약 이런 라이브러리를 사용할 수 없는 특별한 상황에서 값으로 참조해서 사용해야 하는 경우가 있다면 아래와 같은 방법을 쓸 수 있다.

```clojure
(ns value-reload.core)

(def f #(deref #'dollar->won))

(defn foo [x]
  ((f) x))
```

`f`가 `dollar->won` 함수 값을 직접 바인딩하지 않고 `f`를 `dollar->won` 심볼에 연결된 Var의 값을 가져오는 `deref` 함수를 사용해 `foo`를 실행할 때마다 Var를 찾도록 한 예다. 이렇게 하면 `value-reload.calc`만 `:reload`해도 매번 Var를 찾기 때문에 변경된 값을 사용할 수 있다.
