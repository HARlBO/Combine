# Chapter 19 : Testing Combine Code

- 테스트를 작성하는 것은 새로운 기능을 개발할 때, 의도한 기능을 보장할 수 있는 좋은 방법이며, 특히 이전에 잘 동작한 코드가 최신 작업에서도 정상적으로 동작하는지 보장해준다

## Testing Combine operators

- **Given** - 조건 (condition)
- **When** - 실행 될 행동 (action is performed)
- **Then** - 예상되는 결과 (expected result occurs)

`setup()` 각 테스트 메소드가 실행되기 전 

`tearDown()` 각 테스트 메소드가 종료된 후

```swift
var subscriptions = Set<AnyCancellable>()

override func tearDown() {
	subscriptions = []
}
```

### Testing `collect()`

- upstream publisher 에서 emit 된 값을 buffer 하고 complete 까지 기다린 후 값을 array 로 바꾸어서 emit

![/Combine/Section4_Advanced_Combine/images/Chapter19_Testing_Combine_Code/Untitled.png](/Combine/Section4_Advanced_Combine/images/Chapter19_Testing_Combine_Code/Untitled.png)

```swift
func test_collect() {
	// Given
	let values = [0, 1, 2]
	let publisher = values.publisher
        
	// When
	publisher
		.collect()
		.sink(receiveValue: {
		// Then
		XCTAssert(
			$0 == values,
			"Result was expected to be \(values) but was \($0)"
			)
		})
		.store(in: &subscriptions)
}
```

### Testing `flatMap(maxPublishers:)`

- 다수의 upstream publisher 를 하나의 downstream publisher 로 flatten

![/Combine/Section4_Advanced_Combine/images/Chapter19_Testing_Combine_Code/Untitled%201.png](/Combine/Section4_Advanced_Combine/images/Chapter19_Testing_Combine_Code/Untitled%201.png)

```swift
func test_flatMapWithMax2Publishers() {
        // Given
        let intSubject1 = PassthroughSubject<Int, Never>()
        let intSubject2 = PassthroughSubject<Int, Never>()
        let intSubject3 = PassthroughSubject<Int, Never>()
        
        let publisher = CurrentValueSubject<PassthroughSubject<Int, Never>, Never>(intSubject1)
                
        let expected = [1, 2, 4]
        var results = [Int]()
        
        publisher
            .flatMap(maxPublishers: .max(2)) { $0 }
            .sink(receiveValue: {
                results.append($0)
            })
            .store(in: &subscriptions)
        
        // When
        intSubject1.send(1)
        
        publisher.send(intSubject2)
        intSubject2.send(2)
        
        publisher.send(intSubject3)
        intSubject3.send(3)
        intSubject2.send(4)
        
        publisher.send(completion: .finished)
        
        // Then
        XCTAssert(
        results == expected,
            "Result exptected to be \(expected) but were \(results)"
        )
    }
```

