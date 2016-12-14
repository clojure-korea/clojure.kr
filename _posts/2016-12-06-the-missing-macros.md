---
layout: post
title: (require '[plumbing.core :refer :all])
date: 2016-12-09 20:00:00 -0800
author: taeold
---

반년 전 코드 리뷰 중 나의 멘토가 "이 코드 좀 귀여운데?" 라는 코멘트를 달아준 적이 있다.

멘토는 웃으라고 한 얘기일 수도 있겠지만, 지금 생각해도 기분이 좋아지는 칭찬이다.코드를 _집필_ 한다고 하는것은 조금 오버라고 생각하는 사람도 잇겠지만, 나는 주어진 임무를 간결하고 부드럽게 수행하는 코드를 좋아하고, 그런 귀여운(?) 코드를 작성하기 위해 한 번이라도 더 내 머리를 굴려보곤 한다.

오늘 내가 clojure 코드를 귀엽게 작성할 때에 유용하게 쓰고 있는 라이브버리 하나를 소개해볼까 한다.

## [prismatic/plumbing](https://github.com/plumatic/plumbing)

plumbing은 [schema](https://github.com/plumatic/schema)로 유명한 prismatic (a.k.a. plumatic)에서 만든 유틸리티 라이브러리이다. plumbing에서는 clojure 코드를 작성할 때 유용한 여러 툴을 제공하는데, 나는 `plumbing.core` namespace에 있는 매크로들을 정말 좋아한다. 내가 자주 애용하는 매크로 몇 개를 함께 살펴보자.

### [for-map](https://github.com/plumatic/plumbing/blob/master/src/plumbing/core.cljx#L23)
클로져에서 제공하는 `for` 매크로는 기본적으로 sequence 돌려주지만, 사실 내가 필요한 건 map 이라면? `for-map`을 사용해보자.

```clojure
;; {:a 0 :b 1 :c 2}를 만들어보자

;; 이렇게게 쓰는것이 보편적이지만
(into {}
  (for [i [:a :b :c] j (range 3)]
    [i j]))
=> {:a 0 :b 1 :c 1}

;; 좀 더 귀엽게 써보자
(for-map [i [:a :b :c] j (range 3)]
  i j)
=> {:a 0 :b 1 :c 1}
```

### [map-vals/map-keys](https://github.com/plumatic/plumbing/blob/master/src/plumbing/core.cljx#L68)
`map` 함수는 참으로 멋진 함수이지만, map 데이터 스트럭쳐와는 궁합이 잘 맞지 않는다. 그럴 땐 `map-keys`와 `map-vals` 사용해보자.

```clojure
;; {:a 1 :b 2 :c 3} => {"a" 1 "b" 2 "c" 3}?

;; 생각보다 번거롭다
(->> {:a 1 :b 2 :c 3}
     ;; keyword 를 str으로 바꾸고
     (map (fn [k _] [(name k) v]))
     ;; sequence를 다시 map 으로 바꾸자
     (into {}))
=> {"a" 1 "b" 2 "c" 3}

;; 좀 더 귀엽게 써보자
(map-keys name {:a 1 :b 2 :c 3})
=> {"a" 1 "b" 2 "c" 3}
```

### [dissoc-in](https://github.com/plumatic/plumbing/blob/master/src/plumbing/core.cljx#L96)
`assoc-in`, `update-in`은 있지만 왜 `dissoc-in`은 없을까?

```clojure
;; {:a {:b 1 :c 2}}에서 :b 키를 dissoc 해보자

(update {:a {:b 1 :c 2}} :a #(dissoc % :b))
=> {:a {:c 2}}

;; 좀 더 귀엽게 써보자
(dissoc-in [:a :b] {:a {:b 1 :c 2}})
=> {:a {:c 2}}
```

### [update-in-when](https://github.com/plumatic/plumbing/blob/master/src/plumbing/core.cljx#L156)
`update` 함수는 map에 지정한 key가 존재한다는 가정을 세우지만, 세상은 그렇게 호락호락 하지 않다.

```clojure
;; 주어진 map에 :a 키가 있다면 increment 해보자

;; 이 코드엔 무서운 버그가 있다
;; hint: NullPointerException
(fn [x] (update x :a inc))

;; 이렇게 코드를 써볼 수 도있겟지만
(fn [x]
  (if (:a x)
    (update x :a inc)
    x))

;; 좀 더 귀엽게 써보자
(fn [x] (update-in-when x [:a] inc))
```

### [?>](https://github.com/plumatic/plumbing/blob/master/src/plumbing/core.cljx#L301)
Threading 매크로는 정말 깔삼하다. `?>`와 `?>>`는 threading 매크로를 더욱더 깔쌈하게 만들어준다.

```clojure
;; 만약 increment flag가 세워져 있다면 맵의 vals을 increment 해보자
(def increment? true)

;; annonymous fn를 사용해볼 수 있지만, 자연스럽지 않다
(->> {:a 0 :b 1 :c 2}
     (#(if increment? (map-vals inc %) %)))
=> {:a 1 :b 2 :c 3}

;; 좀 더 귀엽게 써보자
(->> {:a 0 :b 1 :c 2}
     (?>> increment? (map-vals inc)))
```  

### [fn->](https://github.com/plumatic/plumbing/blob/master/src/plumbing/core.cljx#L315)
Threading 매크로에 맛이 들이기 시작하면 모든 문제를 threading 매크로로 풀고 싶어 지는데, 이럴땐 `fn->`와 `fn->>`가 도움이 된다.

```clojure
;; map으 vals을  1) inc 하고 2) str 으로 바꾸자

;; 사실 난 이 코드에 아무런 문제가 없지만
(map-vals #(-> % inc str) {:a 0 :b 1 :c 2})

;; 아주 조금 더 귀엽게 써보자
(map-vals (fn-> inc str) {:a 0 :b 1 :c 2})
```

### and more
이외 수많은 매크로가 존재한다. 유용한 매크로는 얼마든 존재하겠지만, `prismatic/plumbing`에서 제공하는 매크로들은 "이 매크로들이 `clojore.core` 없다고?" 생각이 들 정도로 clojure 코드에 자연스럽게 녹아들 수 있다고 생각해 자주 애용하고 있다. 귀여운 코드를 작성하는데 즐거움을 느끼시는 분이라면 꼭 써보길.
