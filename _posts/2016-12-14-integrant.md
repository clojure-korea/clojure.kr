---
layout: post
title: Component 대신 Integrant 써볼까?
date:   2016-12-14 14:50:00 +0900
author: eunmin
---

오늘은 지난주에 올라온 따끈따끈한 라이브러리인 [Integrant](https://github.com/weavejester/integrant)에 대해 살펴보겠습니다.
[Integrant](https://github.com/weavejester/integrant)는 [Component](https://github.com/stuartsierra/component)와
[Mount](https://github.com/tolitius/mount)와 같이 클로저의 리소스 관리 라이브러리입니다.

[https://github.com/weavejester/integrant](https://github.com/weavejester/integrant)

## 리소스 관리 라이브러리

보통 클로저로 규모 있는 애플리케이션을 만들 때 리소스 관리 라이브러리를 사용하는데요.
웹 애플리케이션이라면 데이터베이스 커넥션 풀, 설정, 캐시, 인 웹서버와 같은 리소스들이 필요합니다.
이 리소스들은 보통 `def`를 사용해 `Var`로 초기화하는데요. `Var`는 생성 시점을 다룰 수 없기 때문에
리소스 간 의존성이 있다면 어떤 리소스를 먼저 만들고 어떤 리소스는 나중에 만들지 정할 수 없습니다.
그래서 의존성이 있는 리소스들은 일반적으로 엔트리 포인트인 `main` 함수에서 생성하고 `shutdown hook`
같은 곳에서 제거합니다. 하지만 리소스가 많아지면 리소스 초기화/해제 코드가 복잡해지므로 고치기 힘든 문제가 있습니다.
그래서 `Component`와 같은 리소스 의존성 및 라이프사이클을 관리하는 라이브러리들이 나왔습니다.

## Component

`Component`는 개념은 좋았지만 Record와 같은 자료형을 사용해서 리소스를 만들고 `start`/`stop`과 같은
프로토콜로 구현해줘야 합니다. 그래서 `Component`를 이용해서 애플리케이션을 작성하다 보면 애플리케이션 전체가
`Component`에 의존적이게 됩니다. 또 리소스를 사용하는 함수에 인자로 리소스를 항상 받아야 하므로 `Var`로
리소스를 정의해 놓고 사용하는 것처럼 편하지 않습니다.

컴포넌트에 대해서 더 알아보시려면 전에 작성한 [컴포넌트 빠르게 살펴보기](http://localhost:4000/getting-started-component)
글을 참고하세요.

```clojure
(defrecord Database [host port connection]
  component/Lifecycle
  (start [component]
    (println ";; Starting database")
    (let [conn (connect-to-database host port)]
      (assoc component :connection conn)))

  (stop [component]
    (println ";; Stopping database")
    (.close connection)
    (assoc component :connection nil)))

(defn example-system [config-options]
  (let [{:keys [host port]} config-options]
    (component/system-map
      :db (new-database host port)
      :scheduler (new-scheduler)
      :app (component/using
             (example-component config-options)
             {:database  :db
              :scheduler :scheduler}))))

(alter-var-root #'system component/start)
```

## Mount

`Mount`는 `Compnent` 보다 편리하게 리소스를 관리할 수 있는 라이브러리입니다. `Mount`의 가장 좋은 점은
리소스간의 의존성을 따로 정해주지 않아도 된다는 점입니다. 그럼 어떻게 의존성을 판단하냐고요? 의존성이라는 것은
한 리소스에서 다른 리소스를 필요하다는 말인데요. 결국 `require`로 다른 리소스를 사용하게 됩니다. `Mount`는
이 `ns`에 정의된 `require`의 관계를 분석해 의존성 맵을 자동으로 만들어 줍니다. 또 `Var`를 사용하기 때문에
어디서든 리소스에서 쉽게 접근할 수 있습니다. 단점이라고 한다면 `Var`를 사용하기 때문에 동시에 하나 이상의 상태를
가질 수 없습니다. 예를 들면 `test` 환경과 `repl` 환경을 동시에 가질 수 없습니다. 물론 `test` 할 때는
`test` 상태로 바꾸고 개발 모드일 때는 개발 모드로 실행하는 것은 쉽게 할 수 있습니다. 하지만 동시에 가지고 있을 수
없으므로 `repl` 상태와 `test` 상태를 동시에 유지하고 싶다면 `Var`를 따로 만들어야 합니다.

```clojure
(defstate db :start (connect config)
             :stop (disconnect mem-db))

(mount/start)
```

## Integrant

`Integrant`는 `Ring`을 만들고 여러 유명한 클로저 라이브러리를 소개한 `James Reeves(weavejester)`가
만든 리소스 관리 라이브러리입니다. `Integrant`는 `Mount`보다 `Component`와 비슷합니다. `Component` 보다
좋은 점은 리소스 의존성 설정을 코드 대신 `edn`이나 Map 데이터로 정의 할 수 있다는 점과 리소스 `start`/`stop` 등을
프로토콜이 아닌 멀티메서드로 구현하도록 제공하기 때문에 애플리케이션이 좀 더 유연해진다는 점입니다. 그 밖의
기능은 `Component`와 비슷합니다.

## Integrant 예제

### 프로젝트에 의존성 추가하기

```clojure
[integrant "0.1.4"]
```

### 네임스페이스

```clojure
(require '[integrant.core :as ig])
```

### 의존성 설정하기

- 맵으로 설정하기

```clojure
(def config
  {:adapter/jetty {:port 8080, :handler (ig/ref :handler/greet)}
   :handler/greet {:name "Alice"}})
```

- `edn`으로 설정하기

```clojure
{:adapter/jetty {:port 8080, :handler #ref :handler/greet}
 :handler/greet {:name "Alice"}}

(def config
  (ig/read-string (slurp "config.edn")))
```

### 리소스 초기화 함수 정의하기

```clojure
(defmethod ig/init-key :adapter/jetty [_ {:keys [handler] :as opts}]
  (jetty/run-jetty handler (-> opts (dissoc :handler) (assoc :join? false))))

(defmethod ig/init-key :handler/greet [_ {:keys [name]}]
  (fn [_] (resp/response (str "Hello " name))))
```

- `init-key` 멀티메서드는 두 번째 파라미터로 해당 키에 대한 설정 맵이 넘어옵니다.


### 리소스 해제 함수 정의하기

```clojure
(defmethod ig/halt-key! :adapter/jetty [_ server]
  (.stop server))
```

- `:handler/greet` 리소스는 해제 함수를 정의하지 않았습니다.


### 시스템 시작하기

`Integrant`도 `Component`처럼 리소스 전체를 편의상 시스템이라고 부릅니다.라이브러리 상에 시스템이란
용어는 따로 없으므로 편의상 시스템이라고 부르는 것 같습니다.

```clojure
(def system
  (ig/init config))
```

- 시스템을 시작하면 설정되어 있는 의존성 순서대로 리소스 초기화 함수가 실행됩니다.


### 시스템 종료하기

```clojure
(ig/halt! system)
```

### 정리

여기에서 소개하지 않았지만 `Integrant`는 `Component`처럼 `suspend`, `resume` 기능도 있습니다.
`Integrant`를 살펴본 느낌으로는 충분히 `Component`를 대체할 수 있을 것 같습니다. 하지만 `Mount`와 같은
편의성은 제공하지 않기 때문에 `Mount`는 다른 필요때문에 계속 쓰일 것 같습니다.
