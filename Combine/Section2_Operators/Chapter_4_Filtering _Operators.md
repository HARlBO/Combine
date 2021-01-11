# Chapter 4: Filtering Operators

이번 챕터에서는 Publisher로 부터 emit 된 값 또는 이벤트를 제한하고, 해당 값만 사용 하고 싶을때 사용할 수 있는 operator 를 다룰 것이다: Filtering operators!

> Note :  이번 챕터에서 대부분의 operator는 `filter vs tryFilter` 같은, `try` prefix 를 갖는다. 중요한 차이점은 `tryFilter` 가 `throwing` 클로저를 제공해주는 것이다. Closure 에서 throw 된 에러는 에러와 함께 publisher를 종료 시킬 것이다. 이번 챕터에서는 간결함을 위해, 사실상 동일한, 에러를 던지지 않는 non-throwing 만 다룰 것이다.

<br />

## 🐻‍❄️ Filtering basic

### `Filter`

- Publisher 의 값을 사용하고, 어떤 값을 consumer 에게 전달할지 조건부로 결정
- `Bool` 값을 리턴하고, 제공 된 조건에 맞는 값만 전달하는 클로저

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled.png)

</br>

```swift
example(of: "filter") {
    let numbers = (1...10).publisher
    
    numbers
        .filter { $0.isMultiple(of: 3) }
        .sink(receiveValue: { n in
            print("\(n) is a multiple of 3!")
        })
        .store(in: &subscriptions)
}

——— Example of: filter ———
3 is a multiple of 3!
6 is a multiple of 3!
9 is a multiple of 3!
```

</br>

### `removeDuplicates`

- Publihser가 emit 하는 동일한 값을 무시하고 싶을 때
- `String` 을 포함한 `Equatable` 을 준수 하는 값들에 사용

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled1.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled1.png)

</br>

```swift
example(of: "removeDuplicate") {
    let words = "hey hey there! want to listen to mister mister ?"
        .components(separatedBy: " ")
        .publisher
    
    words
        .removeDuplicates()
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: removeDuplicate ———
hey
there!
want
to
listen
to
mister
?
```

- 두번째 "hey" 와 "mister" 가 생략 되었다.

</br>

> Note : `Equatable` 을 준수하지 않는 값은 어떨까? `removeDuplicate` 은 두개의 값을 갖는 클로저가, 값들이 같은지 아닌지 `Bool` 을 리턴하는 또 다른 오버로드 한 것이 있다.

</br>

## 🐻‍❄️ Compacting and ignoring

### `compactMap`

- Publisher가 `Optional` 값을 emit 할때, 또는 `nil`을 리턴하는 operation의 값을 수행 하고 싶지만, 이 모든  `nil` 을 처리하고 싶지 않을 때

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled2.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled2.png)

</br>

```swift
example(of: "compactMap") {
    let strings = ["a", "1.24", "3",
                   "def", "45", "0.23"].publisher
    
    strings
        .compactMap { Float($0) }
        .sink(receiveValue: {
            print($0)
        })
        .store(in: &subscriptions)
}

——— Example of: compactMap ———
1.24
3.0
45.0
0.23
```
</br>


### `ignoreOutput`

- Publisher 가 실제로 값을 emit 했는지와 상관 없이, emit 하는 것을 끝냈는지 알고 싶을 때

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled3.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled3.png)

- 어떤 값이 몇번 emit 됐는지는 무시하고, completion event 만 받는다.

</br>

```swift
example(of: "ignoreOutput") {
    let numbers = (1...10_000).publisher
    
    numbers
        .ignoreOutput()
        .sink(receiveCompletion: { print("Complted with \($0)") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: ignoreOutput ———
Complted with finished
```

</br>

## 🐻‍❄️ Finding values

- 각각 제공된 조건과 매칭되는 첫번째 또는 마지막 값을 찾거나 emit 하기 위해 사용

</br>

### `first(where:)`

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled4.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled4.png)

- 이 operator는 `lazy` 하다. 즉, 제공된 조건과 매칭 되는 하나를 찾을 때까지 필요한 값만 사용한다.
- 매칭 되는 값을 찾으면, subscription을 취소하고, 완료한다.

</br>

```swift
example(of: "first(where:)") {
    let numbers = (1...9).publisher
    
    numbers
        .first(where: { $0 % 2 == 0 })
        .sink(receiveCompletion: { print("Completed with \($0)") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: first(where:) ———
2
Completed with finished
```

