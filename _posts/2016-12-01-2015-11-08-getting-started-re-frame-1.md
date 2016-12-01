---
layout: post
title: Re-frame 시작하기 (1)
date: 2015-11-08 15:09:10 +0900
author: Eunmin Kim
---

https://github.com/Day8/re-frame

## Re-frame이란?

Re-frame은 Reagent와 ClojureScript로 Single Page Application을 만들기 위한 패턴이다.

Reagent는 ReactJS Wrapper이기 때문에 뷰를 위한 라이브러리다. 실제 규모있는 어플리케이션 개발을
하려면 코드가 많아 뷰에 모든 코드를 작성할 수 없기 때문에 이벤트 처리 코드와 뷰에 제공해야할 데이터 처리등의
코드를 적절히 구분해 작성할 필요가 있다. 보통 GUI 프로그래밍에서는 MVC 처럼 데이터, 컨트롤, 뷰의 역할을
나눠 처리하도록 작성하거나 MVVM, MVP와 같이 역할을 다르게 나눠 코드를 작성한다.
Re-frame도 이 패턴들 처럼 적절하게 역할을 나눠 규모있는 GUI 어플리케이션을 잘 작성할 수 있도록 도와주는
패턴이다. 하지만 ClojureScript는 함수형 패러다임에 기반하기 때무에 기존의 OOP에서 많이 사용하는 패턴들과는
다른 점들이 있다.

## 튜토리얼의 목적

