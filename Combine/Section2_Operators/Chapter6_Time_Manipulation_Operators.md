# Chapter 6: Time Manipulation Operators

반응형 프로그래밍의 핵심 아이디어는 시간 흐름에 따른 비동기 이벤트 흐름을 모델링 하는 것이다. 이러한 측면에서, Combine 프레임워크는 시간을 다룰 수 있는 operator 들을 제공 한다. 특히, 시간이 지남에 따라 시퀀스가 어떻게 반응하고 값을 변형 하는지.

<br />

# 🐻‍❄️ Shifting time

가장 기본적인 시간 조작 operator 는 publisher 가 emit 하는 값을 지연시켜서 실제 발생하는 것 보다 늦게 볼 수 있도록 한다.

`delay(for:tolerance:scheduler:options)` operator 는 값 시퀀스 전체의 시간을 옮긴다. Upstream publisher 가 값을 emit 할 때 마다, `delay` 는 잠시 동안 유지한 후에 Scheduler 에 명시 해놓은 만큼 지연 시킨다.

추후에 조정 할 수 있는 두가지 상수를 정의 한다.

매 초마다 하나의 값을 emit 하는 publisher 를 생성하고, 1.5 초 지연 시키고, 비교를 위해 두가지 타임라인을 동시에 표시한다. 

```swift
let valuesPerSecond = 1.0
let delayInSeconds = 1.5

let sourcePublisher = PassthroughSubject<Date, Never>()

let delayedPublisher = sourcePublisher.delay(for: .seconds(delayInSeconds), scheduler: DispatchQueue.main)

let subscription = Timer
    .publish(every: 1.0 / valuesPerSecond, on: .main, in: .common)
    .autoconnect()
    .subscribe(sourcePublisher)
```

<br />

> Note : 특정 타이머는 Foundation `Timer` 클래스에 있는 Combine extension 이다. `RunLoop` 와 `RunLoop.Mode` 를 갖고, `DispatchQeue` 는 아니다. 또한, timer 는 connectable 한 publisher 의 클래스의 한 부분이다. 이것은 값을 emit 하는 것을 시작 하기 전에 연결 되어야 한다는 것이다. 첫번째 subscription 에서 즉시 연결하는 `autoconnect()` 를 사용하면 된다.

```swift
let sourceTimeline = TimelineView(title: "Emitted values (\(valuesPerSecond) per sec.):")
let delayedTimeline = TimelineView(title: "Delayed values (\(delayInSeconds)s delay):")

let view = VStack(spacing: 50) {
    sourceTimeline
    delayedTimeline
}

PlaygroundPage.current.liveView = UIHostingController(rootView: view.frame(width: 375, height: 600))
```

위 코드를 추가 하면, 화면에 두개의 빈 타임라인을 볼 수 있다. 각각의 publisher 에서 emit 된 값을 넣어 주어야 한다.

```swift
sourcePublisher.displayEvents(in: sourceTimeline)
delayedPublisher.displayEvents(in: delayedTimeline)
```

<br />

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled.png)

위에 타임라인은 timer 에 의해 emit 된 값을 보여준다. 아래는 값은 값을 지연된 타임라인을 보여 준다.

원 안의 숫자들은 실제 그들의 값이 아닌 emit 된 count 를 반영한다.

> Note : Static timeline 은 보통 왼쪽으로 값을 정렬 한다. 하지만 위에 애니메이션 다이어그램을 보면, 가장 최신 것이 오른쪽에 오른쪽에 있다.

<br />

# 🐻‍❄️ Collecting values

특정 상황에서, 지정된 간격으로 publisher 로 부터 emit 된 값을 수집 할 필요가 있을 것이다. 이것은 유용하게 쓰일 수 있는 버퍼링의 한 형태다.

예를 들어서, 짧은 기간 동안 값들의 평균을 내고 평균 값을 output 하고 싶을 때.

```swift
let valuesPerSecond = 1.0
let collectTimeStride = 4

let sourcePublisher = PassthroughSubject<Date, Never>()

let collectedPublisher = sourcePublisher
    .collect(.byTime(DispatchQueue.main, .seconds(collectTimeStride)))

let subscription = Timer
    .publish(every: 1.0 / valuesPerSecond, on: .main, in: .common)
    .autoconnect()
    .subscribe(sourcePublisher)

let sourceTimeline = TimelineView(title: "Emitted values:")
let collectedTimeline = TimelineView(title: "Collected values (every \(collectTimeStride)s):")

let view = VStack(spacing: 40) {
    sourceTimeline
    collectedTimeline
}

PlaygroundPage.current.liveView = UIHostingController(rootView: view.frame(width: 375, height: 600))

sourcePublisher.displayEvents(in: sourceTimeline)
collectedPublisher.displayEvents(in: collectedTimeline)
```

