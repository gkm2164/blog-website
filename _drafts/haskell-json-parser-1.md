---
title: "Making JSON Parser with Haskell - 1 (개요)"
categories:
  - Haskell
tags:
  - Haskell
  - State Monad
---

# Introduction

대개, 함수형 언어를 공부한다고 할 때 그것에 대한 적합한 예제를 찾기는 쉽지 않습니다. 여러 언어를 공부해본 결과, 예를들어 웹을 위한 백엔드를 개발한다 하면 보통은 Java를 많이 쓸 것입니다. 굳이 함수형을 쓴다고 하면 Play Framework, Akka가 있는 Scala가 있으며, 물론 표현력이 좋은것은 알겠으나, 이는 "서버를 함수형 언어로도 짤 수 있고, 다른 언어로 짠 것보다 훨씬 Reliable합니다!"라고 말하는 것과 비슷합니다. 결과적으로 함수형 언어는 데이터 처리에 용이한 언어가 되었고, 실제 웹과 관련한 분야에서는 Reactive 기법이 주로 쓰이겠으나, 이 역시나마 다른 언어, JavaScript(TypeScript) 에서 충분히 지원하고 있어 실제 함수형 언어가 쓰일 수 있는 곳은 드문 듯 합니다.

Usually, when learning a programming language, especially functional one, it's hard to find a good example to study. Of course there is good toy examples. But in case of solving real problem, there are already good solutions outside. For example, when dealing with web services, just using Java is fine. Of course, there is Scala language you can use!

If you can't emphasize with this, I'm sorry. Honestly, it's my story.

여러가지를 고민해본 결과, 프로그래밍 언어 분야에서 파서를 개발하는 것이 조금은 쓸만해보였습니다. 파서는 주로 프로그래밍 언어를 분석하는 것에서 많이 쓰이지만, 본 문서에서는 모호한 표현이 상대적으로 적은 JSON Parser를 소개합니다.

# JSON에 대한 소개

JSON은 Javascript Object Notation의 약자로, 많은 HTTP RESTful통신에서 쓰이고 있는 데이터 표기법입니다. 웹에서 파싱하기가 편하다는 점, 그리고 가독성이 괜찮은 점 떄문에 많이 쓰는 기법입니다. 크게, JSON Object라고 잡아봅시다. 이 Object는 크게 6가지 타입을 갖습니다.

- Null: null
- Boolean: true, false
- 문자열: "123", "한국"
- 숫자: 1, 1.1, -2, 1e+3
- 배열: [JSONObjects1, JSONObject 2, ...]
- 객체: {"key": JSONObject, "key2": JSONObject, ...}

이상의 값을 가지고 여러가지 표현이 가능한데, 예를 들어보면
```json
"문자열"
```

```json
null
```
위 값들은 모두 유효한 JSON값입니다.

```json
[1,2,3]
```
이 값 또한 유효하고,

```json
{
  "name": "Gyeongmin Go",
  "email": "gkm2164@gmail.com"
}
```

이 값 또한 유효합니다. 위 값들을 참조할 때는, JavaScript를 예로들면 아래와 같이 쓸 수 있습니다.

```json
var x = {
  "name": "Gyeongmin Go",
  "email": "gkm2164@gmail.com"
};

console.log("My name is [" + x.name + "].");
// My name is [Gyeongmin Go].
```

# Haskell로 타입 정의하기

이제 처리할 수 있는 값으로 만들기 위해, 하스켈로 표현해봅시다.
여기에 `newtype`을 쓸 수도 있겠지만, 일단 Data를 저장하는 용도가 더 크기 때문에 `data type` 키워드로 해보겠습니다.

```haskell
data JObject = JNumber Int
             | JString String
             | JBool Bool
             | JArray [JObject]
             | JMap [JsonAssoc]
             | JNull
  deriving (Show, Eq, Ord)

data JsonAssoc = (String, JObject)
```

바로 object로 들어가긴 조금 어렵기 때문에, 우선 null, true, false와 같은 키워드를 파싱하는 로직을 만들어봅시다. 우선, null 키워드를 처리하는 함수를 만들고 점차 refactoring 시켜보도록 하겠습니다.

1. 함수 형태 정의하기

```haskell
type JSONString = String

parseNull :: JSONString -> (Maybe JObject, JSONString)
```

파싱이 실패할 수도 있기 때문에 Maybe 모나드를 사용해봅시다. 만약 어디서 어떻게 오류가 났는지를 표현하고 싶다면, Either타입을 써볼 수 있으나, 우선은 쉽게...

2. 함수 정의하기
```haskell
parseNull :: JSONString -> (Maybe JObject, JSONString)
parseNull [] = (Nothing, [])
parseNull str | str `isPrefixOf` "null" = (Just $ JString "null", drop 4 str)
              | otherwise               = (Nothing, str)
```

위의 표현은 정말 단순하게 표현해본 방법입니다. 전반적으로 위의 패턴을 이용하여 false, true 등을 처리할 예정입니다.

3. 사용해보기
```haskell
parseNull "null   "
-- (Just (JString "null"), "   ")
```

그렇다면 true, false등도 아마 비슷한 방식으로 동작할 것입니다. 

# 간단한 Parser

우선 자료 구조를 정의해봅시다.

파서는 Monad로 표현할 수 있습니다. 여기에서는 State 모나드를 사용해봅시다.

```haskell

type State = String

data StateResult a 

newtype ST a = S (State -> (StateResult a, State))

```

집중해야할 타입이 ST인데, Monad하다는 것은 Functor, Applicative가 구현되어 있어야 한다는 것을 의미합니다.


```haskell
instance Functor ST where
  fmap f st = S (\s -> let (v, s') = app st s in (fmap f v, s'))

instance Applicative ST where
  pure x = S (Right x,)
  stF <*> stX = S (\s ->
    let (f, s') = app stF s
        (x, s'') = app stX s'
    in (f <*> x, s''))

instance Monad ST where
  stX >>= f = S (\s ->
    let (x, s') = app stX s in case x of
      Left e -> (Left e, s)
      Right v -> app (f v) s')
```

Json에 대해서 정의해봅시다.


크게 JsonObject는 숫자, 문자열, Boolean 
