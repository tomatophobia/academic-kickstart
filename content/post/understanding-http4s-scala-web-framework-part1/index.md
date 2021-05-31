---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "http4s 역사로 이해하기 - Scala 함수형 웹 프레임워크 1편"
subtitle: ""
summary: ""
authors: [Youngseo Choi]
tags: [Functional Programming, Scala, http4s, HTTP, Web, Framework]
categories: [Functional Programming]
date: 2021-05-27T00:17:49+09:00
lastmod: 2021-05-27T00:17:49+09:00
featured: false
draft: false
toc: true

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
이 글은 [http4s: pure, typeful, functional HTTP in Scala – Ross Baker](https://www.youtube.com/watch?v=urdtmx4h5LE) 영상을 번역 + 부가 설명 + 약간의 사견을 넣어 작성한 글입니다.

총 2편으로 구성되어 있습니다. [2편 바로가기](/post/understanding-http4s-scala-web-framework-part2)

---

## 글을 쓰게된 경위 (넘겨도 무방)

과거 스칼라를 이용해 함수형 프로그래밍을 시작하고 제가 배운 내용을 어떻게 실제로 활용하고 싶어 스칼라로 만들어진 웹 프레임워크들을(또는 라이브러리, 이하 프레임워크라고 하겠습니다.) 찾아보기 시작하였습니다. Play, Akka HTTP, Scalatra, http4s 등 다양한 프레임워크들이 있었습니다. 그 중에서도 순수 함수형 프로그래밍 기반으로 만들어진 http4s가 눈길을 끌었습니다. 하지만 [http4s quickstart 템플릿 코드](https://github.com/http4s/http4s.g8)를 본 순간...

{{< gist tomatophobia c1ca51321ff177ad0a8c0ec1c4c5a845 >}}

{{< gist tomatophobia 408ddb1fb97e9dfa2d5daef40594bca1 >}}

당시 스칼라로 함수형 프로그래밍을 공부하기 시작한지 얼마 되지 않았던 저로서는 저 코드들이 너무 어렵게 느껴졌습니다. 글을 쓰면서 다시보니 그 때만큼 막막해보이진 않네요. 신기한 일입니다.

어쨌든 좀 더 자세한 내용을 파악하기 위해 `HttpRoutes`가 어떻게 동작하는지 파고들어가 보았습니다.

{{< gist tomatophobia fc0eb6ac6b6fd6c68eb5274929a28822 >}}

결국 http4s가 제시하는 핵심은 `type Http[F[_], G[_]] = Kleisli[F, Request[G], Response[G]]` 라는 것을 짐작할 수 있었습니다. 그 후 더 나아가기 위해 [Cats: Kleisli](https://typelevel.org/cats/datatypes/kleisli.html) 설명을 읽었지만 제 실력에 이해하기는 어려웠고, 좌절하면서 http4s 공부를 접었던 슬픈 사연이 있습니다...

결국 시간이 지나 http4s를 실무에서 사용하게 되었지만 여전히 저 의미를 완벽히 파악하고 있지는 않았습니다. 그래도 현재는 함수형 프로그래밍에 대한 기반 지식이 "약간" 쌓였고 결정적으로 [글머리에서 언급한 영상](https://www.youtube.com/watch?v=urdtmx4h5LE)에서 http4s가 어떻게 발전해왔는지 잘 설명되어있어서 그 개념을 단단히 다지고 싶어 이 글을 시작하게 되었습니다.

영상에서는 HTTP 어플리케이션을 간단한 정의부터 시작하여 요구사항에 맞춰 조금씩 발전시켜나가는 과정을 설명하고 있습니다. 그래서 저도 영상에 맞춰 목차를 구성하였습니다.

## HTTP 어플리케이션은 요청을 받고 응답을 반환하는 함수다

{{< gist tomatophobia e07003a0416b8d51bf32ee306c711a81 >}}

HTTP 어플리케이션을 함수로 표현함으로써 패턴 매칭을 통해 쉽게 어플리케이션을 작성할 수 있습니다. 위 코드에서는 간단히 `POST /hello` 요청에 대해 적절한 응답을 보내는 어플리케이션입니다.

약간이지만 여기서 함수형 프로그래밍의 장점을 느낄 수 있습니다. 라우팅이 올바르게 동작하는지 테스트하기 위해 다른 컴포넌트와 함께 통합 테스트를 할 필요도 없고 복잡한 요청, 응답 api를 모킹할 필요도 없습니다. 간단한 유닛테스트를 통해 어떤 요청이 들어가고 어떤 응답이 나오는지 간단한 예시를 만들고 함수를 작동시키기만 하면 끝입니다. 이게 가능한 이유는 결국 우리가 만든 HTTP 어플리케이션이 순수한 함수이기 때문입니다.

HTTP 어플리케이션 중에는 응답을 구성하기 위해 비동기로 작업을 해야하는 경우가 발생합니다. (데이터베이스에서 데이터를 가져온다거나 다른 서버에 데이터를 요청한다거나 등) 다음 코드는 `POST /translate` 요청을 보낼 경우 번역을 하는데 이 때 사용되는 `Translater`는 `Future` 타입의 값을 반환합니다.

{{< gist tomatophobia 456d5b5015dd77870e22acc6e8d9117d >}}

위 코드는 문제없이 잘 작동합니다. 하지만 `Await`를 사용하면 쓰레드가 블럭되기 때문에 이는 좋은 사용자 경험을 제공하기 어렵다는 단점이 있습니다.

## HTTP 어플리케이션은 요청을 받고 Future[응답]을 반환하는 함수다

위에서 언급한 `Await` 사용 시 문제를 해결하기 위해 HTTP 어플리케이션을 확장하였습니다.

{{< gist tomatophobia 981c3d434f9e793b57465966644dbdca >}}

이제 Await를 사용할 필요가 없습니다. 대신 유닛 테스트를 할 때 `asyncAssert`를 사용해야 합니다. 물론 `scalatest`가 이를 지원해주기 때문에 `asyncAssert`에 대해서는 걱정할 필요가 없습니다.

HTTP 어플리케이션을 작성하다보면 다른 어플리케이션과의 통합(Combination)이 유용한 경우가 있습니다. Node JS에서 라우터 코드를 URL 경로 별로 나눠서 작성한 뒤 app.js 파일에서 통합하는 것을 생각하면 쉽습니다. 다음 코드의 `combine` 함수는 이러한 요구사항을 반영하여 만들어진 함수입니다. 두 개의 `HttpApp`을 받아서 하나로 통합합니다.

{{< gist tomatophobia 2a05c942dd7508f565aa607d38764d1c >}}

위 방식의 통합은 프로그램 실행 중 발생하는 에러를 캐치 후 처리하는 방식을 채택하고 있습니다. 이 때 `HttpApp` 타입만 보고서는 에러가 발생할 수 있다는 것을 전혀 암시하고 있지 않기 때문에 프로그래머가 잘못된 가정을 할 수 있다는 문제가 있습니다. 좀 더 설명을 덧붙이면 현재 `type HttpApp = Request => Future[Response]` 인데 이 타입만 보고서는 에러 발생 가능성을 쉽게 알 수 없습니다.

## HTTP 라우터는 요청을 받고 Future[응답]을 반환하는 부분함수다

다행히 스칼라에는 이런 상황에 유용하게 적용할 수 있는 부분 함수(partial function) 개념이 있습니다. 부분 함수는 인풋 타입에 속하는 원소들 중 일부에 대해서만 정의된 함수입니다. (좀 더 수학적으로 표현하면 정의역의 일부에 대해서만 정의된 함수입니다.) 즉 현재 우리 상황에서는 우리가 핸들링 코드를 작성한 일부 `Request`에 대해서만 `Response`가 있다고 이해할 수 있습니다.

문제는 HTTP 어플리케이션은 모든 요청에 대해서 상응하는 응답이 있어야 한다는 점입니다. 따라서 이 시점에서 HTTP 라우터(Routes) 개념을 도입합니다.

{{< gist tomatophobia 93eaa6530a789c59386d5f70cc21677a >}}

이제 프로그램 실행 도중 에러를 발생시키고 핸들링 할 필요가 없어졌습니다. 스칼라 `PartialFunction`이 지원하는 `orElse`를 사용하여 두 `HttpRoutes`를 쉽게 통합할 수 있습니다.

그러나... 여전히 HTTP 라우터에서 처리하지 않는 요청에 대해서 HTTP Application은 에러를 발생시킵니다.

{{< gist tomatophobia 417351c73fea117443ba9b16324de6c6 >}}

## HTTP 라우터는 요청을 받고 Option[Future[응답]]을 반환하는 함수다.

부분함수의 개념을 도입했지만 여전히 HTTP 어플리케이션은 부분함수인 HTTP 라우터를 그대로 사용하고 있습니다. 따라서 어디에선가 명시적으로 모든 요청에 대해 응답을 반환하는 함수로 변경하는 작업이 필요합니다.

{{< gist tomatophobia 9208985d9ae6b9e37e6119343ac879de >}}

`PartialFunction#lift` 함수를 통해 정의되어 있지 않은 입력을 받는 경우 `Option#None`을 반환하는 함수로 변경합니다.

추가적으로 만약 HTTP 라우터 처리의 결과가 `Option#None`인 경우 404 응답을 보내도록 처리하는 `seal` 함수를 이용해 HTTP 어플리케이션의 타입을 맞춰줍니다.

{{< gist tomatophobia 299314d723a5b2d9342e479d6f407ac2 >}}


HTTP 어플리케이션 실행 도중 특별한 작업을 실행 과정 중간에 삽입하거나 실행 결과를 변형하고 싶은 경우가 있을 것입니다. 간단하게 [데코레이터 패턴](https://en.wikipedia.org/wiki/Decorator_pattern)이나 Node JS 등의 [미들웨어](https://medium.com/@selvaganesh93/how-node-js-middleware-works-d8e02a936113)와 대응된다고 볼 수 있습니다. 이러한 방식의 서비스 연결을 서비스 결합(Composition)이라고 칭합니다.

예를 들어 HTTP 어플리케이션 실행 결과를 번역을 해주는 미들웨어를 추가한다고 해보겠습니다.

{{< gist tomatophobia 260ae0738266b12c2497ff4eb47935e4 >}}

위와 같이 `translate: HttpApp => HttpApp` 함수를 추가하여 응답결과를 번역하는 작업을 추가할 수 있습니다.

> **통합(Combination)과 결합(Composition)** \
> 통합은 함수를 횡으로 묶는 것이고 결합은 함수를 종으로 연결하는 개념입니다. \
> 예시) \
> 통합: 두 부분 함수를 통합하여 더 많은 입력을 처리할 수 있는 부분 함수 생성
> 결합: 첫 번째 함수의 결과를 두 번째 함수로 넘기는 방식으로 두 함수를 결합

그러나 `translate`를 전체 HTTP 어플리케이션에 적용하는 것은 가능하지만 어플리케이션을 구성하는 일부 라우터에 적용하면 에러가 발생합니다.

{{< gist tomatophobia e0ff94a0ff018675d8c8ec253f21b739 >}}

이 시점에서 http4s에서 처음으로 [cats](https://typelevel.org/cats/)를 도입합니다. 다음은 cats가 제공하는 데이터 타입들 중 [`Kleisli`](https://typelevel.org/cats/datatypes/kleisli.html)

코드입니다.

{{< gist tomatophobia b5959822da5f65eec1e4359e8c8eda4d >}}

이름은 알아듣기 어렵지만 `Kleisli[F[_], A, b]`는 단순히 `A => F[B]` 타입의 함수를 감싸고 있는 데이터 타입일 뿐입니다. 다만 `Kleisli`로 감싸줌으로써 모나딕한 값을 반환하는 다른 함수와 결합할 수 있습니다. 좀 더 쉽게 말하면 `A => F[B]` 타입의 함수와 `B => F[C]` 타입의 함수 간의 결합을 가능하게 합니다. (원래 함수끼리 결합하기 위해서는 반환 타입과 다음 함수의 입력 타입이 서로 같아야 합니다.)

## HTTP 어플리케이션은 요청을 받고 응답의 다형성 효과를 반환하는 Kleisli 함수다

갑작스럽게 제목이 조금 어려워졌습니다. 괜찮습니다. 내용은 더 어렵습니다. ㅎ

다형성 효과(polymorphic effect)에 대해 잠깐 짚고 넘어가면, 다음 코드에서 `F[_]`가 그 역할을 합니다. 간단히 응답을 감싸는 컨테이너라고 생각하면 좋을 것 같습니다. 함수형 효과(functional effect)에 관해서 [제 글](https://easywritten.org/post/an-introduction-to-functional-design-translation/)을 참고하시는 것도 약간이나마 도움이 되실 수 있을 것 같습니다.

먼저 HTTP는 요청을 받고 어떤 효과가 감싼 응답을 반환하는 함수이고, HTTP 어플리케이션은 `Future` 효과 내부에서 발생하는 HTTP 입니다. 그리고 HTTP 라우터는 `Option[Future[_]]` 내부에서 발생하는 HTTP로 볼 수 있습니다. 코드의 난이도가 갑자기 상승하였지만 `Kleisli[F[_], A, B]`를 `A => F[B]`라고 생각하고 천천히 하나씩 뜯어보면 아주 어렵지는 않습니다.

{{< gist tomatophobia c9fd4e489eeeaeadf1f01f2aed18382d >}}

이제 우리는 다시 `translate` 함수를 정의할 수 있습니다. 먼저 `translate` 함수는 `F`를 타입 파라미터로 받는데 이 때 `F`는 `Monad` 타입클래스의 인스턴스여야 합니다. (타입클래스에 대한 설명은 우선 생략하겠습니다. 개인적으로 스칼라로 타입클래스에 대해 설명한 [이 글](http://jaynewho.com/post/3)이 많은 도움이 되었습니다.)

`F`가 `Monad` 타입클래스의 인스턴스이므로 `flatMap`을 사용할 수 있습니다. 그리고 이 경우 자동으로 `Kleisli`타입은 `Monad` 타입클래스의 인스턴스를 가지게 되어 `Kleisli`에 대해서도 `flatMap`을 사용할 수 있게됩니다. (`Kleisli` 공식 문서의 [Type class instances](https://typelevel.org/cats/datatypes/kleisli.html#type-class-instances) 부분을 참고하시면 좋을 것 같습니다.) 따라서 scala가 제공하는 for comprehension을 사용하여 코드를 좀 더 깔끔하게 만들 수 있습니다.

추가적인 설명으로 `liftF`는 `F[B]`타입의 값을 받아서 `Kleisli[F[_], A, B]` 타입으로 승격시켜주는 함수입니다.

{{< gist tomatophobia a621b9dc607a32a6037898351e96c685 >}}

여기까지 `translate` 함수를 적용시키기 위해 험난한 길을 달려왔음에도 불구하고 컴파일 에러가 발생합니다. 에러 메시지를 간단히 해석해보면 implicit parameter로 넣어주어야 할 `Monad[OptionFuture]` 타입의 evidence paramter가 존재하지 않는다고 합니다. 어려워보이지만 이는 `Monad` 타입클래스의 인스턴스가 사용할 수 있는 함수인 `flatMap`에 들어가는 implicit parameter를 찾을 수 없어서 발생하는 에러입니다. 그리고 그러한 파라미터를 찾을 수 없는 이유는 결국 `Option[Future[_]]`가 `Monad` 타입클래스의 인스턴스가 아니기 때문입니다. (간단히 말해 모나드가 아니라고 할 수 있습니다.)

좀 더 정성적으로 `Option[Future[_]]`가 `flatMap`을 할 수 없는 이유를 생각해보겠습니다. 먼저 `Option`이 바깥쪽에서 둘러싸고 있기 때문에 결과가 `Some`인지 `None`인지 알기 위해서는 먼저 `Future`의 결과를 알 수 있어야 합니다. 먼저 `Future`의 결과를 알려면 쓰레드를 블러킹하거나 마법의 수정 구슬로 미래를 예측할 수 밖에 없습니다. 전자는 매우 나쁜 사용성을 가져올 것이고 (`Future`를 사용하는 의미가 퇴색됨) 후자는 당연히 불가능합니다. 따라서 `Option[Future[_]]`는 `flatMap`을 사용할 수 없습니다.

해결 방법은 의외로 간단합니다. `Option[Future[_]]` 대신 이를 뒤집어 `Future[Option[_]]`을 사용하는 것입니다. 아래 코드에서 `sequence` 함수를 통해 `Option[Future[_]]`에서 `Future[Option[_]]`으로 뒤집히게 됩니다. (`sequence` 함수에 대해서는 [해당 링크](https://www.slideshare.net/pjschwarz/sequence-and-traverse-part-1)에서 슬라이드 자료로 간단히 설명되고 있습니다.) 또한 `FutureOption` 타입에 대한 `Monad` 타입클래스 인스턴스를 생성하였습니다. 이제 우리는 `FutureOption`에 대해 `flatMap`을 사용할 수 있습니다.

{{< gist tomatophobia b0c614bd31acedee4175f46cc55d4bb4 >}}

이제 더 이상 `translate` 함수에서 에러가 발생하지 않습니다. 또한 `seal` 함수와 순서를 바꿔도 문제 없습니다. 즉 `HttpApp`, `HttpRoutes` 모두에 `translate` 함수를 적용시킬 수 있는 것을 볼 수 있습니다.

{{< gist tomatophobia dc9ec68515d0f9c86a2f1413fcebd04d >}}

---

여기까지 1편이었습니다. 내용이 많이 어렵지만 차근차근 습득하면 할 수 있을 것이라 믿습니다. (이건 저에게 말하는 것이기도 합니다만... ㅎ)

글에 틀린 정보가 있거나 이해가 안되는 점이 있으시다면 댓글이나 이메일 주시면 정말 감사드리겠습니다 :)

[2편 바로가기](/post/understanding-http4s-scala-web-framework-part2)
