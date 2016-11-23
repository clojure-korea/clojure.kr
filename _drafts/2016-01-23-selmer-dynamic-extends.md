---
layout: post
title:
date: 2015-02-09 15:09:10 +0900
author: Eunmin Kim
---

Selmer는 extends 태그로 템플릿 레이아웃을 지정할 수 있습니다. extends의 사용 예를 볼까요?

base.html
```
<html>
  <head></head>
  <body>
    {% block yield %}
    {% endblock %}
  </body>
</html>
```

index.html
```html
{% extends "base.html" %}

{% block yield %}
<h1>index</h1>
{% endblock %}
```

`index.html`에 있는 `yield` 블럭은 `base.html`에 `yield` 블럭안에 들어가게 됩니다.

만약 `base.html`을 동적으로 사용하고 싶다면 어떻게 해야할까요? 다음과 같이 하면 될까요?

```clojure
(render-file "index.html" {:layout "base.html"})
```

```html
{% extends layout %}

{% block yield %}
<h1>index</h1>
{% endblock %}
```

하지만 이렇게 하면 에러가 납니다. Selmer는 기본적으로 `extends` 태그 안에서 파라미터 값을 사용할 수 없습니다.

Selmer는 `render-file` 하는 경우 파싱하기 전에 `selmer.template-parser/preprocess-template` 함수에서 `extends` 태그와 `include` 태그를 처리하고 파싱하게 됩니다.

`preprocess-template` 함수는 `extends` 태그를 만나면 `get-parent`라는 함수로 `extends` 태그 다음에 오는 `"base.html"` 문자열을 가져와 따옴표를 제거하고 `base.html`을 가져와 처리하게 됩니다.

그럼 어떻게 하면 동적으로 `extends`를 할 수 있을까요? 쉬운 방법은 `get-parent`를 다음과 같이 `redefs`하는 것입니다.

```clojure
(with-redefs [selmer.template-parser/get-parent (constantly "base.html')]
  (render-file "index.html" {}))
```

```html
{% extends %}

{% block yield %}
<h1>index</h1>
{% endblock %}
```

위 `with-redefs`는 아래와 같이 매크로로 만들어주면 더 알아보기 쉬울 것 같습니다.

```clojure
(defmacro with-layout [layout & body]
  `(with-redefs [selmer.template-parser/get-parent (constantly ~layout)]
     ~@body))

(with-layout "base.html"
  (render-file "index.html" {}))
```

동작은 잘 하지만 한가지 문제는 캐싱 문제입니다. Selmer는 `render-file`에서 템플릿 파일이 변경되었는지 파일 변경 시간을 가지고 있다가 변경이 된 경우에 만 `paser`를 하기 때문에 레이아웃을 동적으로 넘겨도 실제 파일이 변경되지 않으면 변경된 레이아웃이 반영되지 않는 문제가 있습니다.

이것을 해결하려면 `render-file`에 옵션을 이용해 `cache`를 직접 다루면 가능합니다.
