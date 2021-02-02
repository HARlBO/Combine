# Chapter 16: Error Handling

Publisher 의 `Failure` 의 역할에 대해서 알아보자!

![/Combine/Section4_Advanced_Combine/images/Section4_Advanced_Combine/Untitled.png](/Combine/Section4_Advanced_Combine/images/Section4_Advanced_Combine/Untitled.png)

### 🐻‍❄️ Never

- `Never` 타입의 Failure 는 publisher 가 절대 fail 하지 않는다는걸 나타낸다
- Publisher 의 값 소비에만 집중 할 수 있게 한다

![/Combine/Section4_Advanced_Combine/images/Section4_Advanced_Combine/Untitled%201.png](/Combine/Section4_Advanced_Combine/images/Section4_Advanced_Combine/Untitled%201.png)

`Just` 의 failure type alias 를 보면

```swift
public typealias Failure = Never
```

Combine 의 `Never` 에 대한 no-failure 보장은 이론뿐만 아니라 프레임워크와 다양한 API 에 깊이 뿌리를 두고 있다.

Publisher 가 절대 실패하지 않을 경우에만 사용할 수 있는 여러 연산자를 제공한다.

→ `sink(receiveValue:)`

```swift
example(of: "Never sink") {
    Just("Hello")
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: Never sink ———
Hello
```

`sink` 를 오버로드한 `sink(receiveValue:)` 는 publisher 의 완료 이벤트를 무시하고 emit 된 값만을 다룬다.

<br />


### `setFailureType`

- Publisher 의 failure 가 `Never` 타입 일때만 사용가능한 operator

```swift
enum MyError: Error {
  case ohNo
}

example(of: "setFailureType") {
  Just("Hello")
    .setFailureType(to: MyError.self)
    .sink(receiveCompletion: { completion in
        switch completion {
        case .failure(.ohNo):
            print("Finished with Oh No!")
        case .finished:
            print("Finished successfully!")
        }
    }, receiveValue: { value in
        print("Got value: \(value)")
    })
    .store(in: &subscriptions)
}

——— Example of: setFailureType ———
Got value: Hello
Finished successfully!
```

`setFailureType` 는 타입-시스템 정의에만 영향을 미칠뿐이다. Original Publisher 가 `Just` 기 때문에, 에러가 발생하지 않는다.

<br />


### `assign(to:on:)`

- 실패하지 않는 publisher 에서만 작동
- 제공된 key path 에 에러를 전송하면 처리되지 않은 에러 혹은 정의되지 않는 동작이 발생한다.

```swift
example(of: "assign") {
  class Person {
    let id = UUID()
    var name = "Unknown"
  }

  let person = Person()
  print("1", person.name)

  Just("Shai")
    .handleEvents(
      receiveCompletion: { _ in print("2", person.name) }
    )
    .assign(to: \.name, on: person)
    .store(in: &subscriptions)
}

——— Example of: assign ———
1 Unknown
2 Shai
```

`Just` 가 절대 실패하지 않기때문에, `Just` 가 값은 emit 하자마자 `assign` 이 person 의 이름을 업데이트 했다. 

non-Never failure 타입은 어떨까?

```swift
.setFailureType(to: Error.self)

→ referencing instance method 'assign(to:on:)' on 'Publisher' requires the types 'Error' and `Never` be equivalent
```

코드를 추가하면, Swift 표준 에러로 failure 타입을 설정 한것이다.

`Publisher<String, Never>` 가 아닌 `Publisher<String, Error>`

<br />


### `assertNotFailure`

- 개발 중에 자신을 보호하여 publisher 가 실패 이벤트로 완료할 수 없음을 확인 하는 경우 유용
- Upstream 에서 emit 되는 실패 이벤트가 발생하는 것을 방지하지 않는다
- 그러나 에러를 감지하면, `fatalError` 와 함께 크래시가 난다

```swift
example(of: "assertNoFailure") {
  Just("Hello")
    .setFailureType(to: MyError.self)
    .tryMap { _ in throw MyError.ohNo }
    .assertNoFailure()
    .sink(receiveValue: { print("Got value: \($0) ")})
    .store(in: &subscriptions)
}

→ error: Execution was interrupted, reason: EXC_BAD_INSTRUCTION (code=EXC_I386_INVOP, subcode=0x0)
```

- `tryMap` 이 `Hello` 다 downstream으로 푸시될 때 에러를 발생시킨다.
- Publisher 에 실패가 발생되어 크래시가 난다.

<br />


## 🐻‍❄️ Dealing with failure

### try* operators