이 튜토리얼은 Re-frame에 대한 자세한 부분을 다루기 보다 Todo 어플리케이션 예제를 통해 Re-frame의 구조를
알아보려고 한다. 자세한 부분은 Re-frame 프로젝트(https://github.com/Day8/re-frame)에 있는
자료를 참고하기 바란다.

## Todo 어플리케이션

GUI 프로그래밍에서 예제로 많이 다루고 있는 Todo 어플리케이션을 만들어보면서 Re-frame에 대해 알아보려고 한다.
Todo 어플리케이션은 re-frame 리파지토리 examples/todomvc에 있는 예제로 이 예제를 순서대로 진행해가면서
자연스럽게 re-frame을 설명하려고 한다. 내용이 많아 포스팅을 3번으로 나눠 설명하겠다.

### 프로젝트 생성

Re-frame은 프로젝트 템플릿을 제공하고 있지만 하나씩 알아보기 위해 figwheel 템플릿을 사용해서 프로젝트를
만들자.

```bash
lein new figwheel todomvc
```

프로젝트를 만들고 바로 `figwheel`을 실행 해보자.

```bash
cd ./todomvc
lein figwheel
...
Figwheel: Starting server at http://localhost:3449
...
```

3349 포트로 figwheel 서버가 뜨고 브라우저로 `http://localhost:3449`에 접속해보면 프로젝트 템플릿이
생성한 페이지를 볼 수 있다.

### Re-frame dependency 추가

`project.clj`를 열어 `:dependencies`에 `[re-frame "0.5.0"]`을 추가하고 다시 `lein figwheel`을
실행한다.

### Figwheel 템플릿에서 불필요한 부분을 지우기

Figwheel이 생성한 템플릿에서 사용하지 않을 부분은 제거하자.

먼저 `resources/public/index.html`에 아이디가 `app`인 `div`안에 있는 내용은 아래와 같이 삭제한다.

```html
...
<body>
  <div id="app"></div>
  <script src="js/compiled/todomvc.js" type="text/javascript"></script>
</body>
...
```

`src/todomvc/core.cljs` 파일을 열어 `(enable-console-print!)`를 빼고 모두 지운다.

```clojure
(ns ^:figwheel-always todomvc.core)

(enable-console-print!)
```

### Reagent 컴포넌트로 뷰 만들기

먼저 화면에 표시될 todo 뷰를 Reagent 컴포넌트 형태로 `todomvc.views` 네임스페이스에 작성해 보자.

```clojure
(ns todomvc.views)

(defn todo-input []
  [:input {:type "text"}])

(defn todo-list []
  [:ul.todo-list
   [:li
    [:div.view
     [:label "우유 구입"]]]
   [:li
    [:div.view
     [:label "시리얼 구입"]]]])

(defn todo-app []
  [:div
   [:section.todoapp
    [:header#header
     [:h1 "todos"]
     [todo-input]]
    [:div
     [:section.main
      [todo-list]]]]])
```
새로운 todo를 입력할 입력 창과 샘플 todo 항목 두개를 표시하는 목록을 만들었다.

이제 `todomvc.core`를 열어 아이디가 `app`인 `div`에 `todo-app` 컴포넌트를 마운트해보자.

```clojure
(ns ^:figwheel-always todomvc.core
  (:require [reagent.core :as reagent]
            [todomvc.views]))

(enable-console-print!)

(reagent/render [todomvc.views/todo-app] (.getElementById js/document "app"))
```

`reagent.core`와 `todomvc.views` 네임스페이스를 `require`하고 `reagent/render`로
`todo-app` 뷰를 마운트 했다.
브라우저를 확인해 보면 방금 작성한 뷰가 화면에 표시되는 것을 볼 수 있다.

### reagent atom으로 todo 데이터를 표시하기

이제 샘플 데이터를 reagent atom에 넣고 컴포넌트에서 표시해보자.

`todomvc.db` 네임스페이스를 만들고 여기에 reagent atom 형태로 샘플 데이터를 넣자.

```clojure
(ns todomvc.db
  (:require [reagent.core :refer [atom]]))

(def app-db (atom {:todos [{:id 1
                            :title "우유 구입"}
                           {:id 2
                            :title "시리얼 구입"}]}))
```

그리고 위에서 만든 데이터를 뷰에서 사용하도록 뷰를 고쳐보자.

```clojure
(ns todomvc.views
  (:require [todomvc.db :refer [app-db]]))

(defn todo-input []
  [:input {:type "text"}])

(defn todo-item [todo]
  [:li
   [:div.view
    [:input.toggle {:type "checkbox"}]
    [:label (:title todo)]]])

(defn todo-list [todos]
  [:ul.todo-list
   (for [todo todos]
     ^{:key (:id todo)} [todo-item todo])])

(defn todo-app []
  (let [todos (:todos @app-db)]
    (fn []
      [:div
       [:section.todoapp
        [:header#header
         [:h1 "todos"]
         [todo-input]]
        [:div
         [:section.main
          [todo-list todos]]]]])))
```

`todomvc.db`에 있는 reagent atom에서 `:todos` 값을 가져와 `todo-list`에 전달하고 `todo-list`는
각 항목을 표시할 `todo-item` 컴포넌트에 각각의 항목을 넘겨줬다.

### Re-frame으로 todo 데이터 가져오기

Re-frame은 어플리케이션에서 사용하는 데이터를 하나로 유지한다. 이 데이터를 데이터베이스처럼 사용하고
이 데이터베이스에서 필요한 데이터를 가져오기 위해 Query Layer라고 부르는 레이어를 통해 데이터를 가져온다.

Query Layer는 re-frame의 subscription이라는 이름으로 구현된다.
subscription은 일반 함수로 데이터베이스를 파라미터로로 받고 필요한 부분을 리턴해주는 함수다.
그리고 이 함수는 이 쿼리를 사용하기 위한 유일한 키를 가지고 등록해 사용한다.

그러면 아까 만든 데이터에서 `:todos`라는 데이터를 가져오는 subscription을 `todomvc.subs` 네임스페이스에
만들고 등록해보자.

```clojure
(ns todomvc.subs
  (:require-macros [reagent.ratom :refer [reaction]])
  (:require [re-frame.core :refer [register-sub]]))

(register-sub
  :todos
  (fn [db _]
    (reaction (:todos @db))))
```

Subscription 함수는 첫번째 파라미터로 reagent atom이 넘어오는데 이 atom 값이 위에서 말한 전체 어플리케이션에서
유일한 데이터베이스 값이다. 두번째 파라미터는 subscription을 사용할 때 추가적으로 넘길 수 있는 파라미터가 넘어온다.

그리고 이 subscription을 등록하기 위해 re-frame의 `register-sub` 함수에 `:todos`라는 키로 등록 했다.

이제 subscription 구현을 보자. subscription은 reagent atom인 db에서 `:todos`를 가져와 `reaction` 매크로로
감싸 리턴한다. subscription의 리턴 값은 reagent atom 값이어야 하는데 보통 subscription은 데이터베이스의 값을 계산하는
로직이 들어가기 때문에 이 구문 자체를 reagent의 `reaction`으로 감싸 리턴한다. 그래서 데이터베이스의 값이 변경 되는 경우
이 값을 사용하는 부분은 값이 다시 계산되어 자동으로 컴포넌트가 다시 랜더링 된다.

이제 이 subscription을 어디서든 사용할 수 있도록 `todomvc.core`에 `require`해 준다.

```clojure
(ns ^:figwheel-always todomvc.core
  (:require [reagent.core :as reagent]
            [todomvc.subs]
            [todomvc.views]))
```

#### subscription 사용하기

뷰에서 앞서 만든 subscription을 통해 데이터를 가져오도록 고쳐보자.

```clojure
(ns todomvc.views
  (:require [re-frame.core :refer [subscribe]]))

;; 중간 생략

(defn todo-list [todos]
  [:ul.todo-list
   (for [todo @todos]
     ^{:key (:id todo)} [todo-item todo])])

(defn todo-app []
  (let [todos (subscribe [:todos])]
    (fn []
      [:div
       [:section.todoapp
        [:header#header
         [:h1 "todos"]
         [todo-input]]
        [:div
         [:section.main
          [todo-list todos]]]]])))
```

고쳐진 부분을 보면 reagent atom을 직접 가져오는 부분 대신 `subscribe` 함수와 subscription의 키를
가지고 데이터를 가져왔다. 그리고 가져온 데이터가 reagent atom 형식이기 때문에 `todo-list`에서도 todos 파라미터를
`@todos`로 가져오도록 고쳤다.

저장하고 브라우저를 확인해 보면 입력 창만 남고 데이터는 비어있는 것을 볼 수 있다.

### 데이터 초기화 하기

`todomvc.db`에 있는 샘플 데이터는 re-frame의 subscription과 관계가 없기 때문에 화면에 표시되지 않는다.

subscription에서 사용하는 유일한 데이터베이스 값에 데이터를 넣어 줘야 화면에 표시될 것이다.
re-frame에서 이 유일한 데이터베이스를 변경할 수 있도록 handler라는 기능을 제공한다.
handler는 데이터베이스를 파라미터로 받아서 새로운 데이터베이스 값을 리턴하는 함수다.
handler도 subscription과 같이 유일한 키로 등록 해서 사용한다.

`todomvc.handlers` 네임스페이스에 데이터를 초기화하는 핸들러를 만들어보자.

```clojure
(ns todomvc.handlers
  (:require [re-frame.core :refer [register-handler]]
            [todomvc.db :refer [app-db]]))

(register-handler                 
  :initialize-db
  (fn [db _]
    app-db))
```

handler 함수는 subscription과 비슷하게 첫번째 파라미터로 유일한 데이터베이스를 받고 두번째 파라미터로
handler를 사용할 때 넘긴 파라미터 값이 넘어온다. subscription과 다른 점은 유일한 데이터베이스인 db값이
reagent atom 형태가 아니고 일반 값이라는 점과 리턴 값 역시 reagent atom이 아니고 일반 값이어야 한다는 점이다.

등록하는 방법은 subscription과 유사하게 `register-handler`로 handler를 구분하는 유일한 키와
handler 함수를 등록 한다.

위에서 만든 `:initialize-db` handler는 `todomvc.db`에 있는 값을 그대로 리턴해주는 예제다.
먼저 만든 `todomvc.db`에 `app-db`는 reagent atom 값이기 때문에 일반 값으로 바꿔주자.

```clojure
(ns todomvc.db)

(def app-db {:todos [{:id 1
                      :title "우유 구입"}
                     {:id 2
                      :title "시리얼 구입"}]})
```

#### :initialize-db handler를 사용해 초기화 하기

`:initialize-db` handler 사용해보자.
데이터 초기화이기 때문에 `todomvc.core`에서 해주면 적절하다.
handler를 사용하려면 re-frame의 `dispatch` 함수를 사용한다.
형태는 `subscribe`와 유사하다.

`todomvc.core`를 열어 `todomvc.handlers`를 `require`해주고 `:initialize-db` 핸들러를
추가하자.

```clojure
(ns ^:figwheel-always todomvc.core
  (:require [reagent.core :as reagent]
            [re-frame.core :refer [dispatch]]
            [todomvc.handlers]
            [todomvc.subs]
            [todomvc.views]))

(enable-console-print!)

(dispatch [:initialize-db])
(reagent/render [todomvc.views/todo-app] (.getElementById js/document "app"))
```

브라우저를 확인해보면 초기화 한 데이터가 화면에 표시되는 것을 볼 수 있다.

### 정리

지금까지 만들어본 todo 어플리케이션을 정리하면 다음과 같다.

- todomvc.db에 초기화 할 데이터를 선언한다.
- todomvc.handlers에 데이터를 초기화하는 handler를 만들고 등록한다.
- todomvc.subs에 데이터를 접근할 subscription을 만들고 등록한다.
- todomvc.views에서 subscribe 함수로 가져온 reagent atom 데이터를 reagent 컴포넌트로 구성한다.
- todomvc.core에 todomvc.handlers, todomvc.subs를 등록하고 데이터를 초기화하기 위해 dispatch
  함수로 초기화 handler를 사용했다.

다음 장에서는 todo 어플리케이션에 새로운 todo를 추가하고 todo에 대한 완료 처리등을 추가하면서
re-frame에 대해 더 자세히 알아보자.
