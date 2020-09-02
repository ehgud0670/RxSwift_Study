[Reactive-X][1] 문서를 번역한 글입니다. 

# Observable 

* Observable이란? => 비동기적으로 다수의 이벤트를 다루는 방법

ReactiveX 에서는 observer가 Observable를 구독(subscribe)합니다.
그 다음 observer는 Observable이 방출하는 하나 또는 연속된 항목에 반응합니다. 
이 패턴은 병행적(concurrent: 동시성) 작업에 편리합니다. 
그 이유는 Observable이 객체를 배출(emission)할 때까지 기다릴 필요 없이 어떤 객체가 배출되면 그 시점을 감시하는 관찰자를 옵저버 안에 두고 그 관찰자를 통해 배출 알림을 받으면 되기 때문입니다.

이 페이지는 리액티브 패턴이 무엇이고 그리고 Observable이 무엇이고 observer가 무엇인지 설명합니다(그리고 어떻게 observer가 observable을 구독하는지).
<br>다른 페이지에서는 다양한 Observable 연산자를 사용하여 Observable을 함께 연결하고 그들의 동작을 변경하는 방법을 보여줍니다.

이 문서는 "마블 다이어그램"과 함께 ReactiveX를 설명합니다. 여기는 marble 다이어그램이 Observable과 Observable의 변환을 나타내는 방법입니다. 

<img width="605" alt="Observable" src="https://user-images.githubusercontent.com/38216027/91865851-c90fb000-ecac-11ea-9502-16a78fa23ae2.png">

*이건 Observable의 타임라인이다. 시간은 왼쪽(start)에서 오른쪽(end)으로 간다.*
<br>*Observable에 의해 방출된 항목들입니다.*
<br>*이 수직선은 Observable이 성공적으로 끝났음을 가리킵니다.*
<br>*이 Observable은 변환의 결과입니다.*
<br>*어떤 이유로 Observable이 비정상적, 에러와 함께 끝난다면, 수직선은 X로 대체됩니다.*

## Background

많은 소프트웨어 프로그래밍 작업에서, 당신이 작성한 명령들이 작성한 순서대로 한 번에 하나씩 점진적으로 실행되고 완료될 것으로 기대합니다. 
그러나 ReactiveX에서는 많은 명령이 병렬(parallel)로 실행될 수 있으며 그 결과는 나중에 "관찰자"에 의해 임의의 순서로 캡처됩니다. 
메서드를 호출하는 대신 "관찰 가능한" 형식으로 데이터를 검색(retrieving)하고 변환(transforming)하는 메커니즘을 정의하고, 그 다음 관찰자가 이 메커니즘을 구독하는 그 순간, 이전에 정의된 메커니즘이 감시자(sentry)가 서있는 상태에서 작동합니다. 배출물들이 준비될 때마다 포착하고 대응합니다.

이 접근 방식의 장점은 서로 의존(not dependant)하지 않는 여러 작업이 있는 경우 다음 작업을 시작하기 전에 각큼작업이 완료 될 때까지 기다리지 않고 모든 작업을 동시에(parallel) 시작할 수 있다는 것입니다. 
이 방식으로 당신의 전체 태스크 번들은 번들에서 가장 긴 태스크만큼만 시간이 걸립니다. 
(이 말은 즉슨 총 걸린 시간이 제일 오래 걸릴 태스크만큼이라는 것이다. 모두 동시에 작업이 진행되니깐)

이 비동기 프로그래밍 및 디자인 모델을 설명하는 데 사용되는 많은 용어가 있습니다. 이 문서는 다음 용어를 사용합니다. observer는 Observable을 구독합니다. Observable은 observer의 메서드(onNext, onError, onCompleted)를 호출함으로써 항목을 방출(next)하거나 관찰자에게 알림(error, completed)을 보냅니다.

다른 문서 및 기타 컨텍스트에서 "observer"라고 부르는 것을 "subscriber", "watcher" 또는 "reactor"라고도 합니다. 이 모델은 일반적으로 "reactor pattern" 이라고도 합니다.즉 Observer가 Reactor다.