Combine 은 에러를 발생시킬 수 있는 operator 와 그렇지 않은 operator 사이에 구분을 제공한다.

```swift
example(of: "tryMap") {
  enum NameError: Error {
    case tooShort(String)
    case unknown
}

  let names = ["Scott", "Marin", "Shai", "Florent"].publisher

	names
    .map { value in
      return value.count
    }
    .sink(receiveCompletion: { print("Completed with \($0)") },
			receiveValue: { print("Got value: \($0)") })
}

——— Example of: tryMap ———
Got value: 5
Got value: 5
Got value: 4
Got value: 7
Completed with finished
```

`map` 은 에러를 던질수 없는 non-throwing 메소드이다. `tryMap` 을 사용하면 된다

```swift
.tryMap { value -> Int in
  // 1
  let length = value.count
// 2
  guard length >= 5 else {
    throw NameError.tooShort(value)
}
// 3
  return value.count
}

——— Example of: tryMap ———
Got value: 5
Got value: 5
Completed with failure(...NameError.tooShort("Shai"))
```

<br />


### Mapping errors

- `map` 과 `tryMap` 은 에러를 던질수 있는지의 차이이다
- `map` 은 존재하는 failure type 을 가지고 publisher 의 값만을 조작한다
- `tryMap` 은 그렇지 않고, Swift `Error` 타입을 지운다. 
`try` prefix 를 가진 operator 들이 다 동일하다.

<br />


```swift
example(of: "map vs tryMap") {
  enum NameError: Error {
    case tooShort(String)
    case unknown
}

  Just("Hello")
    .setFailureType(to: NameError.self)
    .map { $0 + " World!" } // 4
    .sink(
      receiveCompletion: { completion in
        switch completion {
        case .finished:
          print("Done!")
        case .failure(.tooShort(let name)):
          print("\(name) is too short!")
        case .failure(.unknown):
          print("An unknown name error occurred")
        }
			},
      receiveValue: { print("Got value \($0)") }
    )
	.store(in: &subscriptions)
}

——— Example of: map vs tryMap ———
Got value Hello World!
Done!
```

`Completion` 의 failure type 은 `NameError` 이고, `setFailureType` operator 가 이것을 타겟팅 할 수 있게 해준다.

`map` 을 `tryMap` 으로 바꾸면

`tryMap` 이 에러타입을 지우고 `Swift.Error` 타입으로 교체한다.

일반 에러를 특정 에러타입으로 캐스팅 하는 것은 차선책이 되어야 한다 → `mapError` 

<br />


`tryMap` 을 `mapError` 로 바꾸면

```swift
.mapError { $0 as? NameError ?? .unknown }

——— Example of: map vs tryMap ———
Got value Hello World!
Done!
```

- `mapError` 는 upstream publisher 로부터 던져진 모든 에러를 받고 원하는 에러에 매핑 할 수 있게 한다
- 캐스팅이 실패할 수 이씩 때문에 이 경우에는 fallback 에러를 제공해야 한다.
- 이렇게 하면 `Failure` 가 기존 타입으로 복원되고 publisher 가 `Publisher<String, NameError> 로 돌아간다.

전체 호출을  `tryMap` 으로 바꾸면

```swift
.tryMap { throw NameError.tooShort($0) }

——— Example of: map vs tryMap ———
Hello is too short!
```

<br />


## 🔑 Key points

- `Failure` 타입이 `Never`  인 Publisher 는 실패 완료 이벤트를 emit 하지 않는다
- `sink(receiveValue:)` , `setFailureType` , `assertNoFailure` , `assign(to:on:)` operator 들은 실패하지 않는 publisher 에서만 작동한다
- `try-`prefixed operator 는 그 안에서 에러를 던질 수 있게 한다
- `try-`prefixed operator 는 publisher 의 `Failure` 를 Swift 의 `Error` 로 바꾼다
- Publisher 의 `Failure` 타입 매핑을 위해서 `mapError` 를 사용하고, publisher 의 failure type 을 single type 으로 통합한다
- 고유 `Failure` 타입이 있는 Publisher 를 기반으로 API 를 만들 때, 가능한 에러 타입을 래핑하여 통합하고 API 구현 세부 정보를 숨겨야 한다
- `retry` operator 를 사용하면 추가적으로 여러번 failed publisher 를 재구독 할 수 있다
- `replaceError(with:)` 는 publisher 에 failure case 에서 디폴트 fallback 값 을 제공할 때 유용하다
- `catch` 를 사용해서 failed publisher 를 다른 fallback publisher 로 교체 할 수 있다
