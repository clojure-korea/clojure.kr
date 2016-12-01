---
layout: post
title: monad in clojure 2
date: 2016-01-16 15:09:10 +0900
author: Eunmin Kim
---

이 글은 monad in clojure(https://github.com/khinsen/monads-in-clojure) 문서를 보고 정리 한 내용입니다.
번역이 아니고 읽고 정리한 버전이라 조금 다를 수 있습니다.

지난 글에 이어 두번째 파트를 정리해 봅니다.

먼저 글에서 identity 모나드와 maybe 모나드에 대해서 알아봤습니다. 오늘은 sequence(또는 list) 모나드에
대해 살펴보면서 먼저 나왔던 `m-result`를 언제 사용하면 좋을지 알아보겠습니다.

## sequence 모나드

sequence 모나드도 자주 사용되는 모나드라 클로저에서 기본 기능으로 제공하고 있습니다. 바로 `for` 구문인데요.
사용하는 예를 볼까요?

```clojure
(for [a (range 5)
      b (range a)]
  (* a b))
;; => (0 0 2 0 3 6 0 4 8 12)
```

모양은 `let` 구문과 비슷하게 생겼네요. 하지만 다르게 동작하겠죠? 큰 차이점은 시퀀스만 바인딩할 수 있다는 것입니다.
그리고 결과는 각 바인딩이 중첩된 반복문을 실행한 것과 같은 결과를 가져요.
클로저에서 제공하는 `for` 구문은 `algo.monad` 라이브러리에 `sequence-m` 이름으로 만들어져 있어요.

```clojure
(domonad sequence-m
  [a (range 5)
   b (range a)]
  (* a b))
;; => (0 0 2 0 3 6 0 4 8 12)  
```

`sequence-m`도 먼저 알아본 `indentity-m`, `maybe-m` 처럼 `m-bind`와 `m-result`가 정의되어
있을 텐데요. 어떻게 정의 되어 있는지 한번 같이 구현해 봅시다.

먼저 `identity-m`에서 정의 했던 `m-bind`를 다시 살펴보면 다음과 같습니다.

```clojure
(defn m-bind-indentity [value function]
  (function value))
```

`identity-m`의 `m-bind`는 함수와 값의 순서만 바꿔 실행해주는 함수였습니다.

`sequence-m`의 `m-bind`는 값이 시퀀스이기 때문에 모든 값에 대해 함수를 적용해 줘야하고 잘 알고 있는
잘 알고 있는 `map` 함수를 쓰면 되겠죠.

```clojure
(defn m-bind-first-try [sequence function]
  (map function sequence))
```

이렇게 정의 한 `m-bind` 함수를 실행해볼까요?

```clojure
(m-bind-first-try (range 5)  (fn [a]
(m-bind-first-try (range a)  (fn [b]
(* a b)))))
```

실행 해보면 `(() (0) (0 2) (0 3 6) (0 4 8 12))`가 나옵니다.
우리가 원하는 것은 `(0 0 2 0 3 6 0 4 8 12)`이라 뭔가 더 해줘야겠죠.

안쪽에 있는 괄호를 벗겨주면 되는데 `concat` 함수를 쓰면 되겠네요.

```clojure
(concat [1] [2 3] [4 5 6])
;; => (1 2 3 4 5 6)
```

`concat`을 적용해 `m-bind`를 다시 작성해 봅시다.

```clojure
(defn m-bind-second-try [sequence function]
  (apply concat (map function sequence)))
```

실행해 볼까요?

```clojure
(m-bind-second-try (range 5)  (fn [a]
(m-bind-second-try (range a)  (fn [b]
(* a b)))))
;; => java.lang.IllegalArgumentException: Don't know how to create ISeq from: Integer
```

안타깝게도 에러가 발생했네요. 왜 그럴까요? 문제는 가장 마지막의 계산 결과는 숫자를 담고 있는 시퀀스인데
이 숫자 값들을 `concat` 하려고 했기 때문입니다.

```clojure
... (map (fn [b] (* a b)) [...]) ...
;; => (1 2 3 4 5 ...)
... (apply concat (1 2 3 4 ...))...
;; => java.lang.IllegalArgumentException: Don't know how to create ISeq from: Integer
```

그럼 어떻게 해야할까요? 좋은 방법은 마지막 계산의 결과를 시퀀스로 만들어주면 될것 같습니다.

```clojure
(apply concat '('(1) '(2) '(3) ...))
;; => (1 2 3 ...)
```

다시 해볼까요?

```clojure
(m-bind-second-try (range 5)  (fn [a]
(m-bind-second-try (range a)  (fn [b]
(list (* a b))))))
```

잘 동작하네요. 지난번 봤던 `m-result`를 보셨죠?
`m-result`는 마지막 계산 결과를 감싼 함수인데 이 함수에 `list`로 바꿔주는 기능을 넣으면 되겠군요.

```clojure
(defn m-bind [sequence function]
  (apply concat (map function sequence)))

(defn m-result [value]
  (list value))
```

```clojure
(m-bind (range 5)  (fn [a]
(m-bind (range a)  (fn [b]
(m-result (* a b))))))
```

## `m-result`, `m-bind`, `모나드 표현식`의 관계

`sequence-m`이나 `maybe-m` 처럼 이미 만들어진 모나드를 사용하는 것은 지금 해본 것처럼 쉽게 사용할 수 있습니다.
그럼 새로운 모나드 규칙을 정의하려면 어떻게 하면 될까요? `m-bind`, `m-result`를 정의 하면 됩니다.
그런데 `m-bind`와 `m-result`를 정의하는데 규칙이 있습니다. 왜냐구요? 이 규칙에 맞게 정의하면 모나드로 더 편리한
것들을 많이 할 수 있기 때문이죠. 그래서 이 규칙에 맞도록 작성하면 여러가지 편리한 모나드 함수를 사용할 수 있어요.

참고로 `모나드 표현식`은 identity 모나드 예제에 나왔던 `(inc a)`와 같은 바인딩 되는 곳에 있는 구문을 말합니다.

첫번째 조건 입니다.

```clojure
(= (m-bind (m-result value) function)
   (function value))
```

함수에 값을 적용한 결과는 `m-result`에 값을 적용한 값과 함수를 `m-bind`에 넘긴 결과와 같아야 한다는 뜻입니다.

두번째는 다음과 같습니다.

```clojure
(= (m-bind monadic-expression m-result)
   monadic-expression)
```

 모나드 표현식과 `m-result`를 `m-bind`에 넘기면 모나드 표현식과 같아야한다는 뜻이죠.

이것을 보기 쉽게 `domonad`로 작성해보면 다음처럼 됩니다.

```clojure
(= (domonad
     [x monadic-expression]
      x)
   monadic-expression)
```

모나드 표현식을 어떤 값에 바인딩하고 그 값을 리턴하면 당연히 모나드 표현식과 같아야한다는 뜻입니다.

마지막 조건입니다.

```clojure
(= (m-bind (m-bind monadic-expression
                   function1)
           function2)
   (m-bind monadic-expression
           (fn [x] (m-bind (function1 x)
                           function2))))
```

조금 복잡해 보이니 역시 `domonad`로 작성해보면 어떤 의미인지 조금 더 알아보기 쉬울 것 같습니다.

```clojure
(= (domonad
     [y (domonad
          [x monadic-expression]
          (function1 x))]
     (function2 y))
   (domonad
     [x monadic-expression
      y (m-result (function1 x))]
     (function2 y)))
```

쉽지 않네요.ㅋㅋㅋ 그래도 잘 살펴보면 `y`에 바인딩 된 모나드 실행 결과와 `(m-result (function1 x))`와
같아야 하는 것을 볼 수 있습니다.

그럼 이 규칙을 지켜서 만든 모나드로 편리하게 쓸 수 있는 몇가지 함수를 알아보겠습니다.

## m-lift

`m-lift` 함수는 숫자와 함수를 인자로 받아 함수를 리턴하는 함수입니다. 이 함수는 모나드의 결과 함수로 사용할 수 있습니다.
예제를 보는게 이해하기 빠르겠죠?

```clojure
 (defn nil-respecting-addition
   [x y]
   (domonad maybe-m
     [a x
      b y]
     (+ a b)))
```

`x`와 `y`를 인자로 받고 인자로 받은 `x`와 `y`에 각각 `a`와 `b`를 바인딩하고 `+` 함수의 인자로 넘겨주는 구문인데요.
이 구문에 `maybe-m`을 적용해 중간 값이 `nil`인 경우 `nil`을 리턴되게 했습니다.

위와 같은 구문은 아래와 같이 `m-lift`로 바꿔보면 아래와 같습니다.

```clojure
(def nil-respecting-addition
  (with-monad maybe-m
    (m-lift 2 +)))
```

`m-lift`는 두 개(`x`와 `y`)의 인자를 받아 각각 내부적으로 바인딩해서 `+` 함수의 인자로 넘깁니다.

또 다른 예를 볼까요?

```clojure
(defn my-map [f xs]
  (domonad sequence-m
    [x xs]
    (f x)))
```

위 예제는 `sequence-m`를 이용해서 `map` 함수를 구현한 건데요. 인자로 받은 시퀀스 `xs`를 `x`에 바인딩하고
인자로 받은 `f`를 적용해 `map`처럼 동작하게 했습니다.

이것도 역시 `m-lift`로 볼까요?

```clojure
(with-monad sequence-m
  (defn my-map
    [f xs]
    ((m-lift 1 f) xs)))
```

`m-lift` 결국 함수의 인자들을 순서대로 바인딩해서 함수를 적용해 주되 그 과정에 모나드를 적용할 수있는 함수입니다.

## m-seq

또 다른 모나드 함수로 `m-seq`가 있습니다. `m-seq`는 단순히 인자로 시퀀스를 받아 시퀀스를 내어주는데요.
시퀀스의 항목 순서대로 바인딩해서 모나드를 적용하고 결과를 시퀀스로 내어줍니다.

```clojure
(m-seq [a b c])
```
는 다음과 같이 동작하는 것이지요.

```clojure
(domonad
  [x a
   y b
   z c]
  '(x y z))
```

다음은 `m-seq`에 `sequence-m`를 적용해 인자로 받은 시퀀스로 구성할 수 있는 n개 항목을 가지는 튜플을 만드는 예제입니다.

```clojure
(with-monad sequence-m
  (defn ntuples [n xs]
    (m-seq (replicate n xs))))

(ntuples 3 [1 2])
;; => ((1 1 1) (1 1 2) (1 2 1) (1 2 2) (2 1 1) (2 1 2) (2 2 1) (2 2 2))
```

## m-chain

마지막으로 알아볼 모나드 함수는 `m-chain`입니다. `m-chain` 함수는 실행할 함수 목록을 시퀀스로 받아 모나드를 적용해
연속으로 함수를 실행해주는 함수를 만들어 줍니다.

`(m-chain [a b c])`이 구문은 아래 처럼 동작합니다.

```clojure
(fn [arg]
  (domonad
    [x (a arg)
     y (b x)
     z (c y)]
    z))
```

아래는 `m-chain`을 사용한 예제인데요. Java 클래스와 계층의 단계를 받아 구현하고 있는 것들을 시퀀스에 담아
리턴해주는 코드입니다.

```clojure
(with-monad sequence-m
  (defn n-th-generation
    [n cls]
    ( (m-chain (replicate n parents)) cls )))

(n-th-generation 0 (class []))
;; => (clojure.lang.PersistentVector)
(n-th-generation 1 (class []))
;; => (clojure.lang.IObj clojure.lang.IEditableCollection clojure.lang.APersistentVector
;;     clojure.lang.IReduce)
(n-th-generation 2 (class []))
;; => (clojure.lang.IMeta java.lang.Comparable java.io.Serializable clojure.lang.IPersistentVector
;;     java.util.List clojure.lang.IHashEq clojure.lang.AFn java.lang.Iterable java.util.RandomAccess
;;     clojure.lang.IReduceInit)
```

이번 장에서는 `sequence-m`를 가지고 `m-result`를 어떻게 사용하는지 알아봤고 모나드를 더 편리하게 사용할 수 있는
`m-lift`, `m-seq`, `m-chain` 함수들에 대해 알아봤습니다. 또 이런 함수들을 만들고 사용하기 위해
`m-result`, `m-bind`, `모나드 표현식`이 지켜야할 조건에 대해서도 알아습니다.

그럼 3장에서 다시 뵙겠습니다 ^^
