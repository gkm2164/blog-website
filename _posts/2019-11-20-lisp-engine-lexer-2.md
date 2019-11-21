---
title: "LISP 엔진 만들기 - Lexer - 2"
categories:
  - Lengine
tags:
  - Scala
  - Token
---

이제 어휘를 붙여봅시다. 여러가지 접근 방법이 있겠지만, Lengine에서는 다음과 같은 자료 구조를 따릅니다.

```scala
sealed trait LispToken // Token 의 상위 클래스

case object LeftParenthesis extends LispToken

```