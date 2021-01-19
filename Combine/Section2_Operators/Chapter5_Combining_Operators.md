# Chapter 5: Combining Operators

Combining operators 는 다른 publisher 로 부터 emit 된 이벤트를 조합하고 데이터를 의미 있는 조합으로 생성 할 수 있다.

<br />

# 🐻‍❄️ Prepending

Original publihser 로부터 값이 emit 되기 전에 값을 추가 할 수 있는 operator 이다.

### `prepend(Output...)`

This variation of prepend takes a variadic list of values using the variadic ... syntax. This means it can take any number of values, as long as they’re of the same Output type as the original publisher.

Original publihser의 Output 타입과 같으면, 원하는 개수의 값을 취할 수 있다?

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled.png)

<br />

```swift
example(of: "pretend(Output...") {
    let publihser = [3, 4].publisher
    
    publihser
        .prepend(1, 2)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: prepend(Output...) ———
1
2
3
4
```

`pretend` 가 publisher 의 값 전에 1, 2를 추가했다.

Operator 는 체이닝이 가능하기 때문에, `pretend` 를 쉽게 더 추가 할 수 있다.

위 예제의 `.prepend(1, 2)` 아래에 `.prepend(-1, 0)` 을 추가하면

<br />

```swift

——— Example of: prepend(Output...) ———
-1
0
1
2 
3 
4
```

여기서는 Operation 의 순서가 중요하다. 마지막 `pretend` 가 upstream 에 먼저 영향을 준다.

### `prepend(Sequence)`

위의 pretend 와 비슷 하지만, `Sequence`를 conform 하는 객체를 input 으로 받는다. 예를 들어, `Array` 혹은 `Set`가 올 수 있다. 

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%201.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled1.png)

<br />


```swift
example(of: "prepend(Sequence)") {
    let publisher = [5, 6, 7].publisher
    
    publisher
        .prepend([3, 4])
        .prepend(Set(1...2))
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: prepend(Sequence) ———
2
1
3
4
5
6
7
```

> Note : `Set` 는 `Array` 와 달리 순서가 없다, 그래서 어떤 아이템이 emit 될지 순서가 보장 되지 않는다. 위의 결과에서 처음 두개의 1, 2 은 2, 1 이 될 수 도 있다.

위의 `.pretend(Set(1...2))` 아래에  `.prepend(stride(from: 6, to: 11, by: 2))` 를 추가해보자.

`Strideable` 은 `Sequence` 를 conform 하고 있다.

<br />

```swift
——— Example of: prepend(Sequence) ———
6
8
10
1 
2 
3 
4 
5 
6
7
```

<br />

### `prepend(Publisher)`

Original publisher 의 값이 emit 되기 전에 두번째 publisher 에서 emit 된 값을 추가 할 수 있다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%202.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled2.png)

<br />

```swift
example(of: "prepend(Publisher)") {
    let publisher1 = [3, 4].publisher
    let publisher2 = [1, 2].publisher
    
    publisher1
        .prepend(publisher2)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: prepend(Publisher) ———
1
2
3
4
```
<br />

이 operator 에 대해 더 자세히 알아야 할 것이 있다.

```swift
example(of: "prepend(Publihser) #2") {
    let publisher1 = [3, 4].publisher
    let publisher2 = PassthroughSubject<Int, Never>()
    
    publisher1
        .prepend(publisher2)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
    
    publisher2.send(1)
    publisher2.send(2)
}

——— Example of: prepend(Publisher) #2 ———
1
2
```

왜 publisher2 에서 두개 값만 emit  되었을까? Combine 은 publisher2 가 값을 emit 하는 것을 종료 했다는 것을 어떻게 알까?

값을 emit 하기는 했지만, completion event 가 없다. pretending 이 종료 되었고 primary publisher 로 이어 가는 것을 알기 위해 pretended publisher 는 complete 되어야만 한다. 

`publisher2.send(2)` 뒤에 `publihser2.send(completion: .finished)` 를 추가하자.

```swift
——— Example of: prepend(Publisher) #2 ———
1
2
3
4
```