## Establishing Observers

이 페이지에서는 예제로 Groovy와 유사한 의사 코드를 사용하지만 여러 언어로 된 ReactiveX 구현이 있습니다.

ReactiveX에서의 일반적인 비동기식 병렬 호출이 아닌 원래의 일반적인 메서드 호출에서 흐름은 다음과 같습니다.

1. 메소드를 호출합니다. 
2. 변수에 메소드로부터 저장된 값을 저장합니다. 
3. 변수와 그 새 값을 유용한 일을 하기 위해 사용합니다. 

코드로 표현하면 아래와 같습니다.

```groovy 
// make the call, assign its return value to `returnVal`
returnVal = someMethod(itsParameters);
// do something useful with returnVal
```

비동기식(Asynchronous) 모델에서 흐름은 다음과 같습니다.

1. 비동기적인 호출의 반환값으로 유용한 작업을 수행하는 함수를 정의하세요. 이 메소드는 observer의 일부입니다.
2. Observable로 비동기 호출을 정의합니다.
3. subscribe(구독)을 통해 observer을 Observable에 연결합니다. (이것은 또한 Observable의 액션을 초기화합니다.) 
4. 그럼 이제 다른 일 보세요. 메소드 호출로 결과가 리턴될 때마다, 옵저버의 메소드는 리턴값 또는 Observable이 방출하는 항목들로 연산하기 시작할 것입니다.

코드로 표현하면 아래와 같습니다.

```groovy
// defines, but does not invoke, the Subscriber's onNext handler
// (in this example, the observer is very simple and has only an onNext handler)
def myOnNext = { it -> do something useful with it };
// defines, but does not invoke, the Observable
def myObservable = someObservable(itsParameters);
// subscribes the Subscriber to the Observable, and invokes the Observable
myObservable.subscribe(myOnNext);
// go on about my business

```

### onNext, onCompleted, and onError

`Subscribe` 메소드를 통해 옵저버와 Observable을 연결합니다. 옵저버는 아래의 메소드들을 구현합니다.

`onNext`
<br>Observable을 항목을 배출(emit)할 때마다 이 메소드를 호출합니다.
이 메소드는 Observable이 배출한 항목을 파라미터로 갖습니다. 

`onError`
<br>Observable은 예상된 데이터를 발생시키는데 실패하거나 또는 다른 에러를 만나게 되면 이 메소드를 호출합니다. 
그럼 더 이상 `onNext`나 `onCompleted`를 호출하지 않습니다. `onError` 메소드는 어떤 에러가 발생했는지 정보를 갖는 객체를 파라미터로 갖습니다.

the Observable Contract의 면에서, onNext는 0번 이상 호출 될 수 있으며 그 후에는 onCompleted 또는 onError 둘 중 하나를 마지막으로 호출합니다. 단, 이 둘 모두를 호출하지는 않습니다.

**관례적으로, 이 문서에서는 `onNext`를 주로 항목들의 배출(emissions)이라 부르고, 반면에 `onCompleted`나 `onError`는 노티피케이션(Notification)이라고 부릅니다.**

더 완벽한 `subscribe` 호출은 아래와 같습니다.

```groovy
def myOnNext = { item -> /* do something useful with item */ };
def myError = { throwable -> /* react sensibly to a failed call */ };
def myComplete = { /* clean up after the final response */ };
def myObservable = someMethod(itsParameters);
myObservable.subscribe(myOnNext, myError, myComplete);
// go on about my business
```

## Unsubscribing

ReactiveX 에는 Subscriber라는 특별한 옵저버 인터페이스가 있는데, 이 Subscriber는 unsubscribe라는 메소드를 구현합니다. 
현재 구독 중인 Observable 중, 옵저버가 더 이상 구독을 원하지 않는 경우에는 이 메서드를 호출해서 구독을 해지할 수 있습니다. ( Subscriber == 옵저버 )
<br>만약 더 이상 관심있는 다른 옵저버가 존재하지 않는다면 Observable들은 새로운 항목들을 배출하지 않습니다.

