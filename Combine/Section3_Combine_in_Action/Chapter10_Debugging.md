# Chapter 10: Debugging

## 🐻‍❄️ Printing events

`print(_:to:)` operator 

- publisher 를 통해 처리 되고 있는지 확실하지 않을 때 가장 먼저 사용
- 어떤 일이 일어나고 있는지에 대한 많은 정보를 출력한는 passthrough publisher

 

```swift
let subscription = (1...3).publisher 
	.print("publisher")
	.sink { _ in }

publisher: receive subscription: (1...3)
publisher: request unlimited
publisher: receive value: (1)
publisher: receive value: (2)
publisher: receive value: (3)
publisher: receive finished
```

- Subscription 을 전달 받았을 때 출력하고 upstream publisher 에 대한 설명을 보여준다.
- Subscriber의 요구 요청을 출력해서 몇개의 아이템이 요청 되었는지 보여준다.
- Upstream publisher 가 emit 한 모든 값을 출력한다
- 마지막으로, completion 이벤트를 출력한다

<br />


추가적으로 `TextOutputSream` 객체를 갖는 파라미터가 있다.

- 직접 string 을 logger 에 출력 하게 할 수 있다.
- 현재 날짜나 시간 같은 정보를 로그에 추가 할 수 있다.

```swift
class TimeLogger: TextOutputSream {
	private var previous = Date()
	private let formatter = NSNumberFormatter()

	init() {
		formatter.maximumFractionDigits = 5
		formatter.minimumFractionDigits = 5

	func write(_ string: String) {
		let trimmed = string.trimmingCharacters(in: .whitespacesAndNewlines)
		guard !trimmed.isEmpty else { return }
		let now = Date()
		print("+\(formatter.string(for: now.timeIntervalSince(previous))!)s: \(string)")
		previous = now
	}
}

let subscription = (1...3).publisher
  .print("publisher", to: TimeLogger())
  .sink { _ in }

+0.00111s: publisher: receive subscription: (1...3)
+0.03485s: publisher: request unlimited
+0.00035s: publisher: receive value: (1)
+0.00025s: publisher: receive value: (2)
+0.00027s: publisher: receive value: (3)
+0.00024s: publisher: receive finished
```

<br />


## 🐻‍❄️ Acting on events - performing side effects

### The

`handleEvents(receiveSubscription:receiveOutput:receiveCompletion:receiveCancel:r eceiveRequest:)`

publisher 의 생명 주기 내에서 이벤트를 intercept 하고 각각의 단계에 액션을 취할 수 있다. 

```swift
let request = URLSession.shared
	.dataTaskPublisher(for: URL(string: "https://www.raywenderlich.com/")!)

let subscription = requset
	.handleEvents(receiveSubscription: { _ in
	  print("Network request will start")
	}, receiveOutput: { _ in
	  print("Network request data received")
	}, receiveCancel: {
	  print("Network request cancelled")
	})
	.sink(receiveCompletion: { completion in
		print("Sink received completion: \(completion)")
	}) { data, _ in
		print("Sink received data: \(data)")
	}

Network request will start
Network request cancelled
Sink received data: 153253 bytes
Sink received completion: finished
```

<br />


## 🐻‍❄️ Using the debugger as a last resort

`breakpointOnError()` 

- Upstream publisher 에서 에러를 emit 했을때, Xcode 는 디버거에 break 하고 스택을 볼 수 있도록 해주고 어디서 왜 에러가 나온건지 찾게 해준다.

`breakpoint(receiveSubscription:receiveOutput:receiveCompletion:)`

- 다양한 이벤트에 intercept 하고 케이스 마다 디버거를 멈추고 싶은지 결정한다.

```swift
.breakpoint(receiveOutput: { value in
	return value > 10 && value < 15
}
```

조건적으로 구독과 completion time 을 break 할 수 있지만,  `handleEvents` operator 같은 취소는 intercept 할 수 없다.

> Note : Playground 에서는 breakpoint publisher 가 동작하지 않는다. 실행이 interrupt 되었다는 에러는 볼 수 있지만, 디버거에 들어가진 않는다.

<br />


## 🔑 Key points

- `print` operator 로 publisher 의 생명주기를 추적 할 수 있다.
- `TextOutputStream` 을 생성해서 결과 string 을 커스텀 할 수 있다.
- `handleEvents` operator 는 생명주기 이벤트를 intercept 하고 액션을 수행한다.
- `breakpointOnError` 와 `breakpoint` operator 는 특정 이벤트를 break 한다.