upstream 구독은 어떻게 될까? 짝수에 매칭 되는 값을 찾은 후에도 값을 계속 emit 할까?

numbers 다음 라인에 `.print("numbers")` 를 추가해보면,

</br>

```swift
——— Example of: first(where:) ———
numbers: receive subscription: (1...9)
numbers: request unlimited
numbers: receive value: (1)
numbers: receive value: (2)
numbers: receive cancel
2
Completed with finished
```

매칭 되는 값을 찾은 후에, subscription을 통해, upstream이 값을 emit 하는 것을 멈추는 cancellation을 전송한다.

</br>

### `last(where:)`

- 제공된 조건에 맞는 마지막 값을 찾는데 사용

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled5.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled5.png)

- 이 operator는 `greedy` 하다. 매칭되는 값을 찾았는지 여부를 알고 emit 하기 위해 모든 값들을 기다려야 하기 때문.
- 이런 이유로, upstream은 특정 시점에 완료되는 publisher 이다.

</br>

```swift
example(of: "last(where:)") {
    let numbers = (1...9).publisher
    
    numbers
        .last(where: { $0 % 2 == 0 })
        .sink(receiveCompletion: { print("Completed with : \($0)") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: last(where:) ———
8
Completed with : finished
```

전에 말했던 것 처럼 이 operator 가 작동하려면 publihser가 완료되어야 한다. 왜일까?

왜냐하면, publisher가 다음 라인의 기준과 매칭하는 값을 emit 할지  operator가 알 수 있는 방법이 없기 때문에,  operator는 publisher가 기준과 일치하는 마지막 아이템을 결정 짓기 전에, publihser의 전체 범위를 알아야 한다.

</br>

```swift
example(of: "last(where:)") {
    let numbers = PassthroughSubject<Int, Never>()
    numbers
      .last(where: { $0 % 2 == 0 })
      .sink(receiveCompletion: { print("Completed with: \($0)") },
            receiveValue: { print($0) })
      .store(in: &subscriptions)
    numbers.send(1)
    numbers.send(2)
    numbers.send(3)
    numbers.send(4)
    numbers.send(5)
}

——— Example of: last(where:) ———
```

`PassthroughSubject` 를 이용하여 이벤트를 수동으로 전송하도록 한다.

- Publihser 는 절대 완료되지 않기 때문에, 마지막 값이 기준에 맞는지 결정할 방법이 없다.

이것을 고치기 위해 마지막 라인에 completion 을 전송하는 코드를 추가하자.

</br>

```swift
numbers.send(completion: .finished)

——— Example of: last(where:) ———
4
Completed with: finished
```

</br>

## 🐻‍❄️ Dropping values

- Publisher로 부터 두번째 값이 publishing 될 때까지, 첫번째 값을 무시 할 때
- Stream 의 시작점에서 값의 특정 개수만큼 무시 하고 싶을 때

</br>

### `dropFirst`

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled6.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled6.png)

- `count` 파라미터를 갖는다. 디폴트는 1
- Publisher 로 부터 emit 된 값의 `count` 값 만큼 무시
- `count` 이후에 emit 된 값만 허용

</br>

```swift
example(of: "dropFirst") {
    let numbers = (1...10).publisher
    
    numbers
        .dropFirst(8)
        .sink(receiveValue:  { print($0) })
        .store(in: &subscriptions)
    
}

——— Example of: dropFirst ———
9
10
```
</br>

### `drop(while:)`

- 조건 closure 를 갖고,  Publisher로 부터 emit 된 값이 처음으로 조건에 맞을 때 까지 모든 값을 무시
- 조건을 만족되자마자, operator를 통해 값들이 통과

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled7.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled7.png)

</br>

```swift
example(of: "drop(while:)") {
    let numbers = (1...10).publisher
    
    numbers
        .drop(while: { $0 % 5 != 0 })
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: drop(while:) ———
5
6
7
8 9 10
```
</br>


`filter` 와 차이점

- 둘 다 closure의 결과에 따라, 어떤 값을 emit 할지 값을 제어하는 closure 를 취한다.
1. `filter` 는 closure 에서 `true` 를 리턴하면, 값이 통과 하게 하고, `drop(while:)` 은 클로저가 `true` 를 리턴하면, 값을 스킵한다.
2. 더 중요한 차이는 `filter` 는 upstream publisher에 의해 publish 된 모든 값의 조건을 평가하는 것을 멈추지 않는다. `filter` 조건이 `true` 로 판단 된 후에도, 값들은 계속 질문 받고, closure는 질문에 답해야 한다.
반대로 `drop(while:)` 의 조건 closure 는 조건에 부합한 후에는 절대 다시 실행되지 않는다.