- 이전 reactive programming 에서는 테스트 시간을 기반으로 하여 세부적으로 제어할 수 있는 가상 시간 스케줄러인 test scheduler 를 작성하는 것이 익숙하다
- Combine 은 time scheduler 를 제공하지 않는다
(오픈소스 Entwine([https://github.com/tcldr/Entwine](https://github.com/tcldr/Entwine)) 로 사용 가능)
- Apple's native Combine framework 사용에 초점을 맞춰, `XCTest` 사용

### Testing `publish(every:on:in:)`

- `Timer` publisher
- `XCTest` 의 expectation API 를 사용하여 비동기 연산이 완료될 때까지 대기

```swift
func test_timerPublish() {
        // Given
        func normalized(_ ti: TimeInterval) -> TimeInterval {
            return Double(round(ti * 10) / 10)
        }
        
        let now = Date().timeIntervalSinceReferenceDate
        
        let expectation = self.expectation(description: #function)
        
        let expected = [0.5, 1, 1.5]
        var results = [TimeInterval]()
        
        let publisher = Timer
            .publish(every: 0.5, on: .main, in: .common)
            .autoconnect()
            .prefix(3)
        
        // When
        publisher
            .sink(
                receiveCompletion: { _ in expectation.fulfill() },
                receiveValue: {
                    results.append(
                        normalized($0.timeIntervalSinceReferenceDate - now)
                    )
                }
            )
            .store(in : &subscriptions)
        
        // Then
        waitForExpectations(timeout: 2, handler: nil)
        
        XCTAssert(
            results == expected,
            "Results expected to be \(expected) but were \(results)"
        )
    }
```

### Testing `shareReplay(capacity:)`

- Publisher 의 output 을 여러 subscriber 들과 공유하면서, 새 subscriber 에게 마지막 N 값의 버퍼를 재실행

```swift
func test_shareRelay() {
        // Given
        let subject = PassthroughSubject<Int, Never>()
        let publisher = subject.shareReplay(capacity: 2)
        
        let expected = [0, 1, 2, 1, 2, 3, 3]
        var results = [Int]()
        
        // When
        publisher
            .sink(receiveValue: { results.append($0) })
            .store(in: &subscriptions)
        
        subject.send(0)
        subject.send(1)
        subject.send(2)
        
        publisher
            .sink(receiveValue: { results.append($0) })
            .store(in: &subscriptions)
        
        subject.send(3)
        
        XCTAssert(
            results == expected,
            "Results expected to be \(expected) but was \(results)"
        )
    }
```

## Testing production code

### Issue 1 : Incorrect name displayed

- Action: 앱을 실행한다
- Expected: name label 이 aqua 로 표시 되어야 한다
- Actual: name label 이 Optional(ColorCalc.ColorNam....) 로 보인다

```swift
func test_correctNameReceive() {
        // Given
        let expected = "rwGreen 66%"
        var result = ""

        viewModel.$name
            .sink(receiveValue: { result = $0} )
            .store(in: &subscriptions)

        // When
        viewModel.hexText = "006636AA"

        // Then
        XCTAssert(
            result == expected,
            "Name expected to be \(expected) but was \(result)"
        )
    }
```

### Issue 2 : Tapping backspace deletes two characters

- Action: **←** 버튼을 탭한다
- Expected: hex display 에서 마지막 문자가 제거된다
- Actual: 마지막 두개의 문자가 제거된다

```swift
func test_processBackspaceDeleteLastCharacter() {
        // Given
        let expected = "#0080F"
        var result = ""

        viewModel.$hexText
            .dropFirst()
            .sink(receiveValue: { result = $0 })
            .store(in: &subscriptions)

        // When
        viewModel.process(CalculatorViewModel.Constant.backspace)

        // Then
        XCTAssert(
            result == expected,
            "Hex was expected to be \(expected) but was \(result)"
        )
    }
```

### Issue 3 : Incorrect background color

- Action: **← 버튼을 탭한다**
- Expected: background 가 white 색으로 바뀐다
- Actual: background 가 red 로 바뀐다

```swift
func test_correctColorReceived() {
        // Given
        let expected = Color(hex: ColorName.rwGreen.rawValue)
        var result: Color = .clear
        
        viewModel.$color
            .sink(receiveValue: { result = $0 })
            .store(in: &subscriptions)
        
        // When
        viewModel.hexText = ColorName.rwGreen.rawValue
        
        // Then
        XCTAssert(
            result == expected,
            "Color expected to \(expected) but was \(result)"
        )
    }
```

```swift
func test_processBackspaceReceivesCorrectColor() {
        // Given
        let expected = Color.white
        var result = Color.clear

        viewModel.$color
            .sink(receiveValue: { result = $0 })
            .store(in: &subscriptions)

        // When
        viewModel.process(CalculatorViewModel.Constant.backspace)

        // Then
        XCTAssert(
            result == expected,
            "Hex was expected to be \(expected) but was \(result)"
        )
}
```

### Testing for bad input

잘못된 hex 값에 대한 데이터 방어

```swift
func test_whiteColorRecivedForBadData() {
        // Given
        let expected = Color.white
        var result = Color.clear

        viewModel.$color
            .sink(receiveValue: { result = $0 })
            .store(in: &subscriptions)

        // When
        viewModel.hexText = "abc"

        // Then
        XCTAssert(
            result == expected,
            "Color expected to be \(expected) but was \(result)"
        )
   }
```

## Challenges

### Challenge 1: Resolve Issue 4: Tapping clear does not clear hex display

- Action: ⊗ 버튼을 탭한다
- Expected: hex 값이 # 로 표시된다
- Actual: hex 값이 변하지 않는다

```swift
func test_processClearSetsHexToHashTag() {
        // Given
        let expected = "#"
        var result = ""

        viewModel.$hexText
            .dropFirst()
            .sink(receiveValue: { result = $0 })
            .store(in: &subscriptions)

        // When
        viewModel.process(CalculatorViewModel.Constant.clear)

        // Then
        XCTAssert(
            result == expected,
            "Hex was expected to be \(expected) but was \(result)"
        )
    }
```

### Challenge 2: Resolve Issue5: Incorrect red-green-blue-opacity display for entered hex

- Action: hex 값 006636 를 입력한다
- Expected:  red-green-blue-opacity 가 0, 102, 54, 255 로 표시된다
- Actual: red-green-blue-opacity 가 0, 62, 32, 155 로 표시된다

```swift
func test_correctRGBOTextReceived() {
        // Given
        let expected = "0, 102, 54, 170"
        var result = ""
        
        viewModel.$rgboText
            .sink(receiveValue: { result = $0 })
            .store(in: &subscriptions)
        
        // When
        viewModel.hexText = "#006636AA"
        
        // Then
        XCTAssert(
            result == expected,
            "RGBO text expected to be \(expected) but was \(result)"
        )
    }
```

## 🔑 Key points

- Unit test 는 초기 개발 시 코드가 예상대로 동작하고, 향후에 퇴화(regression) 되지 않는데에 도움이 된다
- Unit test 할 비지니스 로직과 UI test 할 프레젠테이션 로직을 잘 분리하도록 구성해야 한다. MVVM 은 이 목적에 적합한 패턴이다
- Given-When-Then 과 같은 패턴을 사용하면 테스트 코드를 구성하는데 도움이 된다
- expectaion 을 사용하여 시간 기반 비동기 Combine 코드를 테스트 할 수 있다
- 긍정적, 부정적 조건 모두 테스트 하는 것이 중요하다
