---
layout: post
title: Html form Hierarchy
date: 2015-03-27 15:09:10 +0900
author: yoohyeon Jang
---

웹 어플리케이션의 api를 호출 할 때, 아마도 json 형식으로 주로 호출하다보니 전통적인 html-form-submit 의 인풋 엘리먼트를 위한 함수들은 의외로 상당히 등한시 되는 듯 합니다. 많이 찾아보았는데 없네요.. 그래도 놀이삼아 프로젝트를 굴리는데는 페이지 기반으로 url 을 호출하는것이 아무래도 쉽기에 form-submit 으로 넘겨받은 데이터의 hierarchy 를 (map 만) 분석하는 함수를 만들어 보았습니다.

```clojure
(defn parse-param [v]
  (if (re-matches #"^\d+\.?\d*?" v)
    (read-string v)
    v))

(defn multi-path->map [m keyseq v]
  (if (empty? keyseq)
    (parse-param v)
    (let [[k & ks] keyseq
          node  ((keyword k) m)]
      (assoc m
             (keyword k)
             (multi-path->map
              (if node
                node
                {}) ks v)))))

(defn handle-type [m]
  (letfn
      [(iter [keys values result]
         (if (empty? keys)
           result
           (let [[k & ks] keys
                 [v & vs] values]
             (iter ks vs
                   (multi-path->map
                    result
                    (clojure.string/split k #"\.")
                    v)))))]
    (iter (keys m) (vals m) {})))

>  (handle-type {"ab.cd.fff" "3" , "ab.aa" "4", "ab.ab.cc" "5", "ab.cd.bbb" "asdf"})
; => {:ab {:cd {:fff 3, :bbb "asdf"}, :ab {:cc 5}, :aa 4}}
```
