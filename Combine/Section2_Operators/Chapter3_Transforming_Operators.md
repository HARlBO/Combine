# Chapter 3 : Transforming Operators

### 🐻‍❄️ Operators are publishers

- Publisher 으로 부터 온 값을 조작하는 메소드
- Operator은 publihser를 리턴함

일반적으로, publisher은 upstream 값을 전달 받고, 데이터를 조작하고, downstream 으로 데이터를 전송한다. Operator의 목적이 에러를 핸들링 하는 것이지만, upstream publisher로 부터 에러를 받는다면, 해당 에러를 downstream 으로 publish 할 것이다.  

</br>

## 🐻‍❄️ Collecting values

Publihser는 단일값 또는 value의 collection 을 emit 할 수 있다. 주로 collection을 다루고 싶을 것이다.

### `collect()`

- `collect` operator 는 publihser에서 온 단일 값 stream 을 그 값들의 array로 바꾸는 편리한 방법을 제공한다

**Marble diagrams**

![Chapter3_Transforming_Operators/Untitled.png](/Combine/Section2_Operators/images/Untitled.png)

1. 가장 윗 줄 - upstream publihser
2. 박스 - operator
3. 아랫줄 
- subscriber
- 더 명확히는 upstream pulisher로 부터 operator가 값을 조작 한 후에 subscriber가 받을 값
- upstream publihser로 output 부터 받고, operation을 수행하고, downstream으로 값을 보내는 oprator

→ 위 marble diagram 에서 묘사하는 것 처럼, `collect` 는 단일 값의 stream을 upstream publihser 가 완료 될 때, 해당 값들의 array로  buffer 할 것이다. 그 후 array를 downsteram으로 emit 할 것이다.

</br>

```swift
["A", "B", "C", "D", "E"].publisher
  .collect()
  .sink(receiveCompletion: { print($0) },
        receiveValue: { print($0) })
  .store(in: &subscriptions)

——— Example of: collect ———
["A", "B", "C", "D", "E"]
finished
```

`collect`를 추가하면, sink 는 하나의 collection을 emit 받는다.

</br>

> Note: 특정 count 나 제한이 없는 `collect()` 나 다른 버퍼링을 사용하는 operator 를 사용할 때는, 전달 받은 값들은 메모리를 무한정으로 사용 할 수 있기 때문에 주의해야함.

</br>

`collect()` 를 `collect(2)` 이런 식으로 값의 개수를 특정 할 수 있다.

</br>

```swift
위 코드의 collect() 를 .collect(2)

-—— Example of: collect ———
["A", "B"]
["C", "D"]
["E"]
finished
```

마지막 값 `E` 는 array 로 emit 되었다.  `collect` 의 예정된 버퍼가 다 채워지기 전에 upstream publihser 가 완료 되어서,  남은 array 를 전달 하였다.

</br>

## 🐻‍❄️ Mapping values

Combine은 값을 수집 하는 것 외에도, 값들을 변형 시킬 수 있는 mapping operator들을 제공한다.

### `map(_:)`

Swift standard `map` 과 비슷 하지만, publisher에서 emit 된 값들에서만 작동 한다는 점이 다르다.

![Chapter3_Transforming_Operators/Untitled%201.png](/Combine/Section2_Operators/images/Untitled1.png)

marble diagram에서 `map` 은 각각의 값에 2를 곱하는 클로저를 갖는다.

</br>

```swift
example(of: "map") {
    let formatter = NumberFormatter()
    formatter.numberStyle = .spellOut
    
    [123, 4, 56].publisher
        .map {
            formatter.string(for: NSNumber(integerLiteral: $0)) ?? ""
        }
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: map ———
one hundred twenty-three
four
fifty-six
```
</br>

**Map key paths**

`map` operators 에는 key path를 이용해 1~3 개의 프로퍼티를 map 할 수 있는 세가지 버전이 있다.

- `map<T>(_:)`
- `map<T0, T1>(_:_:)`
- `map<T0, T1, T2>(_:_:_:)`

`T` 는 주어진 key path 에서의 값 타입을 대표한다.

`Coordinate` 타입과 `quadrantOf(x:y:)` 메소드

- Sources/SupportCode.swift 에 정의 되어 있음
- `Coordinate` 에는 `x` 와 `y` 두가지 프로퍼티가 있다
- `quadrantOf(x:y:)` 는 `x` 와 `y` 두가지 값을 파라미터로 받아 사분면을 나타내는 String 을 리턴한다.

