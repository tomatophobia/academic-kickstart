---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "ZIO를 통한 부수효과가 있는 코드 관리 [번역]"
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

물론 ZIO에는 이를 위한 해답이 있다. 바로 `effectBlocking`으로 코드를 감싸는 것이다. `effectBlocking`은 해당 코드만을 위한 (dedicated) 쓰레드 풀에서 코드를 실행시키는 것을 보장한다. 반환하는 타입은 ZIO[Blocking, Throwable, A] 이다. 우리는 반환 타입을 통해 "블라킹 환경(= 사용할 쓰레드 풀)" 을 필요로 하면서 에러가 발생할 수 있음을 알 수 있다. 블라킹 환경을 만드는 방법을 몰라도 걱정할 필요없다. 실행할 메인 함수에서  [`zio.App`](https://zio.dev/docs/overview/overview_running_effects#app) 을 사용하면 알맞은 기본 환경이 제공된다.

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

단, `effectAsync`는 콜백이 한 번만 호출될 경우에만 사용할 수 있다. 뒤에서 콜백이 여러 번 호출되는 경우에 사용할 수 있는 방법에 대해 설명하겠다.

## The One living in the Future

우리는 `effectAsync`를 스칼라의 `Future`가 사용되는 곳에서도 사용할 수 있지만 `IO.fromFuture`라는 더 간단한 방법이 있다. `IO.fromFuture`로 감싼 코드에는 암시적으로(implicit) `ExecutionContext`가 제공되어 `Future`를 생성하는데 사용하는 거나 그 외 상황에 맞게 사용할 수 있다.

다음 예시는 Akka의 `ask` 메소드 (`?`로 사용)를 `IO.fromFuture`로 감싸서 사용하는 방법을 보여준다. 결과 효과(effect)는 `Future`가 완료될 때 반환되지만 `Await.result`를 사용할 때 처럼 사용 중인 쓰레드를 블락시키지 않는다. `IO.fromFuture`의 반환 타입은 성공하거나 예외 발생으로 실패할 수 있음을 나타내는 `Task`이다. 

```scala
def sendMsg(actor: ActorRef, msg: String): Task[String] =
  IO.fromFuture { implicit ctx =>
    (actor ? msg).mapTo[String]
  }
```

## The One with the clean stop

어떤  API를 통해 제공받은 오브젝트는 사용한 리소스를 놓아주기 (free) 위하여 사용이 끝난 후에 명시적으로 닫아주어야 한다. 예를 들어 Akka의 `ActorSystem`은 깔끔한 중단(clean stop)을 위한 `terminate` 메소드가 있다. 이 메소드는 Akka로 클러스터링된 노드들을 가지고 있을 때 매우 중요하게 쓰인다. 액터 시스템이 중단되면 `terminate` 메소드를 통해 다른 노드들에 이 사실을 알린다. 이를 통해 라우터, 샤드 그 외 여러 분산된 컴포넌트들은 해당 액터 시스템으로 메시지를 보내는 것을 멈춘다. 만약 `terminate`를 사용하여 액터 시스템을 종료하지 않을 경우 다른 클러스터에서 관련된 노드에 도달할 수 없다는 것을 알아채는데 시간이 걸릴 것이다. 그리고 이는 메시지 유실을 초래할 수 있다.

ZIO에서는 초기화와 종료 로직이 필요한 코드를 감싸는 데이터 타입인 `Managed`가 있다. `Managed`의 생성자는 두 개의 함수를 인자로 받는다. 하나는 오브젝트를 생성할 때 쓰이고 다른 하나는 오브젝트를 방출(release)할 때 사용한다. Java의 `try/finally`와 유사하게 ZIO는 어떤 일이 발생해도 반드시 방출 함수의 호출을 보장한다. 

다음은 `Managed`를 `Task` (`ActorSystem` 생성은 에러가 발생할 수 있다.) 와 `Task.fromFuture` (`terminate` 함수는 `Future` 를 반환한다.) 함께 사용한 예시이다.

```scala
Managed.make(Task(ActorSystem("Chat")))(sys => Task.fromFuture(_ => sys.terminate()).ignore).use {
  actorSystem =>
  // the actor system will be terminated whenever this code completes
}
```

## The One with the loop

이번엔 조금 더 복잡한 상황을 볼 것이다. 예를 들어 데이터를 폴링(polling) 하면서 0 또는 그 보다 많은 갯수의 원소들을 반환하는 API를 사용한다고 하자. 이 경우 계속해서 원소들을 받아오기 위해 반복문 등을 사용할 것이다. 이는 ZIO의 `Streams`를 사용하기 적절한 상황이다.

예시로 AWS SDK for SQS를 이용하겠다. 해당 SDK에는 큐(queue)에서 원소를 받아오는 API가 존재한다. 먼저 이전 예시에서 사용한 것 처럼 `IO.effectAsync`로 감싸면 `Task[List[Message]]`를 받을 수 있다. 그리고 `Stream.fromEffect`를 호출해서 스트림으로 변환하여 `Stream[Throwable, List[Message]]` 타입으로 만든다. 우리는 이 아이템들을 리스트 뭉치가 아닌 하나씩 처리하기 위해 `Stream.fromIterable`과 간단하게 `flatMap`을 이용해 `Stream[Stream[Message]]`를 `Stream[Message]`로 만들 수 있다.

추가적으로 우리가 모든 메시지를 소모할 때마다 메시지를 추가하는 과정을 반복해야 한다. 이를 위해 `.forever`를 사용하면 간편하게 무한한 반복 사이클을 만들 수 있다.

```scala
Stream.fromEffect {
  IO.effectAsync[Any, Throwable, List[Message]] { cb =>
    client
      .receiveMessage(ReceiveMessageRequest.builder.queueUrl(queueUrl).maxNumberOfMessages(10).build)
      .handle[Unit]((result, err) => {
        err match {
          case null => cb(IO.succeed(result.messages.asScala.toList))
          case ex   => cb(IO.fail(ex))
        }
      })
    ()
  }
}.forever
 .flatMap[Any, Throwable, Message](Stream.fromIterable)
```

이 방법을 사용할 경우 메시지를 소모하는 속도보다 더 빠르게 메시지를 추가하는 것을 방지할 수 있다. 이 외에도 `Streams`와 관련된 [다양한 컴비네이터](https://javadoc.io/doc/dev.zio/zio-streams_2.12/1.0.0-RC10-1)들을 쉽게 사용해볼 수 있다. 한 번 둘러보길 추천한다.

## The One that's pushy

방금 우리는 데이터를 받아오는 경우 처리하는 방법에 논의했다. 그렇다면 데이터를 보내는 경우는 어떻게 처리할까? 곤란하게도 이 경우에는 `effectAsync`를 사용할 수 없다. 왜냐하면 `effectAsync`는 콜백이 한 번만 호출되기 때문이다. 다행히도 ZIO에는 `Queue`라는 탈출구가 준비되어있다. 

예시로 Akka Distributed PubSub를 보자. 우리는 특정 토픽을 구독하는 순간부터 클러스터 내부의 노드로부터 이 토픽에 퍼블리시되는 메시지들을 받기 시작한다. 

토픽을 구독하기 위해서 먼저 `Queue.unbounded` 또는 백프레셔([back-pressure](https://medium.com/@jayphelps/backpressure-explained-the-flow-of-data-through-software-2350b3e77ce7)) 관리가 되는 메소드(`bounded`, `dropping`, `sliding`) 등을 이용해 `Queue`를 생성한다. 그 다음 토픽을 구독할 Akka 액터를 생성하고 액터 내부에서 `queue.offer`를 이용하여 우리가 받는 모든 메시지를 큐에 집어넣는다. (enqueue) 액터에서 `offer` 효과(effect)를 실행시키기 위해서 `unsafeRunSync`를 호출해야 한다. 참고로 메인 함수 바깥에서 `unsafeRunSync`를 호출하는 것은 흔히 보기 힘든 상황 중 하나이다. 

어플리케이션 내부에 하나의 `Runtime`만이 존재해야 한다. 이를 위해 `IO.runtime`을 이용해 현재 런타임을 액터에 넘기면 액터에서 `unsafeRunSync`를 실행할 수 있게 된다. 

```scala
def listen(actorSystem: ActorSystem, topic: String): Task[Queue[String]] =
  for {
    queue      <- Queue.bounded[String](1000)
    rts        <- Task.runtime[Any]
    _          <- Task(actorSystem.actorOf(Props(new SubscriberActor(topic, rts, queue))))
  } yield queue
  
case class MessageEnvelope(msg: String)
  
class SubscriberActor(topic: String, rts: Runtime[Any], queue: Queue[String]) extends Actor {

  DistributedPubSub(actorSystem).mediator ! Subscribe(topic, self)

  def receive: PartialFunction[Any, Unit] = {
    case MessageEnvelope(s) => rts.unsafeRunSync(queue.offer(s))
  }
}
```

이 패턴에 대해 완전한 구현은 [`zio-akka-cluster`](https://github.com/zio/zio-akka-cluster) 리포지토리에서 확인할 수 있다.

---

지금까지 ZIO가 제공하는 몇 가지 생성자를 이용해 비-함수형 세상에서 사용되는 패턴을 감싸서 사용하는 방법을 알아보았다. 더 복잡한 상황에서 `Managed`, `Stream`, `Queue`를 결합한다면 순수성을 지키면서 문제를 해결할 수 있을 것이다.

이 글이 많은 도움이 되었기를 바라며, 만약 내가 다루지 않은 케이스를 당신이 만났다면 [ZIO Gitter 채널](https://gitter.im/ZIO/Core) 등을 이용해 알려주길 바란다. (역: [ZIO discord 채널](https://discord.gg/2ccFBr4)도 있습니다.)



## 끝

이번에도 함수형 프로그래밍에 관한 좋은 글을 알게 되어 번역해보았습니다. 이 글을 읽으면서 처음 Play Framework를 이용하여 개발을 하던 때가 떠올랐습니다. 당시 사용하던 MongoDB를 이용하기 위한 라이브러리는 DB 조회 결과를 모두 Future 타입으로 반환하였습니다. 이를 순수 함수형 코드로 만들기 위해 많은 자료를 찾아다녔던 기억이 있습니다. 저와 비슷한 고민을 했던 사람들에게 이 글이 도움이 되길 바랍니다.  

번역이 어색하거나 틀린 정보가 있거나 이해가 안되는 점들은 댓글이나 이메일 주시면 정말 감사드리겠습니다. :)