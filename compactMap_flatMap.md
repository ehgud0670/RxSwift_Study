## compactMap을 사용해야 할 때 

* transform이 Optional 값을 생성할 때 nil이 아닌 값의 배열을 받으려면 compactMap을 사용하세요.

```swift
let scores = ["1", "2", "three", "four", "5"]

let mapped = scores.map{ Int($0) }
// [Optional(1), Optional(2), nil, nil, Optional(5)]

let compactMapped = scores.compactMap{ Int($0) }
// [1, 2, 5]
```

## flatMap을 사용해야 할 때

* transform 할 때 또 다른 컬렉션을 양산하는 경우, 이를 결국 single 레벨의 컬렉션으로 만들고 싶을 때 flatMap을 사용하세요 

```swift 
let scoresByName = ["Kim": [0, 1, 2], "Lee": [3, 4, 5]]

let mapped = scoreByName.map { $0.value }
// [[0, 1, 2], [3, 4, 5]]

let flatMapped = scoreByName.flatMap { $0.value }
// [0, 1, 2, 3, 4, 5]
``

사실 `s.flatMap(transfrom)` 은 `Array(s.map(transform).joined())`와 같습니다.

### flatMap 쓰기 적절한 예: Optional(Optional) 타입을 Optional로 바꾸고 싶을 때  

```swift
let str: String? = "6"
let num = str.map { Int($0) }
// Optional(Optional(6))
```
num의 타입은 뭔 줄 아시나요? 바로 Int?? 입니다. str 자체도 옵셔널 타입이고, Int($0)도 옵셔널 타입이라 Int??가 된 것이죠.

바로 이때 flatMap을 쓰기 좋습니다. Optional(Optional)을 단일한 레벨의 Optional로 만들기 때문입니다. 

```swift
let num = str.flatMap { Int($0) }
// Optional(6)
```

## compactMap vs flatMap

*코드를 치면서 느낀 일반적인 경험상 규칙*

Optional 값을 반환하는 transform이 있다면 compactMap을 사용하세요. 그 이외의 경우라면 map과 flatMap이 필요한 결과를 제공합니다.


출처 : <https://www.avanderlee.com/swift/compactmap-flatmap-differences-explained/>