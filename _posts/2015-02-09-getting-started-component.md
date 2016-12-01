---
layout: post
title: 컴포넌트 빠르게 살펴보기
date: 2015-02-09 15:09:10 +0900
author: Eunmin Kim
---

## 컴포넌트

컴포넌트(https://github.com/stuartsierra/component)는 런타임 상태를 관리해주는 클로저 프레임워크 입니다.
Stuart Sierra가 작성했고 천천히 알아보시려면 Clojure/West 2014에서 발표한 영상을 참고하시면 됩니다.(https://www.youtube.com/watch?v=13cmHf_kt-Q)
이 글은 빠르게 컴포넌트를 적용하고 어떻게 돌아가는지 대략적으로 알아볼 수 있는 글입니다.
컴포넌트의 목적이나 자세한 사용법을 알고 싶다면 Github 페이지(https://github.com/stuartsierra/component)를 방문해 보세요.

## 컴포넌트 의존성

```clojure
[com.stuartsierra/component "0.2.2"]
```

위와 같고 최신 버전은 Clojars(http://clojars.org/)에서 검색하시면 됩니다.

## 컴포넌트의 구성

컴포넌트는 어떻게 보면 객체지향의 의존성 주입 프레임워크(Dependency Injection)와 비슷합니다.
컴포넌트는 Component와 Component를 모아 의존성을 주입하고 연결 시켜주는 System으로 구성됩니다.

## 컴포넌트 예제

### 데이터 베이스 컴포넌트 만들기

컴포넌트는 Lifecycle 프로토콜을 구현하는 클로저 record로 작성합니다.
아래는 컴포넌트 Github에 있는 데이터베이스 Component의 예제입니다.

```clojure
(defrecord Database [host port]
  ;; Lifecycle를 구현 합니다.
  component/Lifecycle

  (start [component]
    (println ";; Starting database")
    ;; 'start' 메서드는 컴포넌트가 시작되는 부분으로
    ;; 데이터베이스 연결이나 스레드 풀을 만들거나 공유 상태를 초기화 하기 적절한 곳입니다.
    (let [conn (connect-to-database host port)]
      ;; 'start' 메서드에는 파라미터로 맵 형식의 componet 자신을 받습니다.
      ;; 컴포넌트에 추가할 데이터가 있다면 componet 맵에 추가하고
      ;; 반드시 컴포넌트를 리턴해야합니다.
      (assoc component :connection conn)))

  (stop [component]
    (println ";; Stopping database")
    ;; 'stop' 메서드는 컴포넌트가 종료되는 부분으로
    ;; 리소스를 해제하기 적합한 곳입니다.
    (.close connection)
    ;; 역시 컴포넌트 자신을 리턴해야합니다.
    (assoc component :connection nil)))
```

위의 예제는 데이터베이스 연결을 맺고 연결을 컴포넌트의 `:connection`이라는 키로 보관하고 있다가 종료될때 연결을 해제하는 역할을 합니다.
보관하고 있는 `:connection`은 다른 컴포넌트나 함수에서 불러 사용할 수 있습니다.
컴포넌트를 정의 했다면 컴포넌트를 생성해야합니다. 컴포넌트 생성은 아래와 같이 생성하는 함수를 만들어 사용하면 편리합니다.

```clojure
(defn new-database [host port]
  (map->Database {:host host :port port}))
```

생성하는 함수는 일반 클로저 함수 입니다. 주목할 부분은 `map->Database` 부분인데 위에서 정의한 `Database` 컴포넌트를 생성해서 리턴해주는 일을 합니다.
이 함수는 맵 형식의 하나의 파라미터를 받는데 컴포넌트에 넘겨질 부가적인 정보를 맵에 담아 넘기게 됩니다.
위에서 `Database` 컴포넌트를 정의할 때 `defrecord`와 이름 다음에 파라미터와 같이 생긴 `[host port]` 백터를 볼 수 있습니다.
이 백터는 `map->Database`의 맵 인자의 키 이름과 매칭이 되어 값이 들어가게 됩니다.
만약 `(map->Database {:host "127.0.0.1" :port 3306})`라고 컴포넌트를 생성한다면 컴포넌트 구현 부에서 `host`값에 "127.0.0.1" `port`값에 3306이 들어옵니다.

### 데이터베이스 컴포넌트를 사용하는 컴포넌트 만들기

이제 `Database` 컴포넌트를 사용하는 컴포넌트를 만들어 봅시다. 이것도 역시 Github 에제에 있는 소스입니다.

```clojure
(defn get-user [database username]
  ;; 위에서 만든 맵 형식의 Database 컴포넌트를 파라미터롤 받아 :connection 키로 연결을 가져와 쿼리를 실행하는 함수 입니다.
  (execute-query (:connection database)
    "SELECT * FROM users WHERE username = ?"
    username))

(defn add-user [database username favorite-color]
  ;; 위에서 만든 맵 형식의 Database 컴포넌트를 파라미터롤 받아 :connection 키로 연결을 가져와 쿼리를 실행하는 함수 입니다.
  (execute-insert (:connection database)
    "INSERT INTO users (username, favorite_color)"
    username favorite-color))

(defrecord ExampleComponent [options database cache scheduler]
  component/Lifecycle

  ;; cache, scheduler를 받지만 여기서는 사용하지 않습니다.
  (start [this]
    (println ";; Starting ExampleComponent")
    ;; 위의 예제에서는 'start' 메서드에서 받는 이름을 component라고 했는데 여기서는 this라고 바꿔봤습니다.
    ;; 컴포넌트가 시작되면 컴포넌트에서 받은 database 값을 가지고 get-user 함수로 admin 사용자를 가져옵니다.
    ;; 그리고 다른 컴포넌트나 함수에서 사용할 수 있도록 컴포넌트 자신에 맵에 :admin 이라는 키로 값을 추가합니다.
    (assoc this :admin (get-user database "admin")))

  (stop [this]
    (println ";; Stopping ExampleComponent")
    ;; 'stop' 메서드는 아무것도 하지 않고 컴포넌트 자신을 리턴합니다.
    this))

(defn example-component [config-options]
  ;; options 파라미터를 가지고 컴포넌트를 만듭니다.
  (map->ExampleComponent {:options config-options
                          :cache (atom {})}))
```

위의 예제는 Database 컴포넌트를 사용하기 위한 예제 컴포넌트인 ExampleComponent 컴포넌트 입니다.
Database 컴포넌트에 있는 `:connection`을 가지고 쿼리를 수행하는 예제 입니다.
여기서 주의깊게 살펴봐야할 부분은 컴포넌트의 파리미터 부분과 컴포넌트를 생성할 때 넘기는 인자입니다.
컴포넌트를 생성할 때는 키로 `:options`와 `:cache`를 넘겼습니다.
이것은 ExampleComponent의 `options`와 `cache` 파라미터로 받게됩니다.
그럼 `database`와 `scheduler`는 누가 넘겨주는 걸까요?
이것은 프레임 워크에 의해 주입되는 컴포넌트 들로 다음에 바로 다룰 시스템에서 정의해주게 됩니다.

### 시스템

시스템은 컴포넌트들 간의 의존성을 정의해 주고 컴포넌트 전체를 시작 또는 종료하는 기능을 합니다.
시스템은 컴포넌트에서 제공하는 `system-map`이라는 매크로로 만들 수 있습니다.
아래는 위에서 만들었던 `Database`와 `ExampleComponent` 그리고 만들지는 않았지만 만들었다고
가정한 `Schedule` 컴포넌트의 관계를 맺어줍니다.

```clojure
(defn example-system [config-options]
  (let [{:keys [host port]} config-options]
    (component/system-map
      :db (new-database host port)
      :scheduler (new-scheduler)
      :app (component/using
             (example-component config-options)
             {:database  :db
              :scheduler :scheduler}))))
```

`system-map`는 키와 값으로 정의하게 되는데 중요한 부분은 `using`이라는 함수를 사용해서 컴포넌트에 다른 컴포넌트를 주입하는 부분입니다.
위에서는 `ExampleComponent`를 생성하는 `example-component` 함수를 부르고 그 결과를 `using`이라는 함수의 첫번째 인자로 넘겨줍니다.
두번째 인자는 맵인데 여기에 앞서 생성해놓은 `ExampleComponent`에서 사용할 다른 컴포넌트의 키를 넘겨주게 됩니다.
키를 넘기는 형식은 앞의 키는 컴포넌트 정의에서 사용한 파라미터 이름이고 뒤의 키는 `system-map`에 있는 생성된 컴포넌트의 키입니다.
만약에 앞의 키와 뒤의 키가 같다면 아래처럼 짧게 쓸 수 도 있습니다.

```clojure
(component/system-map
      :database (new-database host port)
      :scheduler (new-scheduler)
      :app (component/using
             (example-component config-options)
             [:database :scheduler]))
```

### 컴포넌트의 시작과 종료

모든 컴포넌트를 시작하기 위해서는 시스템을 시작해주면 됩니다.
시스템을 시작하려면 `start` 함수를 사용하면 됩니다.

```clojure
(def system (example-system {:host "dbhost.com" :port 123}))

(alter-var-root #'system component/start)

(alter-var-root #'system component/stop)
```

컴포넌트의 `start` 함수는 만들어진 시스템을 찾아 시작하고 결과로 시스템 자체를 리턴합니다.
만약 종료를 하기 위해서는 시작된 시스템을 가지고 있어야하기 때문에 위의 예제 처럼 `start` 후에
Var 바인딩을 변경합니다.
`start`를 하면 시스템에 정의된 순서대로 컴포넌트들의 `start` 메서드가 호출 됩니다.
컴포넌트를 종료하기 위해서는 `stop`을 호출하면 되는데 `stop`은 `start`의 역순으로 컴포넌트들의
`stop` 메서드가 호출 됩니다.

## 마무리

에러 처리나 재시작, 커스터마이징 관련된 정보들은 Github 페이지를 참조하세요.
