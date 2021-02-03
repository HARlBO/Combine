# Chapter 17: Schedulers

## 🐻‍❄️ An introduction to schedulers

애플문서에 따르면 **scheduler** 는 클로저를 언제 어떻게 실행할지 정의하는 프로토콜이다. 정의가 정확하기는 하지만 일부만이다.

스케줄러는 가능한 빨리 혹은 미래 날짜에 미래 작업을 실행할 수 있는 컨텍스트를 제공한다.

액션은 프로토콜 자체에 정의된 클로저이다.

그러나 클로저라는 용어가 특정 스케줄러에서  수항하는 Publisher 로 부터의 전달 되는 값을 숨길 수 있다.

코드가 실행될 쓰레드에 대한 정확한 세부정보는 선택한 스케줄러에 따라 다르다.

**A scheduler is not equal to a thread.**    

![/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled.png](/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled.png)

스케줄러가 foreground / background 실행의 개념에 깊이 뿌리를 두고 있음을 알 수 있다.

또한 선택한 구현에 따라 작업을 직렬화 하거나 병렬화 할 수 있다 (serialized / parallelized)

<br />


## 🐻‍❄️ Operators for scheduling

Combine 프레임워크는 스케줄러를 동작하는 두개의 기본 operator 를 제공한다.

- `subscribe(on:)` 과 `subscribe(on:options:)` 는 특정 스케줄러에 subscripton 을 **생성한다**
- `receive(on:)` 과 `receive(on:options:)` 는 특정 스케줄러에 값을 **전달한다**

### Introducing `subscribe(on:)`

Publisher 는 구독하기 전까지는 동작하지 않는 존재이다. 

Publisher 를 구독하면 :

![/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%201.png](/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%201.png)

Publisher 가 메인 스레드를 차단하지 않도록 백그라운드에서 비용이 많이 드는 계산을 수행하도록 할 수 있다. → `subscribe(on:)` 를 사용하면 된다.

```swift
let computationPublisher = Publishers.ExpensiveComputation(duration :3)

let queue = DispatchQueue(label: "serial queue")

let currentThread = Thread.current.number
print("Start computation publisher on thread \(currentThread)")
```

1. `ExpensiveComputation` 라는 지정된 기간 후에 문자열을 내보내는 장기 실행 계산을 시뮬레이션 하는 pulisher 를 정의 해놓았다.
2. Serial queue 는 특정 스케줄러에서 계산을 트리거 하는데 사용한다. DispatchQueue 는 스케줄러 프로토콜을 따른다.
3.  thread number 1 이라는 플레이그라운드에서 코드가 동작하는 디폴트 쓰레드인, 현재 쓰레드의 번호를 얻는다

<br />


`computationPublisher` 를 구독하면

```swift
let subscription = computationPublisher
    .sink { value in
        let thread = Thread.current.number
        print("Receive compu tation result on thread \(thread): '\(value)'")
    }

Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 1
Beginning expensive computation on thread 1
Completed expensive computation on thread 1
Received computation result on thread 1 'Computation complete'
```

메인스레드인 스레드1 에서 작동되고 구독한다.

<br />


`subscribe(on:)` 을 사용하면

```swift
let subscription = computationPublisher
  .subscribe(on: queue)
  .sink { value in
		let thread = Thread.current.number
		print("Receive compu tation result on thread \(thread): '\(value)'")
   }

Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 5
Beginning expensive computation from thread 5
Completed expensive computation on thread 5
Received computation result on thread 5 'Computation complete'
```

구독은 여전히 메인스레드에서 하지만, 구독을 효과적으로 수행하기 위해 Combine 이 queue 에 위임한다.

queue 는 스레드 중 하나에서 코드를 실행한다.

계산이 스레드 5에서 시간과 완료가 되고 결과 값을 이 스레드에서 emit 하고 이 스레드에서 sink 가 값을 받는다.

> Note : `DispatchQueue` 의 동적 스레드 관리 특성으로 인해, 로그의 스레드 넘버는 다를 수 있다. 중요한 것은 일관성이다. 같은 스레드 넘버가 같은 단계에서 보여질 것이다.