<br />

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%201.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%201.png)

Emitted values 타임라인에는 일정한 값이 emit 되고, 아래 Collected values 는 매 4초 마다 단일 값이 표시 된다.

각 4초 동안 받는 값이 array 이고 이 값을 보기 위해 `collectedPublisher` 에 `flatMap` operator 를 추가해 보자.

```swift
let collectedPublisher = sourcePublisher
  .collect(.byTime(DispatchQueue.main, .seconds(collectTimeStride)))
  .flatMap { dates in dates.publisher }
```

`collect` 가 수집한 값의 그룹을  emit 할 때마다, `flatMap` 은 각각의 값으로 분해하고 즉시 바로 다음 값을 emit 한다. `publisher` extension 인 `Collection` 을 사용해 값의 시퀀스를 모든 시퀀스를 각각 개별 값으로 즉시 emit 하는 Pulisher 에게 보낸다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%202.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%202.png)

이제 매 4초마다, `collect` 가 지난 시간 간격 동안 수집 된 값의 배열을 emit 하는 것을 볼 수 있다.

<br />

# 🐻‍❄️ Collecting values (part 2)

`collect(_*:options:*)` operator 가 제공하는 두번째 옵션은 규칙적인 간격으로 값을 수집 할 수 있게 해주는 것이다. 또한 수집된 값의 개수에 제한을 걸 수 있다.

```swift
let valuesPerSecond = 1.0
let collectTimeStride = 4
let collectMaxCount = 2

let sourcePublisher = PassthroughSubject<Date, Never>()

let collectedPublisher = sourcePublisher
 .collect(.byTime(DispatchQueue.main, .seconds(collectTimeStride)))
 .flatMap { dates in dates.publisher }
let collectedPublisher2 = sourcePublisher
    .collect(.byTimeOrCount(DispatchQueue.main, .seconds(collectTimeStride), collectMaxCount))
    .flatMap { dates in dates.publisher }

let subscription = Timer
    .publish(every: 1.0 / valuesPerSecond, on: .main, in: .common)
    .autoconnect()
    .subscribe(sourcePublisher)

let sourceTimeline = TimelineView(title: "Emitted values:")
let collectedTimeline = TimelineView(title: "Collected values (every \(collectTimeStride)s):")
let collectedTimeline2 = TimelineView(title: "Collected values (at most \(collectMaxCount) every \(collectTimeStride)s):")

let view = VStack(spacing: 40) {
    sourceTimeline
    collectedTimeline
    collectedTimeline2
}

PlaygroundPage.current.liveView = UIHostingController(rootView: view.frame(width: 375, height: 600))

sourcePublisher.displayEvents(in: sourceTimeline)
collectedPublisher.displayEvents(in: collectedTimeline)
collectedPublisher2.displayEvents(in: collectedTimeline2)
```

<br />

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%203.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%203.png)

두번째 타임라인이 `collectMaxCount` 상수에 요구된 시간마다 두개의 값을 수집하도록 제한 하는 것을 볼 수 있다.

<br />

# 🐻‍❄️ Holding off on events

User interface 코딩을 하다 보면, text field 를 자주 다루게 된다. Text field 의 콘텐츠를 액션으로 연결 하는 것은 Combine 에서는 일반적인 작업이다. 

예를 들어, 텍스트필드에 입력된 내용과 일치 하는 아이템의 리스트를 리턴하는 검색 URL 요청을 보내고 싶을때.

하지만 당연히 유저가 글자 하나씩 타이핑 할 때마다 요청을 보내고 싶진 않다. 유저가 잠시동안 타이핑을 끝냈을 때를 알아 낼 수 있는 매커니즘이 필요하다.

<br />

### Debounce

```swift
let subject = PassthroughSubject<String, Never>()

let debounced = subject
    .debounce(for: .seconds(1.0), scheduler: DispatchQueue.main)
    .share()

let subjectTimeline = TimelineView(title: "Emitted values")
let debouncedTimeline = TimelineView(title: "Debounced values")

let view = VStack(spacing: 100) {
    subjectTimeline
    debouncedTimeline
}

PlaygroundPage.current.liveView = UIHostingController(rootView: view.frame(width: 375, height: 600))

subject.displayEvents(in: subjectTimeline)
debounced.displayEvents(in: debouncedTimeline)

let subscription1 = subject
    .sink { string in
        print("+\(deltaTime)s : Subject emitted: \(string)")
    }

let subscription2 = debounced
    .sink { string in
        print("+\(deltaTime)s : Debounced emitted \(string)")
    }

subject.feed(with: typingHelloWorld
```

