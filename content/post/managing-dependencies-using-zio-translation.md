---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "ZIO를 이용해 의존성 관리하기 [번역]"
subtitle: ""
summary: ""
authors: []
tags: [ZIO, Scala, Functional Programming]
categories: [Functional Programming]
date: 2020-08-27T22:07:42+09:00
lastmod: 2020-08-27T22:07:42+09:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

이 글은 [Adam Warski](https://blog.softwaremill.com/@adamwarski)의 [Managing dependencies usin ZIO](https://blog.softwaremill.com/managing-dependencies-using-zio-8acc1539e276)를 번역한 글입니다.

---

[ZIO](https://zio.dev/)는 안전한 타입과 함께 조합 가능하며서 (composable) 비동기 (asynchronous) 또는 동시성 (concurrent) 프로그래밍을 할 수 있는 Scala의 라이브러리이다. 최근 자신들의 ["환경" 컴포넌트를 점검](https://zio.dev/docs/howto/howto_use_layers) 하였다. 여기서 그들은 프로그램을 실행시키기 위한 의존성을 관리 할 때 사용할 수 있는 새로운 접근법을 제시했다.

그런다면 ZIO의 "환경"이란 무엇을 말하는 걸까? ZIO에서는 비동기/동시성 프로세스를 **값**으로 표현하는데 이 때 그 **값**의 타입은 `ZIO[R, E, A]`이다. `E`는 실패할 수도 있는 연산에서 발생할 수 있는 에러를 나타내고, `A`는 연산의 결과 나오는 타입이다. 그리고 `R`은 "환경"(또는 필요조건) 나타내는데 이는 연산에 필요한 것들을 의미한다.

간단하게 `ZIO[R, E, A]`를 `R` 타입을 인자로 받고 연산이 실패할 경우 에러 타입 `E`, 성공할 경우 `A` 타입을 리턴하는  함수 `R => E | A` 로 **표현**할 수도 있다.  당연하지만 ZIO를 이용해 표현한 프로세스가 "보통의" 스칼라 함수보다 더 강력하다. ZIO에서 제공하는 몇 가지 컴비네이터로 비동기, 동시성, 에러 복구/시그널, 여러 환경 필요조건 등 다양한 상황의 연산을 안전하게 구현할 수 있다.

(역: 컴비네이터([combinator](https://stackoverflow.com/questions/58681172/what-is-a-combinator-in-scala-or-functional-programming))는 어떤 개체들을 결합하여 새로운 개체를 생성하는데 개체의 타입은 유지되는 연산자라고 이해하면 될 것 같습니다.)

어떤 어플리케이션을 개발하든 **의존성 관리**는 흔하게 문제를 발생시키고 관련된 논란도 많다. 그렇다면 ZIO에서는 어떻게 의존성을 관리할까? **의존성 주입 (Dependency Injection)**을 사용할 것인가? 만약 사용한다면 다른 의존성 주입 라이브러리들과 다른 점은 무엇일까?

우리는 이 글을 통해 위의 질문들을 하나씩 처리해볼 것이다.

> 이미 [다른 글](https://blog.softwaremill.com/zio-environment-meets-constructor-based-dependency-injection-6a13de6e000)을 통해 ZIO environment와 다른 의존성 주입 접근들을 비교한 바가 있다. 하지만 이는 이전 세대의 ZIO env까지에 적용되고 최근 구현체에서는 상당히 달라진 (그리고 발전된) 형태를 띄고 있기 때문에 새롭게 이 글을 쓰게 되었다.

## But What is DI?

의존성 주입에 대해 들어본 사람이라면 각자 자신만의 정의를 가지고 있을 것이다. 따라서 ZIO에 대해 이야기 하기 전에 내가 의존성 주입이란 말을 어떻게 생각하는지 말하는 것이 도움이 될 것 같다.

이전 글 ["What is Dependency Injection?"](https://blog.softwaremill.com/what-is-dependency-injection-8c9e7805502f) :

> 의존성 주입은 정적이고, 상태가 없는 서비스 오브젝트 그래프를 생성하는 프로세스이다. 각 서비스는 의존성에 의해 매개변수화(parameterised)된다. 
>
> (역: parameterised의 의미를 정확히 파악하지 못해 매개변수화라는 말로 표현하였습니다.)

반면 ZIO environment는 다른 목적으로도 쓰일 수 있다. 여기서 우리는 ZIO에서 어떻게 정적 오브젝트 그래프를 생성하는지에 집중할 것이다. 정적 오브젝트 그래프는 대부분 어플리케이션에 유용한 기능을 부여하기 위해 상태가 없는 서비스 형태로 쓰인다. 

## Basic building blocks

ZIO 기반의 어플리케이션에서 `ZIO[R, E, A]`가 기본 블록이 (building block) 되는 것처럼 의존성을 관리할 때 중요한 기본 블록 컴포넌트는 `ZLayer`이다.

`ZLayer[RInt, E, ROut]`은 `ROut`을 생성하기 위해 `RIn` 에 의존하고 있음을 나타낸다. 그리고 이러한 의존 관계 생성 과정이 실패할 경우 발생하는 에러의 타입은 `E`이다. 

위의 레이어를 이용해 단일 의존관계를 표현할 수 있다. 이는 생성자를 이용한 의존성 주입보다 더 강력하다. 다음은 일반적인 생성자를 이용한 의존성 주입 예시이다.

```scala
class ServiceA(b: ServiceB, c: ServiceC)
```

이 코드를 함수로도 표현할 수 있다.

```scala
(ServiceB, ServiceC) => ServiceA
```

ZIO env 에서는 위 상황을 타입과 함께 "값"으로 나타낼 수 있다.

```scala
ZLayer[Has[ServiceB] with Has[ServiceC], Nothing, Has[ServiceA]]
```

의존 관계 생성을 다양한 방법으로 표현할 수 있다. 위의 예시에서는 생성자를 이용한 방법을 보았다. 예를 들어 의존 관계 생성 과정에 효과 로직 (effectual logic)이 있는 경우가 있을 수 있다. 또는 생성 과정에서 얻은 리소스를 다 사용하고 난 후 방출(release)시키는 과정을 포함해야 할 수도 있다. 이런 경우 안전하게 서비스를 생성하는 것뿐만 아니라 더 이상 필요하지 않은 경우 닫아주는 과정 또한 필요하다. `ZLayer`가 가진 몇 가지 컴비네이터를 이용해 이런 다양한 사용 예를 해결할 수 있다.

> `Has[_]` 타입의 의미가 궁금한 독자들이 있을 것이다. `Has`는 ZIO에서 다중 의존성을 `with`을 이용해 표현하기 위해 만들어진 도구이다. 이 도구를 이용해 여러 의존성끼리 합치거나 제거하여 네임스페이스 충돌을 방지할 수 있다. 대부분의 `Has[_]` 는 기술적인 디테일로 다뤄지고 구현 디테일은 다루지 않는다.

레이어는 다중 의존 관계 생성도 표현할 수 있다. 한 레이어의 결과가 다른 레이어의 입력으로 사용되는 방식으로 레이어들을 결합하여 더 큰 크기의 레이어를 생성할 수 있다. 이는 마치 함수 합성과 비슷하다. 물론 이 과정을 위한 몇 가지 컴비네이트를 다양한 상황에 적용시킬 수 있다.

`ZLayer`는 어플리케이션의 서비스 그래프 단편을 생성하는 방법을 표현한다. 이는 의존성 주입의 정의와 잘 맞아떨어진다. 

당신이 방금 읽은 내용이 맞다. ZIO environment는 의존성 주입을 하나의 어플리케이션에 대응되도록 구현한다. 그와 동시에 타입 안전, 리소스 안전, 잠재적으로 동시적 실행, 에러 처리를 리플렉션([reflection](https://stackoverflow.com/questions/58681172/what-is-a-combinator-in-scala-or-functional-programming))이나 클래스패스 스캐닝 (classpath scanning)없이 할 수 있다.

## Recipe for single ZIO dependency