하지만 화면 상의 정보를 업데이트 하려면, 메인스레드에서 UI 업데이트를 수행하기 위해  `DispatchQueue.main.async{...}` 같은 작업을 수행하야 한다.

<br />


### Introducing `receive(on:)`

- Subscriber 에게 값을 전달하는데 사용할 스케줄러를 지정 할 ㅅ

```swift
let subscription = computationPublisher
  .subscribe(on: queue)
  .receive(on: DispatchQueue.main)
  .sink { value in
		let thread = Thread.current.number
		print("Receive compu tation result on thread \(thread): '\(value)'")
   }

Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 4
Beginning expensive computation from thread 4
Completed expensive computation on thread 4
Received computation result on thread 1 'Computation complete'
```

계산 작업은 백그라운드 스레드에서 작업되고 값을 emit 하지만, 값을 받을 때는 메인 큐라는 것이 보장 된다.

UI 업데이트를 안전하게 하기 위해서는 이렇게 수행되어야 한다.

> Note : “ExpensiveComputation subscriber received...” 이 메세지는 Dispatch 가 스레드 풀을 동적으로 관리하므로 다른 스레드 넘버가 될 수 있지만 1은 표시되지 않는다.

<br />


## 🐻‍❄️ Scheduler implementations

애플은 `Scheduler` 프로토콜의 구체적인 구현 여러개를 제공한다

`ImmediateScheduler` 

- 현재 스레드에서 코드를 즉시 실행하는 간단한 스케줄러
다른 operator 를 사용해서 수정하지 않는다면 디폴트 값이다.

`RunLoop` 

- Foundation 의 `Thread` 객체와 묶여있다.

`DispatchQueue`

- serial 이거나 concurrent 일 수 있다

`OperationQueue`

- work item 의 실행을 관리하는

### ImmediateScheduler

- 스케줄러 카테고리에서 가장 쉬운 항목은 Combine 프레임워크가 제공하는 가장 간단한 스케줄러

```swift
let source = Timer
    .publish(every: 1.0, on: .main, in: .common)
    .autoconnect()
    .scan(0) { counter, _ in counter + 1 }

let setupPublisher = { recorder in
    source
        .recordThread(using: recorder)
        .receive(on: ImmediateScheduler.shared)
        .recordThread(using: recorder)
        .eraseToAnyPublisher()
}

let view = ThreadRecorderView(title: "Using ImmediateScheduler", setup: setupPublisher)
PlaygroundPage.current.liveView = UIHostingController(rootView: view)
```

![/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%202.png](/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%202.png)

```swift
.receive(on: DispatchQueue.global())
```

를 추가하면, global concurrent queue 에서 값이 emit 되도록 할 수 있다.

![/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%203.png](/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%203.png)

**ImmediateScheduler options**

- 인수로 `Scheduler` 를 받는 대부분의 operator 는, `SchedulerOption` 을 받는 `options` 인자 또한 볼 수 있다.
- `ImmediateScheduler` 의 경우, `Never` 로 정의 되어 있어서, `options` 파라미터는 넘기면 안된다.

**ImmediateSechduler pitfalls**

- `ImmediateScheduler` 의 한가지 특징은 즉각적 인것이다
- `schedule(after:)` 같은 변형을 사용할 수 없다.

<br />


### RunLoop scheduler

- `DispatchQueue`  이전에, 메인(UI)스레드를 포 함하여 스레드 수준에서 입력 소스를 관리 하는 방법
- 앱의 메인스레드에는 여전히 `RunLoop` 과 관련이 있다
- `RunLoop.current` 를 통해 모든 Foundation 스레드에 대해 하나 얻을 수 있다

> Note : 요즘은 `DispatchQueue` 가 대부분 상황에서 사용되기 때문에 `RunLoop` 는 덜 유용하지만, 여전히 유용한 특정 경우가 있다. 예를 들어, `Timer` 는 `RunLoop` 에 자체적으로 스케줄링한다. UIKit 및 AppKit 은 다양한 유저의 입력 상황을 처리하기 위해 `RunLoop` 에 의존한다.

