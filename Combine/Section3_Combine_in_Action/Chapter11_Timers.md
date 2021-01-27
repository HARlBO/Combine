# Chapter 11: Timers

## 🐻‍❄️ Using RunLoop

- `Thread` 클래스를 사용하는 메인 스레드와 직접 생성한 스레드는 `RunLoop` 를 가질 수 있다.
- 현재 스레드에서 `RunLoop.current` 를 호출하면, Foundation 이 생성 해 줄 것이다.

> Note : `RunLoop` 클래스는 thread-safe 하지 않다. 현재 스레드의 run loop 에서만 `RunLoop` 메소드를 호출해야 한다.

<br />


`RunLoop` 는 `Scheduler` 프로토콜을 구현한다. low-level 에 관련된 메소드를 정의하고, cancellable 타이머 생성할 수 있는 하나가 있다.

```swift
let runLoop = RunLoop.main

let subscription = runLoop.schedule(
	after: runLoop.new, 
	interval: .seconds(1),
	tolerance: .milliseconds(100)
) {
	print("Timer fired")
}
```

타이머는 어떤 값도 전달하지 않고 publisher 를 생성하지도 않는다. 

<br />


Combine 과 관련해서, 유일한 유용한 점은 `Cancellable` 을 통해 잠시동안 타이머를 멈출 수 있게 해준다.

```swift
runLoop.schedule(after: .init(Date(timeIntervalSinceNow: 3.0))) {
	cancellable.cancel()
}
```

하지만 `RunLoop` 는 타이머를 생성하는 좋은 방법이 아니다. `Timer` 클래스를 사용하는게 더 낫다!

<br />


## 🐻‍❄️ Using the Timer class

Combine 에서는 boilerplate 없이도 publisher 로  바로 사용 할 수 있다.

```swift
let publisher = Timer.publish(every: 1.0, on: .main, in: .common)
```

`RunLoop.current` 를 호출하면 직접 생성했거나 Foundation 에서 얻은 스레드에서도 `RunLoop` 를 얻을 수 있다.

```swift
let publisher = Timer.publish(every: 1.0, on: .current, in: .common)
```

> Note : 이 코드를 `DispatchQueue.main` 이외의 Dispatch queue 에서 실행하면 예측하지 못한 결과가 발생할 수 있다. Dispatch 프레임워크는 run loop 을 사용하지 않고 스레드를 관리한다. run loop 이 `run` 메소드가 프로세스 이벤트에서 호출되기를 요구 하기 때문에,  main 이외의 다른 queue 에서 타이머가 종료되는 것을 볼 수 없을 것이다. 안전하게 `RunLoop.main` 은 타이머에게만 타겟을 걸어야 한다.

<br />


`ConnectablePublisher`

- 타이머가 리턴하는 publisher
- `connect()` 메소드를 호출 하기 전까지 subscription 에서 시작 하지 않는 `Publishr` 변수
- `autoconnect()` operator 를 사용해서 처음 subscriber 가 구독 했을 때 자동으로 연결 할 수 있다.
- publisher 를 생성하는 가장 좋은 방법은 subscription 시 타이머를 시작 하는것

```swift
let publisher = Timer
	.publish(ever: 1.0, on: .main,  in: .common)
	.autoconnect()
```

이 타이머는 반복적으로 현재 날짜를 emit , `Publisher.Output` 타입은 `Date` 

`scan` operator 를 사용해서 타이머가 증가하는 값을 emit 하게 할 수 있다.

```swift
let subscription = Timer
	.publish(every: 1.0, on: .main,  in: .common)
	.autoconnect()
	.scan(0) { counter, _ in counter + 1 }
	.sink { counter in
		print("Counter is \(counter)")
}
```

<br />


## 🐻‍❄️ Using DispatchQueue

- dispatch queue 를 사용해 타이머 이벤트를 발생시키기 위해 수 있다.

```swift
let queue = DispatchQueue.main

let source = PassthroughSubject<Int, Never>()

var counter = 0

let cancellable = queue.scheule(
	after: queue.now,
	interval: .seconds(1)
) {
	source.send(counter)
	counter += 1
}

let subscription = source.sink {
	print("Timer emitted \($0)")
}
```

<br />


## 🔑 Key points

- `RunLoop` 클래스로 타이머를 생성 하는 것은 Objective-C 의 오래된 방식이다.
- `Timer.publish` 로 지정된 `RunLoop` 에서 지정된 간격마다 값을 생성하는 publisher 를 얻기 위해 사용한다.
- `DispatchQueue.schedule` 는 dispatch queue 에서 이벤트를 emit 하는 타이머이다.
