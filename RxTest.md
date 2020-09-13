### RxTest

* RxTest는 RxSwift와 별개로 pod에 추가해야 한다.

```swift
import RxTest

let scheduler = TestScheduler(initialColck: 0)
```

* TestScheduler: 시간에 선형(시간이 흐르는)적인 오퍼레이션 테스트를 세부적으로 제어할 수 있는 virtual time 스케줄러이다.

```swift
override func tearDown() { 
    scheduler.scheduleAt(1000) {
        self.subscription.dispose() 
    }
    scheduler = nil
    super.tearDown() 
}
```
scheduler가 TestTime 이 1000 일때 구독을 dispose 한다는 뜻이다. 

```swift 
func testAmb() {
    let observer = scheduler.createObserver(String.self) 
}
```
observer의 타입은 TestableObserver\<String> 이다. 

```swift
let observableA = scheduler.createHotObservable([
    .next(100, "a"),
    .next(200, "b"),
    .next(300, "c") 
])

let observableB = scheduler.createHotObservable([
    .next(90, "1"), 
    .next(200, "2"), 
    .next(300, "3")
])

let ambObservable = observableA.amb(observableB)

self.subscription = ambObservable.subscribe(observer)
```

```swift
scheduler.start()
let results = observer.events.compactMap {
    $0.value.element 
}

XCTAssertEqual(results, ["1", "2", "3"])
```

obseervableA와 observableB의 타입은 
TestableObservable\<String> 이다. 

`.next(100, "a")`은 TestTime 100 일 때 "a"를 방출한다는 뜻이다.

따라서 제일 먼저 방출되는 것은 observableB 이므로 amb 오퍼레이터에는 observableB의 항목(item)들만 방출된다. 

따라서 results는 `["1", "2", "3"]` 이다.

### RxBlocking

* RxTest는 RxSwift와 별개로 pod에 추가해야 한다.

`toBlocking(timeout:)`은 observalble을 BlockingObservable 객체로 변환해준다.

`toBlocking(timeout:)`은 observable이 정상적으로 종료되거나 timeout에 도달할 때까지 현재의 스레드를 block하는 것이다.

timeout 인수는 default값이 nil 인 옵셔널 TimeInterval이다. 
timeout 값을 설정하고 Observable이 정상적으로 종료되기 전에 timeout이 경과하면 **toBlocking은 RxError.timeout 오류를 발생시킨다.**

`toBlocking`은 본질적으로 비동기 명령어를 동기 명령어로 만들기 때문에 테스트하는데 사용하기 좋다.

```swift
func testToArray() throws {
    let scheduler = ConcurrentDispatchQueueScheduler(qos: .default)

    let toArrayObservable = Observable.of(1, 2).subscribeOn(scheduler)
    
    XCTAssertEqual(try toArrayObservable.toBlocking().toArray(), [1, 2])
}
```

#### RxBlocking의 materialize 메소드 사용하기

RxBlocking also has a materialize operator that can be used to examine the result of a blocking operation. It will return a MaterializedSequenceResult, which is an enum with two cases with associated values. From the documentation:

RxBlocking에는 blocking operation의 결과를 검사하는데 사용할 수 있는 구체화(materialize) 연산자도 있습니다. 
연관된 값이있는 두 case가 있는 열거형인 MaterializedSequenceResult를 리턴합니다.

```swift
func testToArrayMaterialized() {
    let scheduler = ConcurrentDispatchQueueScheduler(qos: .default)

    let toArrayObservable = Observable.of(1, 2).subscribeOn(scheduler)

    let result = toArrayObservable
      .toBlocking()
      .materialize()

    switch result {
    case .completed(let elements):
      XCTAssertEqual(elements,  [1, 2])
    case .failed(_, let error):
      XCTFail(error.localizedDescription)
    }
}
```

### 프로덕션 코드 테스트 하기 

```swift
func testColorIsRedWhenHexStringIsFF0000() throws {
    let scheduler = ConcurrentDispatchQueueScheduler(qos: .default)
    let colorObservable = viewModel.color.asObservable().subscribeOn(scheduler)
        
    viewModel.hexString.accept("#ff0000")
        
    XCTAssertEqual(try colorObservable.toBlocking(timeout: 1.0).first(), .red)
}
```

viewModel.hexString에 "#ff0000" 값을 넘겨주면  color는 해당 rgb값의 UIColor객체를 방출(emit)한다. 

여기서 `colorObservable.toBlocking(timeout: 1.0).first()` 은 1초 동안 blocking해서 방출된 첫번재 UIColor 객체를 반환한다는 뜻이다. 

**이런식으로 프로덕션 코드에서도 비동기적인 RxSwift의 Operator를 동기적으로 만들어 테스트 할 수 있다.** 