- 결과의 일관성을 보장하기 위해, `share()` 를 사용하여  `debounce` 에 단일 subscription 지점을 생성해 모든 subscribers 에게 같은 시간에 같은 결과를 보여 줄 수 있게 한다.
- `feed(with:)` 메소드는 data set 을 가지고 미리 정의된 시간 간격에 주어진 `subject` 에게 데이터를 전송한다.

<br />

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%204.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%204.png)

11 개의 string 이 `sourcePublisher` 에 푸시 되었다. 두 단어 사이에  유저가 `paused` 한것을 볼 수 있다. 이때가 `debounce` 가 captured input 을 emit 할 때이다.

```swift
+0.0s: Subject emitted: H
+0.1s: Subject emitted: He
+0.2s: Subject emitted: Hel
+0.3s: Subject emitted: Hell
+0.5s: Subject emitted: Hello
+0.6s: Subject emitted: Hello
+1.6s: Debounced emitted: Hello
+2.1s: Subject emitted: Hello W
+2.1s: Subject emitted: Hello Wo
+2.4s: Subject emitted: Hello Wor
+2.4s: Subject emitted: Hello Worl
+2.7s: Subject emitted: Hello World
+3.7s: Debounced emitted: Hello World
```

0.6 초에 유저가 잠시 멈추고 2.1 초에 다시 타이핑을 재시작 했다. `debounce` 가 1초 정도 멈추고 기다리고, 전달 된 최신의 값을 emit 했다.

2.7초에 종료되고 `debounce` 는 1초 뒤인 3.7초에 kick 하였다.

> Note : Publihser 가 마지막 값이 emit 된 직후 바로 완료 되고,  `debounce` 에게 설정된 시간이 지났다면, debounced publihser 의 마지막 값은 절대 볼 수 없을 것이다.

<br />

### Throttle

`debounce` 와 매우 유사하지만, 두개의 operator 가 필요 하다.

```swift
let throttleDelay = 1.0

let subject = PassthroughSubject<String, Never>()

let throttled = subject
    .throttle(for: .seconds(throttleDelay), scheduler: DispatchQueue.main, latest: false)
    .share()

let subjectTimeline = TimelineView(title: "Emitted values")
let throttledTimeline = TimelineView(title: "Throttled valuse")

let view = VStack(spacing: 100) {
    subjectTimeline
    throttledTimeline
}

PlaygroundPage.current.liveView = UIHostingController(rootView: view.frame(width: 300, height: 600))

subject.displayEvents(in: subjectTimeline)
throttled.displayEvents(in: throttledTimeline)

let subscription1 = subject
    .sink { string in
        print("+\(deltaTime)s: Subject emitted: \(string)")
    }

let subscription2 = throttled
    .sink { string in
        print("+\(deltaTime)s: Throttled emitted: \(string)")
    }

subject.feed(with: typingHelloWorld)
```

- `debounce` 처럼 모든 subscriber 가 같은 결과를 같은 시간에 볼 수 있게 보장 하기 위해 `share()` operator 를 사용한다.

<br />

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%205.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%205.png)

`throttle` 에 의해 emit 된 값들은 살짝 타이밍이 다르다.

```swift
+0.0s: Subject emitted: H
+0.0s: Throttled emitted: H
+0.1s: Subject emitted: He
+0.2s: Subject emitted: Hel
+0.3s: Subject emitted: Hell
+0.5s: Subject emitted: Hello
+0.6s: Subject emitted: Hello
+1.0s: Throttled emitted: He
+2.2s: Subject emitted: Hello W
+2.2s: Subject emitted: Hello Wo
+2.2s: Subject emitted: Hello Wor
+2.4s: Subject emitted: Hello Worl
+2.7s: Subject emitted: Hello World
+3.0s: Throttled emitted: Hello W
```

- Subject 가 첫번째 값을 emit 할때, `throttle` 은 즉시 교체한다. 그리고 출력을 throttling 하기 시작한다.
- 1.0 초에, `throttle` 은 "He" 를 emit 한다. 1초 후에 첫번째 값을 보내라고 요청 했던 것을 기억해라.
- 2.2 초에, 타이핑이 다시 시작되었다. 이 시점에 `throttle` 은 아무것도 emit 하지 않았다. 왜냐하면 source publisher 로 부터 새로운 값을 전달 받지 않았기 때문이다.
- 3.0 초에, 타이핑이 완료 된 후에,  `throttle` 은 다시 시작 되었고, 2.2 초 값인 첫번재 값을 다시 출력했다.