Combine 은 이제 publihser2 가 작업을 끝냈기 때문에 publisher1이 emission 을 처리 할 수 있다는 것을 안다.

<br />

# 🐻‍❄️ Appending

이번 operator 들은 publisher 가  emit 하는 값을 다른 값과 연결하는 작업을 다룹니다.

### `append(Output...)`

`pretend` 와 유사하게 작동 한다. Output 타입의 variadic list 를 가지고 있고 original publisher 가 `.finished` 이벤트와 함께 종료 된 후에 아이템을 `append` 한다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%203.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled3.png)

<br />

```swift
example(of: "append(Output...)") {
    let publisher = [1].publisher
    
    publisher
        .append(2, 3)
        .append(4)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: append(Output...) ———
1
2
3
4
```

각각의 `append` 는 값을 추가하는 작업을 하기 전에 upstream 이 완료 되기를 기다린다.

이것은 upstream 이 꼭 완료 되어야 한다는 것을 의미 한다. previous publisher 가 모든 값 emit 을 끝냈다는 것을 모르면, appending 도 절대 되지 않을 것이다.

<br />

```swift
example(of: "append(Output) #2") {
    let publihser = PassthroughSubject<Int, Never>()
    
    publihser
        .append(3, 4)
        .append(5)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
    
    publihser.send(1)
    publihser.send(2)
		publihser.send(completion: .finished)
}

——— Example of: append(Output...) #2 ———
1
2
3
4 
5
```

`publisher.send(completion: .finishe)` 를 해주어야 함

Original publihser 가 completion event 를 보내기 전까지 appending 이 발생하지 않는 것은  모든 `append` operator 에 적용된다. 

<br />

### `append(Sequence)`

이 `append` 는 `Sequence` 를 conform 하는 객체에 적용되고 original publisher 가 모든 값을  emit 한 후에 append 한다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%204.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled4.png)

<br />

```swift
example(of: "append(Sequence)") {
    let publisher = [1, 2, 3].publisher
    
    publisher
        .append([4, 5])
        .append(Set([6, 7]))
        .append(stride(from: 8, through: 11, by: 2))
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: append(Sequence) ———
1
2
3
4 
5 
7 
6 
8 
10
```

`append` 의 실행은 다음 append가 실행 되지 전에 이전 publisher 가 완료 되어야 하기 때문에 순차적이다.

그리고 6과 7은 `Set` 이기 때문에 순서가 다를 수 있다.

<br />

### `append(Publisher)`

 `Publisher` 를 취하고 original publisher 가 값 emit 을 종료 한 후에 emit 된 모든 값을 append 한다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%205.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled5.png)

<br />

```swift
example(of: "append(Publisher)") {
    let publisher1 = [1, 2].publisher
    let publisher2 = [3, 4].publisher
    
    publisher1
        .append(publisher2)
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: append(Publisher) ———
1
2
3
4
```
<br />

# 🐻‍❄️ Advanced combining

다른 publisher 를 조합 하는 더 복잡한 operator 에 대해 알아보자.

### `switchToLatest`

It lets you switch entire publisher subscriptions on the fly while canceling the pending publisher subscription, thus switching to the latest one.

보류 중인 publisher 의 구독을 취소하는 동안 최신 subscription 으로 전체 publisher subscription 을 교체 할 수 있다. 

Publisher 를 emit 하는 publisher 에서만 사용 가능하다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%206.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled6.png)

<br />

```swift
example(of: "switchToLatest") {
    let publisher1 = PassthroughSubject<Int, Never>()
    let publisher2 = PassthroughSubject<Int, Never>()
    let publisher3 = PassthroughSubject<Int, Never>()
    
    let publishers = PassthroughSubject<PassthroughSubject<Int, Never>, Never>()
    
    publishers
        .switchToLatest()
        .sink(receiveCompletion: { _ in print("Completed!") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
    
    publishers.send(publisher1)
    publisher1.send(1)
    publisher1.send(2)
    
    publishers.send(publisher2)
    publisher2.send(3)
    publisher2.send(4)
    publisher2.send(5)
    
    publishers.send(publisher3)
    publisher3.send(6)
    publisher3.send(7)
    publisher3.send(8)
    publisher3.send(9)
    
    publisher3.send(completion: .finished)
    publishers.send(completion: .finished)
}

——— Example of: switchToLatest ———
1
2
4
5
7
8
9
Completed!
```