</br>

```swift
example(of: "map key paths") {
  let publisher = PassthroughSubject<Coordinate, Never>()

	publisher
    .map(\.x, \.y)
    .sink(receiveValue: { x, y in
      print(
        "The coordinate at (\(x), \(y)) is in quadrant",
        quadrantOf(x: x, y: y)
	) })
    .store(in: &subscriptions)

  publisher.send(Coordinate(x: 10, y: -8))
  publisher.send(Coordinate(x: 0, y: 5))
}

——— Example of: map key paths ———
The coordinate at (10, -8) is in quadrant 4
The coordinate at (0, 5) is in quadrant boundary
```

</br>


### `tryMap(_:)`

`map` 을 포함한 몇몇의 operator 들에는, 에러를 발생시키는 closure를 갖는 `try` operator 를 포함 한다.

에러를 던지면, 그 에러를 downstream 에 emit 한다.

</br>

```swift
example(of: "tryMap") {
    Just("Directory name that does not exist")
        .tryMap{ try FileManager.default.contentsOfDirectory(atPath: $0) }
        .sink(receiveCompletion: { print($0) },
              receiveValue:  { print($0) })
        .store(in: &subscriptions)
}

——— Example of: tryMap ———
  failure(..."The folder “Directory name that does not exist” doesn't exist."...)
```

</br>

## 🐻‍❄️ Flattening publishers

### `flapMap(maxPublihsers:_:)`

- multiple upstream publihser 를 single downsteram publisher 로 flatten 하는데 사용
- 더 정확히는, 해당 publihser 의 emission 들을 flatten 함
- `flapMap` 에 의해 리턴되는 publisher는 종종 upstream publihser 와 다른 타입이다
- 주로 하나의 publihser에서 emit 된 요소들을 publihser를 리턴하는 메소드에 전달 하고 싶을 때, 그리고 궁극적으로 다른 publisher 로 부터 emit 된 요소들을 subscribe 하고 싶을때 사용한다

</br>

```swift
example(of: "flatMap") {
    func decode(_ codes: [Int]) -> AnyPublisher<String, Never> {
        Just( codes
                .compactMap { code in
                    guard (32...255).contains(code) else { return nil }
                    return String(UnicodeScalar(code) ?? " ")
                }
                .joined() )
            .eraseToAnyPublisher()
    }
    
    [72, 101, 108, 108, 111, 44, 32, 87, 111, 114, 108, 100, 33]
        .publisher
        .collect()
        .flatMap(decode)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: flatMap ———
Hello, World!
```

</br>

- `flatMap` 은 수신된 모든 publihser의 output 을 단일 publihser로 flatten 한다.
- downstrem에서 emit 하는 단일 publihser를 업데이트하기 위해 전송되는 만큼의 많은 publihser 들을 buffer 하여 메모리 문제를 발생 할 수 있다.

![Chapter3_Transforming_Operators/Untitled%202.png](/Combine/Section2_Operators/images/Untitled2.png)

1. 다이어그램에서, `flatMap` 은 P1, P2, P3 3 개의 publihser를 받는다.
2. 각각의 publisher 들은 `value` 프로퍼티를 갖고 이것 역시 publihser이다.
3. `flatMap` 은 P1 과 P2 로 부터 받은 `value` publisher의 값을 emit 한다. 하지만 `maxPublishers` 가 2로 세팅 되어 있어 P3는 무시한다.

</br>

## 🐻‍❄️ Replacing upstream output

Combine 에는 항상 값을 전달 받고 싶을 때 쓰는 operator 가 있다.

### `replaceNil(with:)`

![Chapter3_Transforming_Operators/Untitled%203.png](/Combine/Section2_Operators/images/Untitled3.png)

- optional 값을 전달 받고 `nil`을 특정 값으로 교체 해준다.

</br>

```swift
example(of: "replaceNil") {
    ["A", nil ,"C"].publisher
        .eraseToAnyPublisher()
        .replaceNil(with: "-")
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: replaceNil ———
A
-
C
```

</br>