`debounce` 와 `throttle` 의 차이

- `debounce` 는 전달 받는 값의 일시 정지를 기다렸다가, 지정된 간격 후에 최신 값을 emit 한다
- `throttle` 은 지정된 간격을 기다리고, 그 간격 동안 전달 받은 첫번째와 최신값 모두 emit 한다. 일시 정지에 대해서는 상관 하지 않는다.

<br />


```swift
let throttled = subject
  .throttle(for: .seconds(throttleDelay), scheduler: DispatchQueue.main, latest: true)
  .share()

+0.0s: Subject emitted: H
+0.0s: Throttled emitted: H
+0.1s: Subject emitted: He
+0.2s: Subject emitted: Hel
+0.3s: Subject emitted: Hell
+0.5s: Subject emitted: Hello
+0.6s: Subject emitted: Hello
+1.0s: Throttled emitted: Hello
+2.0s: Subject emitted: Hello W
+2.3s: Subject emitted: Hello Wo
+2.3s: Subject emitted: Hello Wor
+2.6s: Subject emitted: Hello Worl
+2.6s: Subject emitted: Hello World
+3.0s: Throttled emitted: Hello World
```

`latest` 를 `true`로 바꾸면, throttled 출력은 정확히 1.0초와 3.0초에 시간 창의 최신 값이 아닌 가장 최신 값이 발생한다.

`debounce` 와 비교 하면 출력은 같지만, `debounce` 는 일시 정시에 지연된다.

<br />

# 🐻‍❄️ Timing out

타임아웃 조건과 실제 타이머를 의미적으로 구별을 목적

timeout operator 가 종료되면 publisher 가 완료 되거나 지정된 에러가 emit 된다.

```swift
let subject = PassthroughSubject<Void, Never>()

let timedOutSubject = subject.timeout(.seconds(5), scheduler: DispatchQueue.main)

let timeline = TimelineView(title: "Button taps")

let view = VStack(spacing: 100) {
    Button(action: { subject.send() }) {
        Text("Press me within 5 seconds")
    }
    timeline
}

PlaygroundPage.current.liveView = UIHostingController(rootView: view.frame(width: 375, height: 600))

timedOutSubject.displayEvents(in: timeline)
```

> Note : `Void` 값을 emit 하는 subject 를 사용하는 것은 무언가 발생하지만, 처리할 특정 값이 없다는 것이다. `Subject` 에는 `Output` 타입이 `Void`인,  파라미터를 사용하지 않는 `send()` 함수가 있는 extension 이 있다. 이것은 `subject.send(())` 같은 이상한 구문을 쓰지 않게 해준다.

<br />

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%206.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%206.png)

정확히 액션을 취하기 위해, 실패를 전송하기 위해 `timeout` publisher 를 만들어 보자

```swift
enum TimeoutError: Error {
    case timeOut
}

let subject = PassthroughSubject<Void, TimeoutError>()

let timedOutSubject = subject.timeout(.seconds(5), scheduler: DispatchQueue.main, customError: { .timeOut })

let timeline = TimelineView(title: "Button taps")

let view = VStack(spacing: 100) {
    Button(action: { subject.send() }) {
        Text("Press me within 5 seconds")
    }
    timeline
}

PlaygroundPage.current.liveView = UIHostingController(rootView: view.frame(width: 375, height: 600))

timedOutSubject.displayEvents(in: timeline)
```

이제 버튼을 5초 동안 누르지 않으면, `timeOutSubject` 가 실패를 emit 하는 것을 볼 수 있다.

<br />

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%207.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%207.png)


<br />

# 🐻‍❄️ Measuring time

시간을 조작하지 않고 단지 측정만 한다

`measuerInterval(using:)` operator 는 publisher 가 emit 한 두개의 연속적인 값 사이의 간격 시간을 알기 위해 사용

