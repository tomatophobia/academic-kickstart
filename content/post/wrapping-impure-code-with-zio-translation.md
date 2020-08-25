---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Wrapping impure code with ZIO [번역]"
subtitle: ""
summary: ""
authors: []
tags: [Functional Programming, Scala, ZIO]
categories: [Functional Programming]
date: 2020-08-23T23:37:39+09:00
lastmod: 2020-08-23T23:37:39+09:00
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

이 글은 [Pierre Ricadat](https://medium.com/@ghostdogpr) 의 [Wrapping impure code with ZIO](https://medium.com/@ghostdogpr/wrapping-impure-code-with-zio-9265c219e2e) 를 번역한 글입니다. 

---

만약 당신이 스칼라로 함수형 프로그래밍을 하고 있다면 상당히 헤맬 가능성이 높다. 우리가 다루는 코드가 함수형 프로그래밍의 기본 원리인 총체성 (totality), 결정성 (determinism), 순수성 (Purity)을 지키지 않을 수 있기 때문이다. 쉽진 않겠지만 다음을 상기하면서 이 문제를 해결해보자.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">FP is just programming with functions. Functions are:<br><br>1. Total: They return an output for every input.<br>2. Deterministic: They return the same output for the same input.<br>3. Pure: Their only effect is computing the output.<br><br>The rest is just composition you can learn over time.</p>&mdash; John A De Goes (@jdegoes) <a href="https://twitter.com/jdegoes/status/936301872066977792?ref_src=twsrc%5Etfw">November 30, 2017</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

> 함수형 프로그래밍은 함수로 프로그래밍을 하는 것일 뿐이다. 여기서 함수란:
>
> 1. 총체성: 모든 입력에 대해 출력을 반환한다.
> 2. 결정성: 항상 같은 입력에 대해 같은 출력을 반환한다.
> 3. 순수성: 출력을 계산하는 것 외에 효과가 발생하지 않는다.
>
> 이 외에 나머지는 앞으로 너가 배워나갈 것들의 조합일 뿐이다.

JVM 에코시스템은 매우 거대해서 프로그래머가 원하는 거의 모든 라이브러리를 찾을 수 있다. 하지만 그 중 대부분은 심지어 스칼라를 포함해서도 위에서 언급한 원칙을 지키지 않는다. 당신 또한 함수형이 아닌 레거시 코드를 가지고 있을 수 있다. 늦든 빠르든 이런 비함수형 코드들을 처리해야 할 때가 올 것이다. 이 글을 통해 나는 다양한 순수하지 않은 코드들을 어떻게 ZIO 라이브러리를 이용해 처리하는지 이야기할 것이다. 

앞으로 나올 예시들은 다양한 스칼라 또는 자바 코드에 종속성을 가진 실제 라이브러리 코드들을 (`zio-sqs`, `zio-akka-cluster`) 통해 설명한다. 따라서 이 글에서는 당신이 ZIO에 대해 배경지식이 있다고 가정하고 설명한다. 만약 ZIO를 처음 접한다면 [공식 문서](https://zio.dev/) 또는 [리소스](https://zio.dev/docs/resources/resources)들이 도움이 될 것 이다. 

{{< figure src="https://miro.medium.com/max/600/1*lM5WtEonS2vERGNJUpYlqg.jpeg" caption="Scala Developer discovering ZIO" >}}

## The One that throws

먼저 쉬운 예시부터 시작해보자. 우리가 에러가 발생할 수 있는 스칼라 또는 자바 레거시 코드를 호출한다고 하자. 어쩌면 주석을 통해 코드를 파악할 수도 있지만 운이 나쁘다면 이 코드가 에러를 발생시킬지 아닐지조차 알 수 없을 것이다. 이 때, 예외를 발생시키는 것은 총체성에 위배되는 행위이다. 왜냐하면 함수가 모든 가능한 입력에 대해 항상 반환 값이 존재하지 않기 때문이다.

만약 우리가 ZIO의 `IO.succeed` 또는 `map/flatMap` 을 사용하는 경우 에러가 발생하면 [파이버(fiber)](https://en.wikipedia.org/wiki/Fiber_(computer_science))가 중단되기 때문에 에러를 발생시켜서는 안된다. 대신 우리는 `IO.effect` 를 사용해야 한다. `IO.effect` 는 효과(effect) 생성자로써 예외를 잡아서 `IO[Throwable, A]`(aka `Task`) 타입으로 반환한다. 이 방법을 이용하면 `mapError`, `catchAll` 등을 사용하여 에러를 처리할 수 있다.

Note: `Task.apply` 를 사용하여 똑같은 처리를 할 수 있다.

```scala
// Wrapping Java NIO file copy
import java.nio.file.{ Files, Paths }
import zio.IO

def copyFile(path: String, destination: String): IO[Throwable, Unit] = IO.effect {
  Files.copy(Paths.get(path), Paths.get(destination))
}
```

위 방법은 레거시 코드가 에러를 발생시키는지 애매할 때 사용하기 좋은 방법이다. (물론 확실히 에러를 발생시킬 때도 좋다.) 이 경우 부분 코드를 효과(effect)로 바꾸면서 명시적으로 에러가 발생할 수 있음을 나타낸다.

## The One that blocks

이번에는 레거시 코드가 에러를 발생시킬 뿐만 아니라 실행이 완료될 때까지 현재 쓰레드를 블락하는 경우를 생각해보자. 만약 프로그래머가 위의 상황에서 일반적인 `IO` 를 사용한다면 어플리케이션의 메인 쓰레드 풀에 속하는 쓰레드를 블락할 것이고 이는 쓰레드 [기아 상태](https://en.wikipedia.org/wiki/Starvation_(computer_science))를 초래할 것이다. 이런 경우에는 어플리케이션의 메인 쓰레드 풀에서 실행하는 것 보다 블락을 필요로 하는 태스크들만을 위한(dedicated) 쓰레드 풀 내부에서 실행하는 것이 더 좋다.

물론 ZIO에는 이를 위한 해답이 있다. 바로 `effectBlocking`으로 코드를 감싸는 것이다. `effectBlocking`은 해당 코드만을 위한 (dedicated) 쓰레드 풀에서 코드를 실행시키는 것을 보장한다. 반환하는 타입은 ZIO[Blocking, Throwable, A] 이다. 우리는 반환 타입을 통해 "블라킹 환경(= 사용할 쓰레드 풀)"과 에러가 발생할 수 있음을 알 수 있다. 블라킹 환경을 만드는 방법을 몰라도 걱정할 필요없다. 실행할 메인 함수에서  [`zio.App`](https://zio.dev/docs/overview/overview_running_effects#app) 을 사용하면 알맞은 기본 환경이 제공된다.

```scala
import scala.io.{ Codec, Source }
import zio.{ App, console, ZIO }
import zio.blocking._

object SampleApp extends App {

  def getResource(path: String): ZIO[Blocking, Throwable, String] = effectBlocking {
    Source.fromResource(path)(Codec.UTF8).getLines.mkString
  }

  override def run(args: List[String]): ZIO[Environment, Nothing, Int] =
    getResource("test.txt").foldM(ex => console.putStrLn(ex.toString).const(-1), res => console.putStrLn(res).const(0))
}
```

참고로 `Thread.sleep` 대신 블라킹이 없는 (non-blocking) `IO.sleep`을 사용하라.

## The One that calls back

최근 자바 라이브러리들은 블라킹대신 콜백을 사용하는 사례가 늘고 있다. 예를 들면, AWS SDK for Java 2.0 업데이트에서는 블라킹 함수를 `CompletableFuture`로 대체하였다. `CompletableFuture`는 `handle` 메소드를 통해 API 호출의 반환에 맞춰서 콜백을 실행한다. 

이러한 함수들을 처리하기 위해 ZIO의 `effectAsync`를 사용할 수 있다. `effectAsync`는 콜백 신호가 발생했을 때 (triggered) 호출할 수 있는 함수를 제공한다. 이 함수를 통해 효과(effect)를 완료시키고 결과 값 또는 실패를 반환한다.

다음 코드에서 `effectAsync`는 `CompletableFutre`로 부터 콜백 신호가 발생했을 때 (triggered) 호출할 수 있는함수인 `cb`를 제공한다. `cb` 는 효과(effect)를 인자로 받을 수 있기 때문에 에러가 발생했을 땐ㄴ `IO.fail`을 넘겨주고 성공한 경우에는 `IO.succeed`를 넘겨준다. (코드에서는 `IO.unit`을 사용하였다. 이는 `IO.succeed(())`와 동일하다.)

```scala
def send(client: SqsAsyncClient, queueUrl: String, msg: String): Task[Unit] =
  IO.effectAsync[Any, Throwable, Unit] { cb =>
    client
      .sendMessage(SendMessageRequest.builder.queueUrl(queueUrl).messageBody(msg).build)
      .handle[Unit]((_, err) => {
        err match {
          case null => cb(IO.unit)
          case ex   => cb(IO.fail(ex))
        }
      })
    ()
  }
```

