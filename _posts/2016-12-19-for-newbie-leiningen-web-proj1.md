---
layout: post
title: "[따라하기] Luminus project 만들기" 
date:   2016-12-19 15:50:00 +0900 
author: dayoungle
---


### 목차

---
- 신규 프로젝트를 Luminus를 이용해서 생성해보기 
- Luminus 기본 프로젝트 구조 
	- 프로젝트의 root directory에서 프로젝트를 실행시켜본다
	- localhost:3000 에 접속했을 때 luminus welcome page가 나오면 프로젝트 기본 틀 완성!
- src 디렉토리에 생성된 파일들의 역할을 파악해본다
	- src/clj/sample-proj/core.clj
	- src/clj/sample-proj/handler.clj
	- src/clj/sample-proj/routes/home-routes.clj

---


### 신규 프로젝트를 Luminus를 이용해서 생성해보기 

```
lein new luminus sample-proj
```
참고: [http://www.luminusweb.net/](http://www.luminusweb.net/)


---


### Luminus 기본 프로젝트의 구조

``` 
.
├── Capstanfile ;The Capstanfile is a YAML config file for specifying Capstan images.
├── Dockerfile ;For docker image
├── Procfile ;used to facilitate Heroku deployments
├── README.md
├── env ;config를 dev/prod/test 별로 분리해서 사용하도록 미리 디렉토리 생성이 되어있음
│   ├── dev
│   │   ├── clj
│   │   │   ├── sample_proj
│   │   │   │   ├── dev_middleware.clj
│   │   │   │   └── env.clj
│   │   │   └── user.clj
│   │   └── resources
│   │       ├── config.edn
│   │       └── logback.xml
│   ├── prod
│   │   ├── clj
│   │   │   └── sample_proj
│   │   │       └── env.clj
│   │   └── resources
│   │       ├── config.edn
│   │       └── logback.xml
│   └── test
│       └── resources
│           ├── config.edn
│           └── logback.xml
├── profiles.clj ; profile 별로 설정 구분을 위해 사용하는 profile 설정 파일
├── project.clj ; leiningen으로 project config와 dependency 를 관리하는 파일
├── resources ;env 폴더 하위에 있는 것과 달리 static 한 html/img/js/sql 파일들을 모아두는 곳. 
│   ├── docs
│   │   └── docs.md
│   ├── public
│   │   ├── css
│   │   │   └── screen.css
│   │   ├── favicon.ico
│   │   ├── img
│   │   └── js
│   └── templates
│       ├── about.html
│       ├── base.html
│       ├── error.html
│       └── home.html
├── src ;코드 소스를 관리하는 디렉토리
│   └── clj
│       └── sample_proj
│           ├── config.clj
│           ├── core.clj
│           ├── handler.clj
│           ├── layout.clj
│           ├── middleware.clj
│           └── routes
│               └── home.clj
└── test ;테스트 코드를 src와 같은 구조로 관리하는 디렉토리 
    └── clj
        └── sample_proj
            └── test
                └── handler.clj

26 directories, 30 files
```

- 프로젝트 생성 후 자동 생성되었으나 프로젝트 계획 상 불필요한 파일은 삭제함

---
- Captstanfile
- Dockerfile
- Procfile

---


---



#### 프로젝트의 root directory에서 프로젝트를 실행시켜본다 

```
~/sample-proj $ lein run 
```

---


#### localhost:3000 에 접속했을 때 luminus welcome page가 나오면 프로젝트 기본 틀 완성!

![leiningen_start](/assets/leiningen_start.png)



---



### src 디렉토리에 생성된 파일들의 역할을 파악해본다
src 디렉토리 하위에는 소스코드를 관리한다. 프로젝트의 이름이 root namespace로 결정된다. 

```
├── src ;코드 소스를 관리하는 디렉토리
│   └── clj
│       └── sample_proj
│           ├── config.clj
│           ├── core.clj
│           ├── handler.clj
│           ├── layout.clj
│           ├── middleware.clj
│           └── routes
│               └── home.clj
```



---


#### src/clj/sample-proj/core.clj 
- 메인함수가 포함된 application의 entry point라고 생각하면 된다 
- 서버의 시작과 중단을 관리하는 로직이 들어있고  mount를 이용하여 시작,중단 시 동작해야 할 기능이 설정되어있다.

```clojure 
(ns sample-proj.core
  (:require [sample-proj.handler :as handler]
            [luminus.repl-server :as repl]
            [luminus.http-server :as http]
            [sample-proj.config :refer [env]]
            [clojure.tools.cli :refer [parse-opts]]
            [clojure.tools.logging :as log]
            [mount.core :as mount])
  (:gen-class))

(def cli-options
  [["-p" "--port PORT" "Port number"
    :parse-fn #(Integer/parseInt %)]])

(mount/defstate ^{:on-reload :noop}
                http-server
                :start
                (http/start
                  (-> env
                      (assoc :handler (handler/app))
                      (update :port #(or (-> env :options :port) %))))
                :stop
                (http/stop http-server))

(mount/defstate ^{:on-reload :noop}
                repl-server
                :start
                (when-let [nrepl-port (env :nrepl-port)]
                  (repl/start {:port nrepl-port}))
                :stop
                (when repl-server
                  (repl/stop repl-server)))


(defn stop-app []
  (doseq [component (:stopped (mount/stop))]
    (log/info component "stopped"))
  (shutdown-agents))

(defn start-app [args]
  (doseq [component (-> args
                        (parse-opts cli-options)
                        mount/start-with-args
                        :started)]
    (log/info component "started"))
  (.addShutdownHook (Runtime/getRuntime) (Thread. stop-app)))

(defn -main [& args]
  (start-app args))

```

---


#### src/clj/sample-proj/handler.clj 
- application의 가장 기본이 되는 base routes를 정의한다. 
- app-routes 를 정의하면서 compojure 의 routes 기능을 활용한다. (compojure의 개념에 대해서는 차후에 정리하는 것으로 한다)
- app-routes에서 정의된 home-routes 에 정의된 api 주소들이 route로 등록이 된다.
- routes 함수 내에 인자로 사용된 wrap-routes들은 인자로 추가한 middleware로 route를 싼다(wrap)는 의미이다. 

``` clojure 
(ns sample-proj.handler
  (:require [compojure.core :refer [routes wrap-routes]]
            [sample-proj.layout :refer [error-page]]
            [sample-proj.routes.home :refer [home-routes]]
            [compojure.route :as route]
            [sample-proj.env :refer [defaults]]
            [mount.core :as mount]
            [sample-proj.middleware :as middleware]))

(mount/defstate init-app
                :start ((or (:init defaults) identity))
                :stop  ((or (:stop defaults) identity)))

(def app-routes
  (routes
    (-> #'home-routes
        (wrap-routes middleware/wrap-csrf)
        (wrap-routes middleware/wrap-formats))
    (route/not-found
      (:body
        (error-page {:status 404
                     :title "page not found"})))))


(defn app [] (middleware/wrap-base #'app-routes))

```

---


#### src/clj/sample-proj/routes/home-routes.clj 
- 위의 handler에서 등록한 route의 resource를 상술하도록 되어있다 
- 이때 compojure 의 defroutes를 사용해서 route를 손쉽게 정의할 수 있도록 샘플이 나와있다. 
	- 앞서 보았던 handler에서 routes로 등록된 routes의 이름 부분과 defroutes로 선언된 이름이 같아야 한다. 
- GET, POST와 같은 http 메서드들은 compojure에서 지원하는 것이다. 
- api를 지원하는 것 외에도 html을 응답으로 내려주거나 document를 내려주는 것도 가능하다. 
	
``` clojure
(ns sample-proj.routes.home
  (:require [sample-proj.layout :as layout]
            [compojure.core :refer [defroutes GET]]
            [ring.util.http-response :as response]
            [clojure.java.io :as io]))

(defn home-page []
  (layout/render
    "home.html" {:docs (-> "docs/docs.md" io/resource slurp)}))

(defn about-page []
  (layout/render "about.html"))

(defroutes home-routes
  (GET "/" [] (home-page))
  (GET "/about" [] (about-page)))
```