```swift
let subject = PassthroughSubject<String, Never>()

let measureSubject = subject.measureInterval(using: DispatchQueue.main)

let subjectTimeline = TimelineView(title: "Emitted values")
let measureTImeline = TimelineView(title: "Measured values")

let view = VStack(spacing: 100) {
    subjectTimeline
    measureTImeline
}

PlaygroundPage.current.liveView = UIHostingController.init(rootView: view.frame(width: 375, height: 600))

subject.displayEvents(in: subjectTimeline)
measureSubject.displayEvents(in: measureTImeline)

let subscription1 = subject.sink {
    print("+\(deltaTime)s: Subject emitted: \($0)")
}

let subscription2 = measureSubject.sink {
    print("+\(deltaTime)s: Measure emitted: \($0)")
}

subject.feed(with: typingHelloWorld)

+0.0s: Subject emitted: H
+0.0s: Measure emitted: Stride(magnitude: 16818353)
+0.1s: Subject emitted: He
+0.1s: Measure emitted: Stride(magnitude: 87377323)
+0.2s: Subject emitted: Hel
+0.2s: Measure emitted: Stride(magnitude: 111515697)
+0.3s: Subject emitted: Hell
+0.3s: Measure emitted: Stride(magnitude: 105128640)
+0.5s: Subject emitted: Hello
+0.5s: Measure emitted: Stride(magnitude: 228804831)
+0.6s: Subject emitted: Hello
+0.6s: Measure emitted: Stride(magnitude: 104349343)
+2.2s: Subject emitted: Hello W
+2.2s: Measure emitted: Stride(magnitude: 1533804859)
+2.2s: Subject emitted: Hello Wo
+2.2s: Measure emitted: Stride(magnitude: 154602)
+2.4s: Subject emitted: Hello Wor
+2.4s: Measure emitted: Stride(magnitude: 228888306)
+2.4s: Subject emitted: Hello Worl
+2.4s: Measure emitted: Stride(magnitude: 138241)
+2.7s: Subject emitted: Hello World
+2.7s: Measure emitted: Stride(magnitude: 333195273)
```

`measureInterval` 이 emit 하는 값의 타입은 "제공된 스케쥴러의 시간 간격" 이다.  `DispatchQueue` 의 경우 `TimeInterval` 은 “A `DispatchTimeInterval` created with the value of this type in nanoseconds.” 라고 정의 되어 있다.

<br />

위에 결과는 나노세컨드동안 전달 받은 연속적인 값 사이의 개수이다.  값을 출력 하기 위해 코드를 수정하자

```swift
let subject = PassthroughSubject<String, Never>()
let measureSubject2 = subject.measureInterval(using: RunLoop.main)

let measureSubject = subject.measureInterval(using: DispatchQueue.main)

let subjectTimeline = TimelineView(title: "Emitted values")
let measureTImeline = TimelineView(title: "Measured values")

let view = VStack(spacing: 100) {
    subjectTimeline
    measureTImeline
}

PlaygroundPage.current.liveView = UIHostingController.init(rootView: view.frame(width: 375, height: 600))

subject.displayEvents(in: subjectTimeline)
measureSubject.displayEvents(in: measureTImeline)

let subscription1 = subject.sink {
    print("+\(deltaTime)s: Subject emitted: \($0)")
}

let subscription2 = measureSubject.sink {
 print("+\(deltaTime)s: Measure emitted: \(Double($0.magnitude) / 1_000_000_000.0)")
}
let subscription3 = measureSubject2.sink {
 print("+\(deltaTime)s: Measure2 emitted: \($0)")
}

subject.feed(with: typingHelloWorld)

+0.0s: Subject emitted: H
+0.0s: Measure emitted: 0.016503769
+0.0s: Measure2 emitted: Stride(magnitude: 0.015684008598327637)
+0.1s: Subject emitted: He
+0.1s: Measure emitted: 0.087991755
+0.1s: Measure2 emitted: Stride(magnitude: 0.08793699741363525)
+0.2s: Subject emitted: Hel
+0.2s: Measure emitted: 0.115842671
+0.2s: Measure2 emitted: Stride(magnitude: 0.11583995819091797)
```

이제 스케줄러가 우리가 원하던 대로 측정하는데에 사용 된다. 일반적으로 `DispatchQueue` 를 고수하는 것이 좋다.

<br />

# 🔑 Key points

- 비동기 이벤트에 대한 Combine 의 처리는 스스로 시간을 조작하기로 확장된다.
- 시간 여행 옵션을 제공하지는 않지만,  프레임워크는 단지 개별 이벤트를 처리하는 것 대신에, 장기간동안 작업을 추출할수 있게 해주는 operator 를 가지고 있다.
- `delay` operator 로 시간을 옮길 수 있다.
- `collect` 를 사용해서 시간의 지남에 따른 값의 흐름을 조절 할 수 있다.
- `debounce` 와 `throttle` 을 사용하면 시간에 따라 개별 값을 쉽게 선택 할 수 있다.
- `timeout` 은 시간이 다 지나버리지 않게 한다.
- `measureInterval` 로 시간을 측정 할 수 있다.
