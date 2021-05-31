---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "http4s 역사로 이해하기 - Scala 함수형 웹 프레임워크 2편"
subtitle: ""
summary: ""
authors: [Youngseo Choi]
tags: [Functional Programming, Scala, http4s, HTTP, Web, Framework]
categories: [Functional Programming]
date: 2021-05-31T23:48:57+09:00
lastmod: 2021-05-31T23:48:57+09:00
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

총 2편으로 구성되어 있습니다. [1편 바로가기](/post/understanding-http4s-scala-web-framework-part1)

---

## SemigroupK? 어디까지 가는거니..

http4s는 `Kleisli`처럼 또 쓸만한 녀석들이 있는지 찾아봤나봅니다. 먼저 [`Semigroup`](https://typelevel.org/cats/typeclasses/semigroup.html) 타입클래스는 결합 법칙이 성립하는 `combine` 연산자를 정의합니다. [`SemigroupK`](https://typelevel.org/cats/typeclasses/semigroupk.html)는 타입 파라미터를 1개 받는 타입 생성자(type constructor)에 대해 `combineK` 메소드 확장시켜줍니다.

공식문서만 읽어보면 어렵게 느껴지지만 (지금도) 간단히 같은 타입의 두 값을 하나로 합쳐준다고 생각하면 될 것 같습니다. 그리고 그 합쳐주는 연산에 대해 결합법칙이 성립해야 하는 조건이 추가적으로 달려있을 뿐입니다.

http4s에서는 이 타입클래스를 이용해 여러 라우터들을 결합하는데 활용하고 있습니다. 이전에 우리는 `combine(x: HttpRoutes, y: HttpRoutes): HttpRoutes` 함수를 이용해 두 개의 라우터를 결합하였습니다. 다음 코드를 보면 `Kleisli[F, A, _]` 타입 생성자와 `FutureOption` 타입생성자에 대해 `SemigroupK` 타입클래스 인스턴스를 생성하여 `en.combineK(es)` 와 같이 두 개의 라우터를 결합하고 있습니다. 이 방식의 기존 `combine`함수(`Semigroup`말고 우리가 이전에 만든) 보다 좋은 점은 다양한 효과 타입 `F`에 대해 일반화된 구현이라는 점입니다. (좀 더 뒤에서 장점이 드러나게 됩니다.)

{{< gist tomatophobia d4a379b0b666ece4007faeaea4b85112 >}}

## 로깅 추가

전체 요청 중에서 각 요청들이 차지하는 비율을 알고 싶어서 로그를 추가할 수 있습니다. 코드는 비교적 간단합니다. `Kleisli#mapF`를 활용하는데 간단히 보면 코드에서 `app` 함수를 실행한 후 그 결과를 `mapF`가 파라미터로 받는 함수가 파라미터로 받아 실행합니다. 내부에 있는 `*>`는 cats의 `Apply` 타입클래스의 `productR`의 별칭(alias)인데 첫 번째 인자의 결과를 무시하고 두 번째 인자의 결과를 리턴하는 단순한 함수입니다.

{{< gist tomatophobia 4a68b42dcb2cb04315bf53a2393fb9ca >}}

그러나 현재 로깅에는 약간 아쉬운 문제가 하나 있습니다. 위 코드에서 테스트 케이스를 보면 분명 uri로 `/hello`에 요청을 보냈음에도 불구하고 "TRANSLATING"이 출력되는 것을 볼 수 있습니다. `/hola` uri를 사용하지 않았음에도 "TRANSLATING"이 출력되는 이유는 `Future`이 "조급"(eagar)하기 때문입니다. [eager evaluation](https://ko.wikipedia.org/wiki/조급한_계산법), [lazy evaluation](https://ko.wikipedia.org/wiki/느긋한_계산법) 위키 글에 잘 설명되어 있습니다. 간단히 차이를 설명하면 함수에 인자를 적용하기 전에 인자의 계산이 선행되면 조급한 계산, 인자의 계산을 실제로 인자를 사용할 때까지 미루면 느긋한 계산이라고 합니다. 따라서 `Future`는 조급하기 때문에 `/hola` uri에 요청을 보내지 않았음에도 불구하고 먼저 연산이 일어나 "TRANSLATING" 메시지가 출력됩니다.

조급한 계산은 우리가 의도하지 않은 부수효과를(더 부정적으로 보면 부작용, side effect) 발생시킵니다. 위 예시에서는 부수효과가 단순히 콘솔에 메시지를 출력하는 것이었지만 예를 들어 데이터베이스의 데이터를 삭제하는 부수효과에 대해 조급한 계산이 일어나는 경우 의도하지 않은 시점에 연산이 실행된다면 매우 수습하기 어려운 상황이 발생할 수 있습니다.

## 조급한 Future, 느긋한 IO

`Future`는 비동기로 작동하는 편리한 데이터 타입이지만 결국 우리에게는 `Future`보다 조금 더 느긋한 녀석이 필요합니다. [cats effect](https://typelevel.org/cats-effect/docs/2.x/getting-started) 라이브러리에서 이런 일에 사용할 수 있는 타입클래스인 `Sync`를 제공합니다. `Sync`는 `Monad`를 약간 확장하여 부수효과가 발생하는 연산을 지연할 수 있는 `suspend` 함수를 지원합니다.

아쉽게도 우리는 `Future` 타입에 대해 `Sync` 타입클래스 인스턴스를 만들 수 없습니다. 왜냐하면 `Future` 타입의 값을 생성하는 순간 비동기적으로 연산이 시작되기 때문입니다. 따라서 이를 대체할 새로운 데이터 타입이 필요한데 이 역시 cats effect에서 제공하는 [`IO`](https://typelevel.org/cats-effect/docs/2.x/datatypes/io)를 사용합니다. `IO`는 `Future`와 유사하게 비동기 작업을 콜백함수를 사용하지 않고 처리할 수 있습니다. 그리고 거기에 더해 부수효과를 세상의 끝(end of the world)까지 지연시킬 수 있습니다.

> 세상의 끝? (End of the World)
모든 부수효과들이 발생하는 프로그램의 가장 바깥쪽 레이어를 의미합니다. 우리가 봤던 코드들 중에서는 유닛 테스트가 여기에 해당합니다. 또는 HTTP 서버 프로그램에서 요청-응답 사이클의 마지막 응답을 받는 부분이 세상의 끝에 해당할 것입니다. 많은 함수형 프로그래밍 언어들에서 사용되는 개념이며 프로그램의 코어 부분은 순수한 함수로 작성하고 발생하는 모든 부수효과(콘솔 입출력, 데이터베이스 쿼리, 외부 네트워크 통신)는 세상의 끝까지 지연시킵니다.

이제 우리가 사용하던 `Future`들을 모두 `IO`로 대체하였습니다.

{{< gist tomatophobia f3e1e5bbeb22043d1fc8fc72849b7ddd >}}

추가적으로 우리가 `FutureOption` 타입에 대해 `Monad` 타입클래스 인스턴스를 만들어 `flatMap`을 가능하게 했던 것처럼 `IOOption`에 대해서도 같은 작업을 해주어야 합니다. 또한 `SemigroupK`, `Sync` 타입클래스 인스턴스도 생성해주어야 합니다. 어려워보이지만 `IOOption`의 기능을 확장시켜주는 부분이라고 생각하면 좋을 것 같습니다.

{{< gist tomatophobia 3de524571145eea7365ff3cd671a7405 >}}

이제 다시 문제가 발생했던 `log`를 `IOOption`을 이용하는 라우터로 재작성해봅니다. 달라진 점은 `Monad#pure`를 사용하는 대신 `Sync#delay`를 사용했다는 점입니다. 따라서 실제로 app이 실행되지 않으면 "TRANSLATE" 메시지가 출력되지 않습니다.

{{< gist tomatophobia 86628ba3398f909e18202a794fe47706 >}}

`Future`보다 느긋한 `IO`를 사용하였기 때문에 `/hello` uri에 요청을 보냈을 때는 "TRANSLATE" 메시지가 출력되지 않는 것을 확인할 수 있습니다.

## Option이 계속 붙어있으니까 불편한데?... 그럼 OptionT!

이전 코드에서 등장한 `FutureOption`, `IOOption`에서 보면 알 수 있듯이 뭔가 반복된 패턴에 대해 추상화시킨 무언가가 있지 않을까 하는 생각이 들 수 있습니다. (저는 안들었습니다.) 이런 상황을 위해 cats에서 준비해둔 [`OptionT`](https://typelevel.org/cats/datatypes/optiont.html) 데이터 타입이 있습니다. cats 공식 문서에 따르면 `OptionT[F[_], A]`는  `F[Option[A]]`를 의미하는 간단한 래퍼(wrapper)라고 설명하고 있습니다. 좀 더 정확히 말하면 `Option`에 대한 [모나드 트랜스포머](https://sungjk.github.io/2019/01/27/monad-transformers.html)인데 만약 모나드 트랜스포머가 무엇인지 이해하기 너무 어렵다면 우선 넘어가도 좋습니다. cats에서도 `OptionT`의 유용함에 집중하면 된다고 말하고 있네요. 추가로 편리한 점은 위 코드에서 했던 것 처럼 타입클래스 인스턴스들을 만들어줄 필요 없이 cats가 준비해놓은 인스턴스를 사용할 수 있다는 것입니다.

다음은 `OptionT`를 이용하여 바뀐 코드입니다. 이제 `HttpApp`과 `HttpRoutes`는 `IO`에 국한되지 않고 여러 데이터 타입을 받을 수 있습니다. (다만 `Sync` 타입클래스 인스턴스를 가진 데이터 타입이어야겠죠?) 기존에 돌아가던 테스트들이 모두 정상적으로 작동하는 것을 볼 수 있습니다.

{{< gist tomatophobia 91be7e4cefde44b2abcd9e4ad9cd4f81 >}}

다시 한 번 강조하면 이제 `hello` 서비스는 더 이상 `IO`에 국한되어 있지 않습니다. 다른 대체제가 있다면 얼마든지 바꿀 수 있습니다. (`Future`는 안되겠지만요) 예시로 [monix](https://monix.io)가 제공하는 [`Task`](https://monix.io/docs/current/eval/task.html) 데이터 타입을 사용할 수 있습니다.

## 응답이 너무 길다?... 그럼 Stream!

만약 우리 서비스에서 책 한 권을 통째로 번역해야 한다면 어떻게 될까요? 다음 코드처럼 매우 긴 응답이 이어질 것입니다. 응답을 받는 입장에서 책 전체 번역이 끝나는 것을 언제까지 기다려야 하는지 알 수 없고, 만약 끝까지 기다려서 데이터를 받는다고 해도 메모리 버퍼를 초과해버릴지도 모른다는 문제가 있습니다.

{{< gist tomatophobia bcc0d6fc63ea80068c8f2debc06c5851 >}}

사실 HTTP에는 이미 이런 긴 응답을 처리하기 위해 데이터를 쪼개서 응답할 수 있는 [스펙](https://datatracker.ietf.org/doc/html/rfc2616#section-3.6.1)이 마련되어 있습니다. 간단히 설명하면 긴 데이터를 청크 단위로 쪼개서 응답을 받고 만약 빈 청크를 받으면 데이터가 끝났다는 것을 알 수 있습니다. 그러므로 http4s에서도 이 스펙을 활용하여 응답을 쪼개서 받는 기능을 지원해야 할 것입니다.

이를 위해 `Request`와 `Response`를 약간 바꿔봅니다. 요청과 응답의 body가 단순한 `String`에서 `fs2.Stream[F, Byte]`로 변경되었습니다. [fs2](https://fs2.io/#/getstarted/install)에 대해서는 이 글에서는 자세히 설명하지 않겠습니다. 다만 Stream에 대해 간단히 설명하면 출력 타입(코드에서는 `Byte`)의 데이터를 `F` 컨텍스트 하에서 읽을 수 있습니다. `F`가 필요한 이유는 예를 들어 파일을 비동기로 읽는 함수가 있다고 할 때 그 함수의 결과는 `Future[Byte]`가 될 것입니다. (`Future`말고 `IO`를 써도 됩니다.) 이 때 `Future`가 `F`가 된다고 생각하시면 될 것 같습니다.

{{< gist tomatophobia 65b3a789cd1b23e4c22dcaf7dc5a0063 >}}

## 진짜 최종 HTTP 어플리케이션

그럼 이제 Stream까지 도입한 HTTP 어플리케이션 코드를 보겠습니다. (자세히 보시면 글의 시작에서 언급한 http4s 코드와 같아진 것을 보실 수 있습니다.) HTTP 함수는 두 개의 컨텍스트(`F`, `G`) 하에서 작동합니다. `F`는 응답이 발생시키는 효과, `G`는 스트림에서 발생되는 효과입니다. (영상의 글을 그대로 인용하면 "HTTP is just a Kleisli function from a streaming request to a polymorphic effect of a streaming response." 입니다.)

{{< gist tomatophobia 3711e5140c6f8cf245202213b0517952 >}}

`app` 함수 내부에서 번역이 일어나는 로직을 잠시 살펴보면, 먼저 `Request`의 body(`Stream[IO, Byte]` 타입)를 받아서 디코딩을 한 후 (`Stream[IO, String`으로 변환) 번역 서비스에게 번역을 요청합니다.  그 후 결과를(`Stream[IO, String]` 타입) 인코딩 한 후 (다시 `Stream[IO, Byte]`로 변환) `Response`의 body에 넣어주고 있습니다.

유닛 테스트를 보면 어플리케이션의 결과가 `Stream`으로 도착하고 있기 때문에 전체 결과가 전송되는 것을 기다릴 필요도 없고, 응답 받는 쪽에서 메모리 버퍼 관리도 용이하게 할 수 있습니다.

## 마치며...

http4s의 발전 과정을 짚어보니 "매우" 어렵게 느껴졌던 http4s에 대한 이해도가 많이 올라간 것 같은 기분이 듭니다. 간단한 함수에서 시작하여 `Kleisli`, `SemigroupK`, `Stream`, ... 어려워했던 다양한 개념들이 하나씩 추가되는 것을 차근차근 따져보니 이해하는데 많은 도움이 되었습니다.

사실 타입 클래스, 모나드 등의 개념에 익숙하지 않은 분들이 보기엔 여전히 어려운 글이라는 생각이 듭니다. (물론 관련 주제들도 하나씩 글로 써보고 싶습니다.) 그래도 함수형 프로그래밍을 시작한지 조금 된 초보에서 중수로 넘어가는 분들에게 이 글이 조금이나마 도움이 되었으면 좋겠다고 생각합니다.

글에 틀린 정보가 있거나 이해가 안되는 점이 있으시다면 댓글이나 이메일 주시면 정말 감사드리겠습니다 :)
