* [홈페이지][1]를 번역한 글

[1]: http://reactivex.io/

# Reactive-X

* An API for **asynchronous** programming with **observable streams**
<br>=> observable한 stream과 함께 비동기 프로그래밍을 위한 API

## 옵저버 패턴은 옳습니다. 

<img width="869" alt="reactive-x" src="https://user-images.githubusercontent.com/38216027/91788990-903def80-ec48-11ea-87be-7d94130dd34f.png">
<br>=> 동그라미를 marble이라고 한다. 
<br>=> debounce는 operator 중 하나이다. 

> ReactiveX는 Observer 패턴, Iterator 패턴, 그리고 functional 프로그래밍의 조합입니다. 

* CREATE

  * 쉽게 이벤트 **스트림**, 데이터 **스트림**을 만듭니다.

* COMBINE

  * 쿼리(query)같은 operator로 **스트림**을 구성(compose)하고 변형(transform)합니다. 

* LISTEN

  * side - effect를 행하기 위해 어떠한(any) observable 스트림도 구독(Subscribe)합니다. 

## 더 나은 코드 베이스 (Better codebases)

* Functional 
  * observable 스트림으로 클린한 input / outpout 을 이용해서, **복잡한 stateful 프로그램을 피하세요.** 
<br> => 정말 강력합니다.

* Less is more 
  * ReactiveX의 operator들은 종종 단 몇 줄의 코드로 복잡한 코드를 한번에 줄일 수 있습니다.
<br> => 한 프로세스 과정을 응집시켜서 볼 수 있겠습니다.

* Async Error Handling
  * 전통적인 try/catch 는 **비동기적인 연산에서의 에러에 힘이 없지만**, ReactiveX는 에러 핸들링에 더 적절한 메커니즘을 갖추고 있습니다.

* Concurrency made easy
  * ReactiveX의 Observables 그리고 Schedulers를 사용하면 **프로그래머가 low - level 의 스레딩, 동기화 및 동시성 문제를 추상화** 할 수 있습니다. 


## Reactive Revolution: 리액리브 혁명 

* ReactiveX는 API 그 이상입니다. ReactiveX는 프로그래밍에 대한 한 **아이디어**이자 **혁신**입니다. ReactiveX는 여러 다른 API, 프레임워크, 심지어 프로그래밍 언어에 영감을 주었습니다.