유저가 네트워크 요청을 트리거 하는 버튼을 탭한다. 그 즉시, 유저가 두번째 네트워크 요청을 트리거하는 그 버튼을 또 탭한다. 보류 중인 요청은 어떻게 제거하고 가장 최신 것만 사용 할 때 `switchToLatest` 를 사용 하면 된다.

<br />

```swift
example(of: "switchToLatest - Network Request") {
    let url = URL(string: "https://source.unsplash.com/random")!
    
    func getImage() -> AnyPublisher<UIImage?, Never> {
        URLSession.shared
            .dataTaskPublisher(for: url)
            .map { data, _ in UIImage(data: data) }
            .print("image")
            .replaceError(with: nil)
            .eraseToAnyPublisher()
    }
    
    let taps = PassthroughSubject<Void, Never>()
    
    taps
        .map { _ in getImage() }
        .switchToLatest()
        .sink(receiveValue: { _ in})
        .store(in: &subscriptions)
    
    taps.send()
    
    DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
        taps.send()
    }
    DispatchQueue.main.asyncAfter(deadline: .now() + 3.1) {
        taps.send()
    }
}

——— Example of: switchToLatest - Network Request ———
  image: receive subscription: (DataTaskPublisher)
  image: request unlimited
  image: receive value: (Optional(<UIImage:0x600000364120 anonymous {1080, 720}>))
  image: receive finished
  image: receive subscription: (DataTaskPublisher)
  image: request unlimited
  image: receive cancel
  image: receive subscription: (DataTaskPublisher)
  image: request unlimited
  image: receive value: (Optional(<UIImage:0x600000378d80 anonymous {1080, 1620}>))
  image: receive finished
```

실제로 두개의 이미지만 fecth 된 것을 볼 수 있다. 왜냐하면 두번째와 세번째 탭 사이에 10분의 1초만 지났기 때문이다. 두번째 fetch 의 반환 값을 받기 전에, 두번째 subscription을 취소 하면서 세번째 탭이 새로운 요청을 전환했다. 그래서 `image: receive cancel.` 이라는 라인이 있다.

<br />

더 나은 시각화를 원할 경우, 아래 버튼을 탭해라.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%207.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled7.png)

이미지를 우클릭 하고, Value History 를 클릭 하면 두 이미지를 스크롤 하면서 볼 수 있다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%208.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled8.png)

<br />

### `merge(with:)`

이 opreator 는 동일한 유형의 다른 publisher 로 부터 emission 값들을 상호 배치 한다(interleaves).

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%209.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled9.png)

<br />

```swift
example(of: "merge(with:)") {
    let publisher1 = PassthroughSubject<Int, Never>()
    let publisher2 = PassthroughSubject<Int, Never>()
    
    publisher1
        .merge(with: publisher2)
        .sink(receiveCompletion: { _ in print("Completed") },
              receiveValue: { print($0) })
        .store(in: &subscriptions)
    
    
    publisher1.send(1)
    publisher1.send(2)
    
    publisher2.send(3)
    
    publisher1.send(4)
    
    publisher2.send(5)
    
    publisher1.send(completion: .finished)
    publisher2.send(completion: .finished)
}

——— Example of: merge(with:) ———
1
2
3
4
5 Completed
```

<br />

### `combineLatest`

다른 publisher 들을 조합하는 또다른 operator 이다. 다른 값 타입의 publisher 를 조합 가능하다.

하지만, 모든 publisher 의 emission 을 조합하는 대신, publisher 가 값을 emit 할 때 가장 최신의 값을 튜플 형태로 emit 한다.

 `combineLatest` 로 전달된 Origin publisher 와 모든 publisher 는 `combineLatest` 가 어떤 것을 emit 하기 전에, 적어도 하나의 값을 emit 해야만 한다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%2010.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled10.png)

