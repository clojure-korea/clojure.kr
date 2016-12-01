---
layout: post
title: Clojure test에서 setup/teardown
date: 2015-03-24 00:00:03 +0900
author: Eunmin Kim
---

Clojure test에서 setup과 teardown은 clojure.test에 있는 use-fixtures를 사용하면 된다. use-fixtures는 아래와 같이 사용한다.

```clojure
(defn my-test-fixture [f]
        (create-db)
        (f)
        (destroy-db))

(use-fixtures :once my-test-fixture)

(deftest test-using-db
  (is ...
))
```

use-fixture의 첫번째 인수는 :once 또는 :each를 지정할 수 있고 각각 전체 테스트에서 한번 수행할지 각 테스트 마다 수행할지를 지정할 수 있다. 두번째 인수는 테스트 함수를 파라미터로 받는 fixture 함수를 정의 하면 된다. 파라미터로 받는 함수 앞 뒤로 각각 setup 함수와 teardown 함수를 실행해주면 된다.
