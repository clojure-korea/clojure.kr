---
layout: post
title: Clojure with-open 매크로
date: 2015-03-24 00:00:10 +0900
author: Eunmin Kim
---

Clojure에는 Java의 open-close 형태의 클래스를 편리하게 사용할 수 있는 `with-open` 매크로를 제공한다. `with-open`은 Java의 아래와 같은 패턴을 매크로로 제공한다.

```Java
try {
    client = new Closable();

    ...
} finally {
    client.close();
}
```

사용법은 아래와 같다.

```Clojure
(with-open [out-data (io/writer out-file)]
      (csv/write-csv out-data out-sos)))
```

finally에 close가 아닌 다른 메서드를 호출한다면 매크로를 참조하여 만들면 된다. 아래는 close 대신 stop을 부르는 `with-start`라는 매크로다.  매크로의 파라미터를 검사하는 `assert-args`도 추가했다.

```Clojure
(defmacro assert-args
  [& pairs]
  `(do (when-not ~(first pairs)
         (throw (IllegalArgumentException.
                  (str (first ~'&form) " requires " ~(second pairs) " in " ~'*ns* ":" (:line (meta ~'&form))))))
     ~(let [more (nnext pairs)]
        (when more
          (list* `assert-args more)))))

(defmacro with-start
  [bindings & body]
  (assert-args
     (vector? bindings) "a vector for its binding"
     (even? (count bindings)) "an even number of forms in binding vector")
  (cond
    (= (count bindings) 0) `(do ~@body)
    (symbol? (bindings 0)) `(let ~(subvec bindings 0 2)
                              (try
                                (with-open ~(subvec bindings 2) ~@body)
                                (finally
                                  (. ~(bindings 0) stop))))
    :else (throw (IllegalArgumentException.
                   "with-start only allows Symbols in bindings"))))
```