<br />

```swift
example(of: "combineLatest") {
    let publisher1 = PassthroughSubject<Int, Never>()
    let publisher2 = PassthroughSubject<String, Never>()
    
    publisher1
        .combineLatest(publisher2)
        .sink(receiveCompletion: { _ in print("Completed") },
              receiveValue: { print("P1: \($0), P2: \($1)")})
        .store(in: &subscriptions)
    
    publisher1.send(1)
    publisher1.send(2)
    
    publisher2.send("a")
    publisher2.send("b")
    
    publisher1.send(3)
    
    publisher2.send("c")
    
    publisher1.send(completion: .finished)
    publisher2.send(completion: .finished)
}

——— Example of: combineLatest ———
P1: 2, P2: a
P1: 2, P2: b
P1: 3, P2: b
P1: 3, P2: c
Completed
```

`publisher1` 으로 부터 emit 된 1 은 `combineLatest` 로 푸시 되지 않는다. 왜냐하면 `combineLatest` 는 모든 publisher 가 적어도 하나의 값을 emit 했을 때만 조합 하기 때문이다.

<br />

### `zip`

Swift standard library 메소드의 `Sequence` 타입인 것에서 온 것을 알 수 있다. 이 operator 는 동일한 인덱스의 값의 쌍을 튜플로 emit 한다. publisher 가 아이템을 emit 하기를 기다리고, 모든 publisher 가 현재 인덱스의 모든 값을 emit 했을 때 튜플을 emit 한다.

이것은 두개의 publisher 를 zip 했을 때, 두개의 publisher 에서 emit 된 값 두개로 단일 튜플을 emit 받는 다는 것을 의미한다.

![/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled%2011.png](/Combine/Section2_Operators/images/Chapter5_Combining_Operators/Untitled11.png)

<br />

```swift
example(of: "zip") {
    let publisher1 = PassthroughSubject<Int, Never>()
    let publisher2 = PassthroughSubject<String, Never>()
    
    publisher1
        .zip(publisher2)
        .sink(receiveCompletion: { _ in print("Completed") },
              receiveValue: { print("P1: \($0), P2: \(1)") })
        .store(in: &subscriptions)
    
    publisher1.send(1)
    publisher1.send(2)
    publisher2.send("a")
    publisher2.send("b")
    publisher1.send(3)
    publisher2.send("c")
    publisher2.send("d")
    
    publisher1.send(completion: .finished)
    publisher2.send(completion: .finished)
}

——— Example of: zip ———
P1: 1, P2: a
P1: 2, P2: b
P1: 3, P2: c
Completed
```

각각의 emit 된 값은 다른 zipped 된 publisher 가 값을 emit 하기를 기다린다. 마지막에 두번째 publisher 에서 emit 된 "d" 는 첫번째 publisher 로 부터 쌍으로 대응되는 값이 없기 때문에 무시 되었다.

<br />

# 🔑 Key points

- `prepend` 와 `append` operator 는 original publisher 의 전이나 이후에 하나의 publisher 에 emission 을 추가 할 수 있다.
- `switchToLatest` 는 복잡하지만 매우 유용하다. publisher 를 받아, 최신의 publisher 로 교체하고 이전 publisher 의 subscription 은 취소한다.
- `merge(with:)` 는 다수의 publisher 의 값을 interleave 할 수 있다.
- `combineLatest` 는 모든 조합된 publisher 가 적어도 하나의 값을 emit 한 후에, 모든 조합 된 publisher 가 값을 emit 할 때 마다, 가장 최신의 값을 emit 한다.
- `zip` 은 모든 publisher 가 값을 emit 한 후에, 다른 publisher 의 emission들의 튜플로 쌍을 지어준다.
- Combination operator publisher 와 그들의 emission 간의 관계를 흥미롭고 복잡한 하게 만들기 위해 혼합 시킬 수 있다.
