## DisposeBag

아래 예제의 Observable은 VC가 pop되어도 클로저를 모두 수행하고 Completed 된다.

```swift
import UIKit

import RxSwift
import RxCocoa

final class PopButRunningVC: UIViewController {
    let observable = Observable<Int>.interval(
        DispatchTimeInterval.seconds(1),
        scheduler: MainScheduler.init())
        .take(10)
    
    override func viewDidLoad() {
        super.viewDidLoad()
        testRx()
    }
    
    deinit {
        print(#file, #function)
    }
    
    private func testRx() {
        observable
            .subscribe(
                onNext: { print($0) },
                onError: { print($0.localizedDescription) },
                onCompleted: { print("completed") },
                onDisposed: { print("disposed")}
        )
    }
}

// 실행 결과
// 0
// /Users/kimdo/workspace/DisposeBagExample/DisposeBagExample/PopButRunningVC.swift deinit
// 1
// 2
// 3
// 4
// 5
// 6
// 7
// 8
// 9
// completed
// disposed
```

* VC가 dealloc 됐는데 클로저가 계속 작동중인걸 바라는 개발자는 없다. 
<br>따라서 VC가 dealloc 되자마자 클로저가 작동되지 않도록 하자. **어떻게? DisposeBag으로**

## 해결책: DisposeBag을 이용한다. 

```swift
import UIKit

import RxSwift
import RxCocoa

final class PopAndNotRunningVC: UIViewController {
    let observable = Observable<Int>.interval(
        DispatchTimeInterval.seconds(1),
        scheduler: MainScheduler.init())
        .take(10)
    
    var disposeBag = DisposeBag()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        testRx()
    }
    
    deinit {
        print(#file, #function)
    }
    
    private func testRx() {
        observable
            .subscribe(
                onNext: { print($0) },
                onError: { print($0.localizedDescription) },
                onCompleted: { print("completed") },
                onDisposed: { print("disposed")})
            .disposed(by: disposeBag)
    }
}

// 실행 결과 
// 0
// 1
// 2
// 3
// /Users/kimdo/workspace/DisposeBagExample/DisposeBagExample/PopAndNotRunningVC.swift deinit
// disposed
```

위의 코드는 VC가 pop되어 dealloc 되면 프로퍼티인 disposeBag도 dealloc 된다. 
<br>따라서 disposeBag이 dealloc 되면서 가지고 있던 **disposable의 dispose 메소드를 호출하고 따라서 클로저는 동작하지 않는다.** 

## RxCocoa에서의 확장된 Observable

```swift
let observable = textField.rx.text.orEmpty.asObservable()
```

* 위의 observable은 해당 VC가 dealloc 되는 상황만 completed 된다. 다음 예시를 보자 

> example

VC가 pop되어 dealloc 되면 textField도 자연스럽게 dealloc 되고 이후 observable도 completed 되고, disposed 된다. 

```swift 
import UIKit

import RxSwift
import RxCocoa

final class NotCompletedObservableVC: UIViewController {
    @IBOutlet weak var textField: UITextField!

    override func viewDidLoad() {
        super.viewDidLoad()
        testRx()
    }

    deinit {
        print(#file, #function)
    }
    
    private func testRx() {
        let observable2 = textField.rx.text.orEmpty.asObservable()
        
        observable2
            .subscribe(
                onNext: { print($0) },
                onError: { print($0.localizedDescription) },
                onCompleted: { print("completed") },
                onDisposed: { print("disposed")}
        )
    }
}

// 출력결과
// pop 이후 
// /Users/kimdo/workspace/DisposeBagExample/DisposeBagExample/NotCompletedObservableVC.swift deinit
// completed
// disposed
```


* 이외의 경우 이 observable은 계속 이벤트를 기다려야 하므로 completed 되지 않는다. 따라서 잘못하면 **메모리 누수 문제가 발생**할 수 있다. **언제? self를 캡쳐한 경우** 

> example

```swift
import UIKit

import RxSwift
import RxCocoa

final class NotCompletedObservableVC: UIViewController {
    @IBOutlet weak var textField: UITextField!
    
    var disposeBag = DisposeBag()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        testRx()
    }
    
    deinit {
        print(#file, #function)
    }
    
    private func testRx() {
        let observable2 = textField.rx.text.orEmpty.asObservable()
        
        observable2
            .map(toInt(str:))
            .subscribe(
                onNext: { print($0) },
                onError: { print($0.localizedDescription) },
                onCompleted: { print("completed") },
                onDisposed: { print("disposed")}
        ).disposed(by: disposeBag)
    }
    
    private func toInt(str: String) -> Int {
        return str.count
    }
}
```
=> map에서 self를 strong하게 캡쳐하고 있다.

```
textFields's Observable's closure -> VC 
navigationViewController -> VC
: VC's refcount is 2

VC -> textField
: textField's refcount is 1

retain cycle (o)
```

인 상황에서 VC가 pop되어도 closure가 self를 캡처하기 때문에 VC의 refcount는 1이라 VC는 dealloc 되지 않는다. 
<br>또 closure가 절대 completed 되지 않으므로 VC도 영원히 dealloc 되지 않는다.  
그리고 VC가 살아있으므로 disposeBag도 소용없다. 

<br>

### 해결책 1: VC가 사라지는 시점에 DisposeBag 초기화한다.
=> 이러면 disposeBag은 VC를 캡쳐한 클로저를 처분하게 되고 자연히 클로저의 VC에 대한 refcount 가 1 줄게 되면서 VC는 정상적으로 dealloc 된다. 

```swift 
override func viewDidDisappear(_ animated: Bool) {
    disposeBag = DisposeBag()
}
//실행 결과 
// 0
// disposed
// /Users/kimdo/workspace/DisposeBagExample/DisposeBagExample/NotCompletedObservableVC.swift deinit
```
<br>

### 해결책 2: self를 캡쳐하는 부분을 weak하게 캡쳐하도록 한다.
=> 애초에 클로저가 VC의 refCount를 1 올리지 않으므로 VC가 pop 되면 자연히 VC는 dealloc 된다. 

```swift 
observable2
    .map { [weak self] in self?.toInt(str:$0) }
    .subscribe(
        onNext: { print($0) },
        onError: { print($0.localizedDescription) },
        onCompleted: { print("completed") },
        onDisposed: { print("disposed")}
    )

//실행 결과 
// Optional(0)
/// Users/kimdo/workspace/DisposeBagExample/DisposeBagExample/NotCompletedObservableVC.swift deinit
// completed
// disposed
```