</br>

### `drop(untilOutputFrom:)`

- `isReady` publisher 가 어떤 결과를 emit 하기 전까지의, publisher에서 emit 되는 값을 무시

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled8.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled8.png)

</br>

```swift
example(of: "drop(untilOutputFrom:)") {
    let isReady = PassthroughSubject<Void, Never>()
    let taps = PassthroughSubject<Int, Never>()
    
    taps
        .drop(untilOutputFrom: isReady)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
    
    (1...5).forEach { n in
        taps.send(n)
        if n == 3 {
            isReady.send()
        }
    }
}

——— Example of: drop(untilOutputFrom:) ———
4
5
```

## 🐻‍❄️ Limiting values

- 조건에 맞을때까지 값을 받고, Publisher 가 완료되도록 강제

</br>

### `prefix(_:)`

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled9.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled9.png)

</br>

```swift
example(of: "prefix") {
    let numbers = (1...10).publisher
    
    numbers
        .prefix(2)
        .sink(receiveCompletion:  { print("Completed with : \($0)") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
        
}

——— Example of: prefix ———
1
2
Completed with: finished
```

</br>

- 처음 두 값의 emission 만 허용하고, 두 값이 emit 된 후에는, publisher가 완료 된다
- `first(where:)` 와 같이, 이 operator 도 `lazy` 하다. 필요한 만큼의 값만 취하고 종료한다.
- `numbers` 가 1 과 2 이후에 값을 추가적으로 생산하는 것을 막는다.

</br>

### `prefix(while:)`

- 조건 clousure 를 갖는다
- 클로저가 `true` 를 리턴하면, upstream publihser 의 값이 통과하는 것을 허용
- 클로저의 결과가 `false` 를 리턴하면 publisher 는 완료된다.

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled10.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled10.png)

</br>

```swift
example(of: "prefix(while:)") {
    let numbers = (1...10).publisher
    
    numbers
        .prefix(while: { $0 < 3 })
        .sink(receiveCompletion: { print("Completed with: \($0)") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: prefix(while:) ———
1
2
Completed with: finished
```

</br>

### `prefix(untilOutputFrom:)`

- 두번째 publihser 가 emit 하기 전까지 값을 스킵하는  `drop(untilOutputFrom:)` 과 반대로,   
`prefix(untilOutputFrom:)` 두번째 publisher 가 값을 emit 할 때까지 값을 취한다.

</br>

![/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled11.png](/Combine/Section2_Operators/images/Chapter4_Filtering_Operators/Untitled11.png)

</br>

```swift
example(of: "prefix(untilOutputFrom:)") {
    let isReady = PassthroughSubject<Void, Never>()
    let taps = PassthroughSubject<Int, Never>()
    
    taps
        .prefix(untilOutputFrom: isReady)
        .sink(receiveCompletion: { print("Complted with: \($0)") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
    
    (1...5).forEach { n in
        taps.send(n)
        if n == 2 {
            isReady.send()
        }
    }
}

——— Example of: prefix(untilOutputFrom:) ———
1
2
Completed with: finished
```
</br>


## 🔑 Key points

- Filtering oprerator 들은 Upstream publihser 에서 downstream,  다른 operator 또는 comsumer에게  전송되는 값을 제어 할 수 있게 해준다
- 값 자체는 무시하고, completion event 만을 원한다면, `ignoreOutput` 을 사용
- 제공된 조건에 부합하는 첫번째 또는 마지막 값을 찾기 위한다면,  `first(where:)` `last(where:)` 사용
- First 계열의 operator 들은  `lazy` 하다. 필요한 만큼의 값을 취하고 완료한다.
Last 계열의 operator 들은 `greedy` 하다. 어떤 값이 조건을 충족하는 마지막 값인지 결정하기 전에, 값의 전체 범위를 알아야 한다.
- `drop` 계열의 operator를 사용해서 downstream에 값을 보내기 전에, upstream publisher 로 부터 emit 될 값의 개수를 제어 할 수 있다.
- `prefix` 계열의 operator를 사용해서 , 완료되기 전에 upstream publihser 가 몇개의 값을 emit 할지 제어 할 수 있다