> Note: `replaceNil(with:)` 의  잘못된 사용으로 과부화?로 인한 이슈가 있다. 타입이 완전히 언래핑 되지 않고 Optional<String> 같이 남아 있게 된다. `earseToAnyPublisher()` 가 해당 버그에 사용 된다. 
Swift forums : [https://bit.ly/30M5Qv7](https://bit.ly/30M5Qv7)
	
</br>

nil 병합 연산자인 `??` 을 사용하는 것과 `replaceNil`  를 사용하는 것 사이에는 미묘하지만 중요한 차이가 있다.

- `resplaceNil` 은 아니지만 `??` 연산자는 여전히 `nil` 의 결과를 가질 수 있다.
- `replaceNil(with: "-" as String?)` 로 바꾸면 optional은 반드시 언래핑 되어야 한다는 에러가 생길 것이다.

</br>

### `replaceEmpty(with:)`

- Publihser 가 값은 emit 하지 않은채로 완료 되었을 때 사용

![Chapter3_Transforming_Operators/Untitled%204.png](/Combine/Section2_Operators/images/Untitled4.png)

- Publihser 가 값은 emit 하지 않은채로 완료 되었을 때, `reppaceEmpty(with:)` operator 가 값을 넣어 준 뒤 downstream 에 publish 한다.

</br>

```swift
example(of: "replaceEmpty") {
    let empty = Empty<Int, Never>()
    
    empty
        .sink(receiveCompletion: { print($0) },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: replaceEmpty ———
finished
```

</br>

- `Empty` publiher 타입은 즉시 `.finihsed` completion 이벤트를 emit 하는 publihser를 생성하는데 사용 된다.
- `completeImmediately` 파라미터에 `false` 를 전달 하면 아무것도 emit 하지 않게 할 수 있다.
- 이 publisher는 데모나 테스트 또는 subscriber 에게 태스크의 단일 completion 이벤트 시그널만 주고 싶을 때 유용하다.

</br>

```swift
example(of: "replaceEmpty") {
    let empty = Empty<Int, Never>()
    
    empty
        .replaceEmpty(with: 1)
        .sink(receiveCompletion: { print($0) },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
}

1
finished
```

`sink` 전에 `replaceEmpty(with: 1)` 을 추가 하면, completion 전에 1을 리턴받는다.

</br> 

## 🐻‍❄️ Incrementally transforming output

Swift standard 에서 볼 수 있는 고차 함수 와 유사한 operator 를 보았다. 하지만 Combine은 upstream publihser 로 부터 전달 받은 값들을 조작 하는 더 많은 트릭이 있다.

</br>

### `scan(_:_:)`

- upstream publisher 에서 emit 된 현재 값을 clousre 로 제공하고, 해당 클로저에 의해 리턴 된 마지막 값을 제공한다.

![Chapter3_Transforming_Operators/Untitled%205.png](/Combine/Section2_Operators/images/Untitled5.png)

1. `scan` 은 시작 값 0 을 저장 하면서 시작한다.
2. Publihser로 부터 각각의 값을 전달 받으면, 이전에 저장 해놓은 값에 더해서 저장하고, 결과를 emit 한다.

</br>

```swift
example(of: "scan") {
    var dailyGainLoss: Int { .random(in: -10...10) }
    
    let august2019 = (0..<22)
        .map { _ in dailyGainLoss }
        .publisher
    
    august2019
        .scan(50) { latest, current in
            max(0, latest + current)
        }
        .sink(receiveValue: { _ in})
        .store(in: &subscriptions)
}
```

</br>

- 에러를 던지는 `tryScan` operator 또한 있다. 클로저가 에러를 throw 하면, `tryScan` 은 해당 에러로 fail 된다.

</br> 

## 🔑 Key points

- Publihser의 output 대해 작업을 수행하는 메소드를 Operator 라 부른다.
- Operator 도 역시 Publihser 이다.
- Transforming Operator 는 upstream publihser 를 downstream 사용에 적합한 ouput 으로 변환 한다.
- Marble diagram 은 Combine operator 가 어떻게 동작하는지 시각화 하는데 좋은 방법이다.
- `collect` 나 `flatMap` 같이 값을 buffer 하는 operator 를 사용할 때에는 메모리 문제에 유의해야 한다.
- Swift standard library 와 이름이 비슷 한 Combine operator는 비슷하지만 완전히 다른 방식으로 동작 한다.
- 하나의 Subscription 에서 다수의 operator를 체이닝 할 수 있다.
