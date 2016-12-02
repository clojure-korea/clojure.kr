---
layout: post
title: Clojure 1.8.0에 추가되는 Direct Linking
date: 2015-12-18 15:09:10 +0900
author: Eunmin Kim
---

클로저 1.8.0에 Direct Linking이라는 방식이 추가된다고 합니다. (https://github.com/clojure/clojure/blob/master/changes.md)
그럼 Direct Linking이 뭔지 한번 살펴보겠습니다.

문서 상에서는 Direct Linking을 사용하려면 컴파일 옵션에 "-Dclojure.compiler.direct-linking=true"를 주고 컴파일하면 된다고 합니다.

예제를 보기 전에 먼저 설명을 적어보면 클로저는 실행할 때 심볼 값과 실제 값을 연결해주는 Var라는 구조를 사용합니다. 따라서 런타임에 심볼 값이 유지되고 심볼 값으로 동적으로 값을 찾을 수 있습니다. 예를 들어 프로그램 입력에 심볼 값을 주고 런타임에 그런 심볼 값을 찾아서 값을 사용할 수 있습니다. 보통 다른 프로그램들이 컴파일하면 변수 값(또는 주소 값)으로 바뀌어 직접 값을 참조하는 것과 다르게 동작한다고 볼 수 있습니다.

이런 구조는 프로그램에 동적인 부분을 더 해주는데요. Var를 한번 뒤져서 실제 값(자바 인스턴스 값)을 찾아야 하기 때문에 미세하게 성능 상 불리한 점이 있습니다.

Direct Linking은 코드에 심볼로 작성된 값을 Var를 뒤지지 않고 직접 값으로 연결됩니다.

따라서 alter-var-root 같은 심볼에 연결된 값을 변경하는 함수들이 제대로 동작하지 않습니다. 그럼 예제를 통해 한번 확인해봅시다.

```clojure
(defproject direct-link "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.8.0-RC4"]]
  :jvm-opts ["-Dclojure.compiler.direct-linking=true"])
```

먼저 `project.clj` 파일을 보면 `1.8.0-RC4` 버전을 사용하는 것과 `:jvm-opt`에  Direct Linking을 사용한다는 옵션을 줬습니다.

확인해볼 코드는 다음과 같습니다. `foo`라는 심볼에 `nil`을 리턴하는 함수 값을 연결하고 실행시 `alter-var-root`로 `foo`에 새로운 함수 값 (인자를 그대로 리턴하는 값)을 연결하고 `foo`를 실행해 봅니다.

```clojure
(ns direct-link.core)

(def foo (fn [x] nil))

(defn -main [& args]
  (alter-var-root #'foo (fn [_] (fn [x] x)))
  (println (foo 1)))
```

Direct Linking을 사용하지 않았다면 출력 값은 `1`이지만 실제로는 `nil`이 출력됩니다.

```bash
$ lein run -m direct-link.core
nil
```

위에서 설명한 것처럼 `println`에서 사용되고 있는 `foo`라는 값은 컴파일 시점의 `foo`의 값인 `nil`을 리턴하는 함수 값이기 때문에 `nil`이 출력되었습니다

클로저 1.8.0에는 Direct Linking을 사용하면서도 기존 처럼 동적인 기능을 사용할 수 있습니다. `^:redef`라는 메타데이터가 추가 되었는데요. 위 예제를 가지고 확인해보겠습니다.

```clojure
(ns direct-link.core)

(def ^:redef foo (fn [x] nil))

(defn -main [& args]
  (alter-var-root #'foo (fn [_] (fn [x] x)))
  (println (foo 1)))
```

```bash
$ lein run -m direct-link.core
1
```

`alter-var-root`가 잘 동작합니다. 또 다른 방법은 기존에 사용하던 `^:dynamic` 메타데이터를 사용하는 것입니다. 사실 기존 코드에서 클로저 관행상 동적으로 다시 바인딩 되는 것들은 `:dynamic` 메타데이터와 `*심볼*` 네이밍으로 구분해줬기 때문에 실제로 혼란은 많이 없을 것 같습니다.

```clojure
(ns direct-link.core)

(def ^:dynamic *foo* (fn [x] nil))

(defn -main [& args]
  (alter-var-root #'*foo* (fn [_] (fn [x] x)))
  (println (*foo* 1)))
```

Direct Linking은 동적인 부분을 포기하고 성능이 조금 더 좋게 할 수 있는 방법이고 잘 이해하고 사용한다면 좀 더 효율적인 클로저 코드를 작성할 수 있을 것 같습니다.