```swift
let source = Timer
  .publish(every: 1.0, on: .main, in: .common)
  .autoconnect()
  .scan(0) { (counter, _) in counter + 1 }

let setupPublisher = { recorder in
    source
        .receive(on: DispatchQueue.global())
        .recordThread(using: recorder)
        .receive(on: RunLoop.current)
        .recordThread(using: recorder)
        .eraseToAnyPublisher()
}

let view = ThreadRecorderView(title: "Using RunLoop", setup: setupPublisher)
PlaygroundPage.current.liveView = UIHostingController(rootView: view)
```

![/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%204.png](/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%204.png)

**A little comprehension challenge**

`receive(on:)` 대신 `subscribe(on: DispatchQueue.global()` 을 사용하면 모두 스레드 1에서 기록 된다. Publisher 가 현재 queue 에서 구독된다. 그러나 메인 `RunLoop` 에서 값을 emit 하는 `Timer` 를 사용하고 있기 때문에 Publisher 를 구독하기 위해 선택한 스케줄러와 상관 없이 값은 항상 메인스레드에서 시작한다.

**Scheduling code execution with RunLoop**

- 스케줄러를 사용하면 가능한 빨리 또는 미래 날짜 이후에 실행되는 코드를 스케줄링 할 수 있다
- `RunLoop` 는 지연 실행을 수행할 수 있다
- 각각의 스케줄러 구현은 자체 `SchedulerTimeType` 을 정의한다
- `RunLoop` 의 경우 `SchedulerTimeType` 값은 `Date` 이다
- `ThreadRecorder` 클래스에 publisher 에 대한 구독을 중지하는데 사용 할 수 있는 선택적인 `Cancellable` 이 있다

**RunLoop options**

`ImmediateScheduler` 처럼, `RunLoop` 도 호출시, `SchedulerOptions` 파라미터를 갖는 옵션을 제공하지 않는다.

**RunLoop pitfalls**

- `RunLoop` 의 사용은 메인스레드의 실행 루프와, 필요한 경우 제어하는 Foundation 스레드에서 사용할 수 있는 `RunLoop` 로 제한되어야 한다
- `DispatchQueue`  에서 실행되는  `RunLoop.current` 의 사용은 피해야 한다
→ `DispatchQueue` 스레드가 일시작이기 때문에, `RunLoop` 에 의존하는 것은 거의 불가능한다

<br />


### DispatchQueue scheduler

- `Dispatchqueue` 는 `Scheduler` 프로토콜을 따르고
- `Scheduler` 파라미터를 취하는 operator 에서 완전히 사용될 수 있다
- Dispatch 프레임워크는 시스템에의해 관리되는 dispatch queue에 작업을 전송하여 멀티 코어 하드웨어에서 코드를 동시에 실행할 수 있는 Foundation 의 강력한 구성 요소이다

**A serial queue**

- 순서대로 work item 을 실행
- 일부 작업이 겹치지 않음을 보장
- 모든 작업이 동일한 대기열에서 발생하는 경우 lock 없이 공유 리소스 사용

**A concurrent queue** 

- CPU 사용을 최대화 하기 위해, 여러 work item 을 병렬로 실행
- 가능한 많은 작업을 동시에 실행
- 계산에 더 적합

**Queues and thread**

- `DispatchQueue.main` 은 메인(UI)스레드에 직접 매핑 되어 있고, UI 업데이트는 메인스레드에서만 허용된다
- serail or concurrent queue 모두 시스템에서 관리하는 스레드에서 코드를 실행
- 모든 dispatch queue 는 동일한 스레드 풀을 공유한다
- `subscribe(on:)` `recevie(on:)` 같은 `Scheduler` 를 파라미터로 취하는 operator 를 사용할 때, 스레드가 매번 동일하다고 가정하면 안된다

<br />


### Using DispatchQueue as a scheduler

```swift
let serialQueue = DispatchQueue(label: "Serial queue")
let sourceQueue = DispatchQueue.main

let source = PassthroughSubject<Void, Never>()

let subscription = sourceQueue.schedule(after: sourceQueue.now, interval: .seconds(1)) {
    source.send()
}

let setupPublisher = { recorder in
    source
        .recordThread(using: recorder)
        .receive(on: serialQueue)
        .recordThread(using: recorder)
        .eraseToAnyPublisher()
}

let view = ThreadRecorderView(title: "Using DispatchQueue", setup: setupPublisher)
PlaygroundPage.current.liveView = UIHostingController(rootView: view)
```

