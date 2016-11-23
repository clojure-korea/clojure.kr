---
layout: post
title:
date: 2015-02-09 15:09:10 +0900
author: Eunmin Kim
---

이 글은 monad in clojure(https://github.com/khinsen/monads-in-clojure) 문서를 보고 정리
한 내용입니다. 번역은 아니고 읽고 정리한 버전이라 조금 다를 수 있습니다.

다른 함수형 언어들 처럼 클로저도 모나드를 몰라도 쓸 수 있습니다. 저도 잘 모르는데 불편함 없이 잘 쓰고 있습니다.
모나드를 알아보려는 이유는 뭔가 고수들만 알것 같은 모나드가 대체 뭔가에 대한 호기심으로 찾아보기 시작했습니다.
그러던 중 클로저로 모나드에 대한 내용을 설명한 글을 발견했는데 읽어보니 조금은 알 것 같아 정리해봤습니다.

글에서 모나드란 "Monads are about composing computational steps into a bigger multi-step computation."라고 합니다.
글로 봐서는 감이 잘 잡히지 않고 그냥 예제를 통해 감을 잡는 것이 좋을 것 같습니다.

참! 이 예제는 설명을 돕기 위해 `clojure.algo.monads` 라이브러리를 사용합니다.

https://github.com/clojure/algo.monads

## identity 모나드

먼저 가장 쉬운 모나드라고 하는 identity 모나드에 대해 알아보겠습니다. 사실 클로저는 내장 기능으로 identity 모나드를 제공하고 있습니다.
바로 `let` 구문인데요. 다음 예제를 살펴봅시다.

```clojure
(let [a  1
      b  (inc a)]
  (* a b))
```

클로저를 아는 사람이라면 아주 쉬운 구문이죠. `let` 구문은 로컬 바인딩하고 그 바인딩 심볼을 사용할 수 있는 구문이죠.
위 계산은 `(* 1 (inc 1))` 입니다. 이렇게 안쓰고 `let`을 쓰는 이유는 억지로 예제를 만드려는 것도 있겠지만 간혹 계산의 중간 과정을 이름으로
바인딩 해서 중복 계산을 줄이거나 가독성을 좋게 하기 위해서 `let` 구문을 사용합니다.

그럼 계산 과정을 하나씩 살펴봅시다.

- 먼저 `1` 값을 계산합니다. `1`을 계산하면 `1`이죠. 그걸 `a`라는 심볼에 바인딩합니다.
- 그리고 `(inc a)`를 계산합니다. 그걸 `b`라는 심볼에 바인딩합니다.
- 그리고 `(* a b)`를 계산합니다. 그 값을 리턴합니다.

각 계산 과정은 앞선 계산 결과에 의존합니다. 그럼 `let` 구문이 없다면 어떻게 표현 할 수 있을 까요?
클로저는 함수형 언어를 기반으로 하고 있기 때문에 함수 파라미터 바인딩과 closure로 다음과 같이 표현 할 수 있습니다.

```clojure
( (fn [a]  ( (fn [b] (* a b)) (inc a) ) )  1 )
```

익명 함수를 사용해서 눈에 잘 안들어 오시죠? 그래도 짧은 편이니 천천히 살펴봅시다.

가장 바깥 쪽에 있는 `a`를 받는 함수 부터 보시죠.

```clojure
( (fn [a] ... ) 1)
```

먼저 `a`를 받는 함수에 `1`을 넘겨서 함수를 실행합니다. 다음으로 그 안에 있는 `b`를 받는 함수를 살펴봅시다.

```clojure
... ( (fn [b] ... ) (inc a) ) ...
```

`b`를 인자로 받는 함수는 이 함수를 감싸고 있는 함수에서 받은 `a`에 `(inc a)`를 해서 `b`에 넘겨 실행합니다.
마지막으로 `b`를 받는 함수 안쪽을 살펴봅시다.

```clojure
... (fn [b] (* a b)) ...
```

결과로 `(* a b)`를 리턴합니다.

천천히 살펴본 것 처럼 위 식은 `let` 구문 처럼 동작합니다. 하지만 알아보기 힘들죠 최대한 들여쓰기를 해볼까요?

```clojure
((fn   [a]  
  ((fn [b]       
                (* a b))
       (inc a)))
       1)
```

아무리 들여쓰기를 해도 `a`가 `1`이고 `b`가 `(inc a)`고 결과가 `(* a b)`라고 알아보기는 힘들것 같습니다.

하지만 클로저는 함수형 언어이기 때문에 머리를 잘 쓰면 잘 보이게 할 수 있을 것 같습니다. 하지만 저는 머리를 잘 못쓰기 때문에 다른 사람이 써놓은 방법으로 다시 작성해봅시다.

아래와 같이 도움이 함수를 만들어 봅시다.

```clojure
(defn m-bind [value function]
  (function value))
```

간단한 함수 입니다. 값을 먼저 쓰고 다음에 함수를 쓰면 함수를 그 값으로 실행해 줍니다. 위에서 들여쓰기를 했을 때는 함수가 먼저 오고 인자가 뒤에 오기 때문에 표현에 한계가 있었습니다. 위에 만든 `m-bind`는 인자가 먼저오고 함수가 뒤에 오기 때문에 표현의 제약을 덜어줄 것 같습니다. `m-bind`의 도움을 받아 다시 작성해봅시다.

```clojure
(m-bind 1        (fn [a]
(m-bind (inc a)  (fn [b]
        (* a b)))))
```

이제 `a`가 `1`이고 `b`가 `(inc a)`이고 결과가 `(* a b)`라는 것이 눈에 잘 들어오네요. 그래도 만족스럽지 못하시죠? 클로저는 매크로라는 강력한 감춤기능이 있기 때문에 매크로를 동원해서 규칙적으로 반복되고 있는 `m-bind`와 `(fn [심볼])`을 감출수 있습니다. 여기서 그 매크로를 작성하진 않겠습니다.

다행이 `clojure.algo.monads` 라이브러리에는 이런 기능을 하는 매크로가 있습니다.
바로 `domonad` 매크로인데요. `domonad` 매크로를 사용해서 표현해봅시다.

```clojure
(domonad identity-m
  [a  1
   b  (inc a)]
  (* a b))
```

와! `let` 처럼 되었네요. `let` 대신 `domonad`를 쓰고 뒤에 `identity-m`을 빼면 `let`과 같네요.
`macroexpand-1`로 한번 풀어볼까요?

```clojure
(clojure.algo.monads/with-monad identity-m
  (m-bind 1 (fn [a] (m-bind (inc a) (fn [b] (m-result (* a b)))))))
```

익숙한 `m-bind`와 함수가 보입니다. 그리고 낯선 `with-monad`와 마지막 결과 값을 `m-result`로 보내 결과를 리턴하는 구문이 보이네요.

`m-result` 함수는 그냥 그 값을 리턴하는 함수라고 한다면 똑같죠.

```clojure
(defn m-result [value]
  value)
```

그럼 `with-monad`와 `identity-m`은 어떻게 설명하면 좋을까요?

만약 `m-bind`와 `m-result`를 마음데로 정의해 놓고 거기에 이름을 붙이면 뭔가 유용한 다른 것도 만들 수 있을 겁니다.

`identity-m`은 `m-bind` 함수와 `m-result` 함수를 아래와 같이 정의한 것이라고 볼 수 있습니다.

```clojure
(defn m-bind [value function]
  (function value))

(defn m-result [value]
  value)
```

그리고 `with-monad`는 뒤에 나오는 `identity-m`에 정의된 `m-bind`와 `m-result` 값을 가져와서
`m-bind`와 `m-result`에 로컬 바인딩해서 구문을 실행해주는 것으로 볼 수 있습니다.

이것을 identity 모나드라고 합니다.

## maybe 모나드

그럼 `m-bind`를 좀 변형해서 좀 더 가치있는 기능을 만들어 봅시다.

만약 `m-bind`가 다음과 같이 정의 되어 있다고 합시다.

```clojure
(defn m-bind [value function]
  (if (nil? value)
      nil
      (function value)))
```

identity 모나드에 정의한 `m-bind`와 거의 비슷하지만 한가지 다른 점은  `value`가 `nil`이면 함수를 실행하지 않고 `nil`를 리턴하게 됩니다.

그럼 여기에 있는 `m-bind`는 어떻게 동작할까요? 다시 identity 모나드 예제를 살펴봅시다.

```clojure
(defn f [x]
  (domonad identity-m
    [a  x
     b  (inc a)]
    (* a b)))
```

만약 `x` 값이 `nil` 이라면 `a`에 `x`를 적용하는 것은 문제 없이 동작하지만 다음 계산 단계인 `(inc a)`할 때
`(inc nil)`이 되어 예외가 발생하고 계산이 중단됩니다. 중단되지 않고 결과를 `nil`로 리턴하려면
계산 마다 `nil`인지 아닌지 확인하는 코드를 넣어줘야하는데요. 위에 만든 `m-bind`를 사용한다면 중간에 `nil`인
값이 들어오면 다음 함수를 실행하지 않고 `nil`을 리턴하게 됩니다.

이것을 maybe 모나드라고 부릅니다. 그리고 자주 사용되는 모나드라 `identity-m` 처럼 `clojure.algo.monads` 라이브러리에 정의되어 있습니다.

```clojure
(defn f [x]
  (domonad maybe-m
    [a  x
     b  (inc a)]
    (* a b)))
```

위 예제를 실행해보면 `x`가 `nil`이여도 예외가 발생하지 않고 `nil`이 리턴되는 것을 볼 수 있습니다.

다음 글에서는 `m-result`를 사용하는 예제인 sequence 또는 list 모나드라고 부르는 것 대해서 알아보고 `m-bind`와 `m-result`를
작성하기 위한 몇 가지 규칙에 대해서 알아보겠습니다. 또 `m-bind`, `m-result` 외에 몇가지 유용한 도우미 함수에 대해서도
알아보겠습니다.

:)
