---
layout: post
title: Leiningen에 Repository 추가하기
date: 2015-03-24 00:00:20 +0900
author: eunmin
---

별도의 Repository를 사용하려면 `project.clj`에 `:repositories`키로 다음과 같이 Repository를 추가할 수 있다.

```Clojure
:repositories [["java.net" "http://download.java.net/maven/2"]
                 ["sonatype" {:url "http://oss.sonatype.org/content/repositories/releases"
                              ;; If a repository contains releases only setting
                              ;; :snapshots to false will speed up dependencies.
                              :snapshots false
                              ;; Disable signing releases deployed to this repo.
                              ;; (Not recommended.)
                              :sign-releases false
                              ;; You can also set the policies for how to handle
                              ;; :checksum failures to :fail, :warn, or :ignore.
                              :checksum :fail
                              ;; How often should this repository be checked for
                              ;; snapshot updates? (:daily, :always, or :never)
                              :update :always
                              ;; You can also apply them to releases only:
                              :releases {:checksum :fail :update :always}}]
                 ;; Repositories named "snapshots" and "releases" automatically
                 ;; have their :snapshots and :releases disabled as appropriate.
                 ;; Credentials for repositories should *not* be stored
                 ;; in project.clj but in ~/.lein/credentials.clj.gpg instead,
                 ;; see `lein help deploying` under "Authentication".
                 ["snapshots" "http://blueant.com/archiva/snapshots"]
                 ["releases" {:url "http://blueant.com/archiva/internal"
                              ;; Using :env as a value here will cause an
                              ;; environment variable to be used based on
                              ;; the key; in this case LEIN_PASSWORD.
                              :username "milgrim" :password :env}]]
```