![/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%205.png](/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%205.png)

두번째 `recoredThread(using:)` 이 `receive(on:)`  operator 이후에 현재 스레드의 변경 사항을 기록한다.

`DispatchQueue` 가 work item 이 실행되는 스레드를 보장하지 않는 다는 것을 보여준다.

`recevie(on:)` 의 경우 **work item** 이 현재 스케줄러에서 다른 스케줄러로 이동하는 값이다.

<br />


`sourceQueue` 를 serialQueue 로 바꾸면

```swift
let sourceQueue = serialQueue
```

![/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%206.png](/Combine/Section4_Advanced_Combine/images/Chapter17_Schedulers/Untitled%206.png)

`DispatchQueue` 의 스레드를 보장해주지 않지만, `receive(on:)` 이 스레드를 바꾸지 않는다.

추가 전환을 피하기 위해 내부적으로 최적화가 진행된 것 같다.

**DispatchQueue options**

- Operator 가 `SchedulerOption` 인자를 사용할 때 전달할 수 있는 옵션을 제공하는 유일한 스케줄러
- 이 옵션은 주로 QoS 값을 지정하는 것

```swift
.receive(
  on: serialQueue,
  options: DispatchQueue.SchedulerOptions(qos: .userInteractive)
)
```

가능한 빨리 UI 업데이트를 할경우 사용하고, 빠른 전송에 대한 부담이 적을 때는 `.background` 를 사용하면 된다.

<br />


### OperationQueue

- 작업 실행을 규제하는 규제하는 시스템 클래스이다
- 종속성이 있는 작업을 생성 할수 있는 규제 매커니즘이지만, Combine 에서는 다르다

```swift
let queue = OperationQueue()

let subscription = (1...10).publisher
    .receive(on: queue)
    .sink { value in
        print("Received \(value) on thread \(Thread.current.number)")
    }

Received 1 on thread 5
Received 2 on thread 4
Received 4 on thread 7
Received 7 on thread 8
Received 6 on thread 9
Received 10 on thread 10
Received 5 on thread 11
Received 9 on thread 12
Received 3 on thread 13
```

- 각각의 값은 다른스레드에서 수신된다. 각 값에 대해 동일한 스레드를 사용한다는 보장이 없다.
- `OperationQueue` 에는 `maxConcurrentOperationCount` 파라미터가 있고, 
operation queue 가 동시에 많은 operation 을 실행할 수 있게 해준다
- Publisher 가 거의 동시에 아이템을 emit 하기 때문에, Dispatch 의 concurrent queue 에 의해 다수 스레드에 dispatch 된다

<br />


queue 정의 뒤에 아래를 추가하면

```swift
queue.maxConcurrentOperationCount = 1

Received 1 on thread 3
Received 2 on thread 3
Received 3 on thread 3
Received 4 on thread 3
Received 5 on thread 4
Received 6 on thread 3
Received 7 on thread 3
Received 8 on thread 3
Received 9 on thread 3
Received 10 on thread 3
```

하나의 serial queue 를 사용해서 순서대로 실행되고 순서대로 값을 받는다.

**OperationQueue options**

`OperationQueue` 에 사용할 수 있는 `SchedulerOptions` 가 없다

**OperationQueue pitfalls**

- 기본적으로 `OperationQueue` 는 concurrent `DispatchQueue` 처럼 동작하지만,
- Publisher 가 값을 emit 할때마다, 수행해야 할 중요한 작업이 있을때 좋은 도구가 된다
- `maxConcurrentOperationCount` 파라미터를 조정하여 제어할 수 있다

<br />


## 🔑  Key points

- `scheduler` 는 작업에 대한 실행 컨텍스트를 정의한다
- 애플의 OS 는 코드 실행을 스케줄링하는 다양한 도구를 제공한다
- `Scheduler` 프로토콜로 상황에 가장 적합한 스케줄러를 선택할 수 있다
- `receive(on:)` 을 사용할 때 마다, Publisher 의 operator 들은 특정 스케줄러에서 실행된다. 즉 자체적으로 `Scheduler` 파라미터를 취할 필요가 없다