이 구독 취소의 결과는 observer가 구독한 Observable에 적용되는 연산자 체인을 통해 다시 계단식으로 전달되며, 이로 인해 체인의 **각 링크가 항목 방출을 중지**하게 됩니다. 그러나 이것은 즉시 발생하는 것이 보장되지 않으며 **심지어 이러한 방출을 관찰할 observer가 남아 있지 않은 후에도** Observable이 잠시 동안 항목을 생성하고 방출하려고 시도 할 수 있습니다.

## “Hot” and “Cold” Observables

언제 Observable이 일련의 항목들을 배출(emit)할까요? 이것은 Observable에 따라 다릅니다. "hot" Observable은 생성되자마자 항목들을 배출할 수도 있기 때문에, 이 Observable을 구독하는 옵저버들은 어떤 경우에는 항목들이 배출되는 중간부터 Observable을 구독할 수 있습니다.

반면에, "cold" Observable은 옵저버가 구독할때까지 항목들을 배출하지 않습니다. 따라서 observer는 시작부터 항목 전체를 구독할 수 있도록 보장받습니다.  

ReactiveX의 구현 코드 중에는 “연결 가능한(Connectable)” Observable이라고 불리는 Observable 객체가 존재하는데, 이 Observable은 옵저버의 구독 여부와는 상관 없이 자신의 Connect 메서드가 호출되기 전까지 항목들을 배출하지 않습니다. (신기하구먼)

## Observable 연산자를 활용한 구성 

Observable과 옵저버는 그저 ReactiveX의 시작점일 뿐입니다. 우리가 알고 있는 표준 옵저버 패턴을 조금 확장한 것이며, 연속된 이벤트를 처리하는데 있어서는 싱글 콜백보다는 훨씬 더 효과적인 방법을 제공합니다.

"리액티브 확장(reactive extensions)"(그래서 "ReactiveX"로 부르는)의 진짜 힘은 **연산자**로부터 나옵니다.(두둥!) 연산자들은 Observable이 배출하는 연속된 항목들을 변환(transform)시키고, 결합(combine)하고, 조작(manipulate)하는 기능들을 제공한다.

이 연산자들은 콜백이 제공하는 효율적인 장점들을 바탕으로 선언적인 방법을 통해 연속된 비동기 호출을 구성할 수 있는 방법을 제공하는데, **중요한 것은 일반적인 비동기 시스템이 가진 중첩된 콜백 핸들러의 단점들(콜백 지옥)을 제거했다는 점**이다.

=> 그건 바로 연산자 체인이고, 이건 매우 강력하다!!

## Chaining Operators

대부분의 연산자들은 **Observable 상에서 동작하고 Observable을 리턴**한다. 이런 접근 방법은 연산자들을 하나의 Observable에 적용하고 또 다음 연산자에 다시 적용할 수 있는 **연산자 체인**을 제공한다. 연산자 체인에 걸려있는 각각의 연산자들은 이전 연산자가 리턴한 Observable을 변경한다.

특정 클래스의 다양한 메서드 연산을 통해서 같은 클래스에 있는 항목들을 변경하는 빌더 패턴 같은 것도 존재한다. 이 패턴 역시 비슷한 방법으로 메서드 체인을 제공한다. **하지만, 연산자의 호출 순서가 문제가 되지 않는 빌더 패턴과는 달리, Obervable 연산자들은 호출 순서에 영향을 받는다.**

=> Observable 연산자들은 호출 순서에 영향을 받는다!!!!

Observable 연산자 체인은 원본 Observable과는 떨어져서 동작할 수 없고 순서대로 동작하기 때문에, **호출 체인 중 바로 이전에 호출된 연산자가 리턴한 Observable을 기반으로 실행된다.**

[1]: http://reactivex.io/documentation/observable.html