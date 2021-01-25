# Chapter 7: Sequence Operators

- Publisher 값의 array 나 set 같은 collection 과 관련된 작업을 한다 .
- Sequence operator 는 는 대부분 시퀀스 전체를 처리하며 다른 operator 들 처럼 개별 값을 다루지 않는다.
- Swift standard library 와 거의 비슷한 이름과 행위를 한다.

<br />

## 🐻‍❄️ Finding values

### `min`

- Publisher 에 의해 emit 된 값 중 가장 작은 값을 찾는다.
- greedy → Publisher 가 `.finished` completion 이벤트를 보낼 때까지 대기한다.
- Publisher 가 종료 되면, operator 에 의해 가장 작은 값만 emit 된다.

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled.png)

<br />

```objectivec
example(of: "min") {
    let publihser = [1, -50, 246, 0].publisher
    
    publihser
        .print("publisher")
        .min()
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: min ———
publisher: receive subscription: ([1, -50, 246, 0])
publisher: request unlimited
publisher: receive value: (1)
publisher: receive value: (-50)
publisher: receive value: (246)
publisher: receive value: (0)
publisher: receive finished
-50
```

Publisher 가 모든 값을 emit 하고 종료 한 후에 `min` 이 가장 작은 값을 찾아서 `sink` 가 print 할 수 있도록 downstream 으로 전송한다.

- `Comparable` 프로토콜을 conform `min()` 을 바로 사용할수 있다.

<br />

`Comparable` 을 conform 하지 않는 경우에는, `min(by:)`  operator 를 통해 comparator closure 를 제공 할 수 있다.

```objectivec
example(of: "min non-Comparable") {
    let publihser = ["12345",
                     "ab",
                     "hello world"]
        .compactMap { $0.data(using: .utf8) }
        .publisher
    
    publihser
        .print("publisher")
        .min(by: { $0.count < $1.count })
        .sink(receiveValue: { data in
            let string = String(data: data, encoding: .utf8)!
            print("Smallest data is \(string), \(data.count) bytes")
        })
        .store(in: &subscriptions)
}

——— Example of: min non-Comparable ———
publisher: receive subscription: ([5 bytes, 2 bytes, 11 bytes])
publisher: request unlimited
publisher: receive value: (5 bytes)
publisher: receive value: (2 bytes)
publisher: receive value: (11 bytes)
publisher: receive finished
Smallest data is ab, 2 bytes
```

전 예제 처럼, Publisher 가 모든 `Data` 객체가 emit 되고 끝난 후에, `min(by:)` 가장 작은 byte size 를 찾아서 emit 하고 `sink` 가 출력 한다. 

<br />

### `max`

- Publisher에 의해 emit 된 값 중 가장 큰 값을 찾는다.

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%201.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%201.png)

<br />

```objectivec
example(of: "max") {
    let publihser = ["A", "F", "Z", "E"].publisher
    
    publihser
        .print("publihser")
        .max()
        .sink(receiveValue: { print($0) })
        .store(in: &subscriptions)
}

——— Example of: max ———
publihser: receive subscription: (["A", "F", "Z", "E"])
publihser: request unlimited
publihser: receive value: (A)
publihser: receive value: (F)
publihser: receive value: (Z)
publihser: receive value: (E)
publihser: receive finished
Z
```

`min` 처럼 greedy → upstream publisher 가 값 emit 을 종료할 때까지 기다리고 최대값을 찾는다.

> Note : `min` 과 같이, `max` 도 non-Comparable 값 사이에서 최대값을 예측하는`max(by:)` operator 가 있다.

<br />

### `first`

- Swift collection 의 `first` 프로퍼티와 유사지만
- 첫번째 값이 emit 되면 종료 된다.
- lazy → upstream publisher 가 종료되길 기다리지 않고, 첫번째 값을 emit 한 후에 구독을 취소한다.

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%202.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%202.png)

<br />

```objectivec
example(of: "first") {
    let publisher = ["A", "B", "C"].publisher
    
    publisher
        .print("publisher")
        .first()
        .sink(receiveValue: { print("First value is \($0)") })
        .store(in: &subscriptions)
}

——— Example of: first ———
publisher: receive subscription: (["A", "B", "C"])
publisher: request unlimited
publisher: receive value: (A)
publisher: receive cancel
First value is A
```

`first()` 는 첫번째 값이 통과 하자마자 upstream publisher 의 구독을 즉시 취소하게 한다.

<br />

`first(where:)` 를 사용하면 제공된 조건에 매치하는 첫번째 값을 emit 한다.

```objectivec
example(of: "first(where:)") {
    let publisher = ["J", "O", "H", "N"].publisher
    
    publisher
        .print("publisher")
        .first(where: { "Hello World".contains($0) })
        .sink(receiveValue: { print("First match is \($0)") })
        .store(in: &subscriptions)
}

——— Example of: first(where:) ———
publisher: receive subscription: (["J", "O", "H", "N"])
publisher: request unlimited
publisher: receive value: (J)
publisher: receive value: (O)
publisher: receive value: (H)
publisher: receive cancel
First match is H
```

<br />


### `last`

- Publisher 가 emit 하는 가장 마지막 값
- greedy → upstream publisher 가 종료되기를 대기

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%203.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%203.png)

<br />

```objectivec
example(of: "last") {
    let publisher = ["A", "B", "C"].publisher
    
    publisher
        .print("publisher")
        .last()
        .sink(receiveValue: { print("Last value is \($0)") })
        .store(in: &subscriptions)
}

——— Example of: last ———
publisher: receive subscription: (["A", "B", "C"])
publisher: request unlimited
publisher: receive value: (A)
publisher: receive value: (B)
publisher: receive value: (C)
publisher: receive finished
Last value is C
```

`last` 는 upstream publisher 가 가장 마지막 값을 downstream 에 emit 하고 `sin` 가 출력하도록 하는 시점인  `.finished` completion 이벤트를 보낼때까지 대기 한다.

> Note : `first` 처럼 `last` 도 지정한 조건을 만족하는 값을 마지막 값을 emit 하는 `last(where:)` 가 있다.

<br />

### `output(at:)`

- Upstream publisher 로 부터 emit 된 값 중 지정한 index 의 값만 통과

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%204.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%204.png)

<br />

```objectivec
example(of: "output(at:)") {
    let publisher = ["A", "B", "C"].publisher
    
    publisher
        .print("publisher")
        .output(at: 1)
        .sink(receiveValue: { print("Value at index 1 is \($0)") })
        .store(in: &subscriptions)
}

——— Example of: output(at:) ———
publisher: receive subscription: (["A", "B", "C"])
publisher: request unlimited
publisher: receive value: (A)
publisher: request max: (1) (synchronous)
publisher: receive value: (B)
Value at index 1 is B
publisher: receive cancel
```

- 이 operator 는 모든 emit 된 값 이후에 하나를 더 요구한다. 지정된 인덱스의 아이템만 찾고 있다는걸 알기 때문에?

<br />

### `output(in:)`

- 제공된 범위 내에 있는 인덱스의 값을 emit

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%205.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%205.png)

<br />

```objectivec
example(of: "output(in:)") {
    let publisher = ["A", "B", "C", "D", "E"].publisher
    
    publisher
        .output(in: 1...3)
        .sink(receiveCompletion: { print($0) },
              receiveValue: { print("Value in range: \($0)")})
        .store(in: &subscriptions)
}

——— Example of: output(in:) ———
Value in range: B
Value in range: C
Value in range: D
finished
```

- 범위 내의 인덱스들을 콜렉션이 아닌 각각의 값으로 emit 한다.
- 범위 내의 모든 아이템들이 emit 되었기 때문에, 필요한 모든 항목을 받는 즉시 구독을 취소한다.

<br />

## 🐻‍❄️ Querying the publihser

- Publisher 에서 emit 된 값의 전체 set 을 다루지만, 지정된 값을 생산하지 않는다.
- publisher 전체의 query를 대표하는 다른 값을 emit

### `count`

- Publisher 가 `.finished` completion 이벤트가 전송하면, Upstream publisher 에서 얼마나 많은 값이 emit 되었는지에 대한 숫자를 emit

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%206.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%206.png)

<br />

```objectivec
example(of: "count") {
    let publisher = ["A", "B", "C"].publisher
    
    publisher
        .print("publisher")
        .count()
        .sink(receiveValue: { print("I have \($0) items") })
        .store(in: &subscriptions)
}

——— Example of: count ———
publisher: receive subscription: (["A", "B", "C"])
publisher: request unlimited
publisher: receive value: (A)
publisher: receive value: (B)
publisher: receive value: (C)
publisher: receive finished
I have 3 items
```

<br />

### `contains`

- Upstream Publisher 로 부터 지정된 값이 emit 되면 `true` , emit 되지 않으면 `false`

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%207.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%207.png)

<br />

```objectivec
example(of: "contains") {
    let publisher = ["A", "B", "C", "D", "E"].publisher
    let letter = "C"
    
    publisher
        .print("publisher")
        .contains(letter)
        .sink(receiveValue: { contains in
            print(contains ? "Publisher emitted \(letter)!"
                    : "Publisher never emitted \(letter)!")
        })
        .store(in: &subscriptions)
}

——— Example of: contains ———
publisher: receive subscription: (["A", "B", "C", "D", "E"])
publisher: request unlimited
publisher: receive value: (A)
publisher: receive value: (B)
publisher: receive value: (C)
publisher: receive cancel
Publisher emitted C!
```

- lazy → 지정 된 값을 찾으면, 구독을 취소하고 더이상 값을 생산하지 않는다.

<br />

```objectivec
let letter = "F"

——— Example of: contains ———
publisher: receive subscription: (["A", "B", "C", "D", "E"])
publisher: request unlimited
publisher: receive value: (A)
publisher: receive value: (B)
publisher: receive value: (C)
publisher: receive value: (D)
publisher: receive value: (E)
publisher: receive finished
Publisher never emitted F!
```

<br />

`contains(where:)`

`Comparable` 을 conform 하지 않는 emit 된 값의 존재를 제공하거나 확인할 때

```objectivec
example(of: "contains(where:)") {
    struct Person {
        let id: Int
        let name: String
    }
    
    let people = [
        (456, "Scott Gardner"),
        (123, "Shai Mishali"),
        (777, "Jinha Park"),
        (214, "Florent Pillet")
    ]
    .map(Person.init)
    .publisher
    
    people
        .contains(where: { $0.id == 800 })
        .sink(receiveValue: { contains in
            print(contains ? "Criteria matches!" : "Couldn't find a match for the criteria")
        })
        .store(in: &subscriptions)
}

——— Example of: contains(where:) ———
Couldn't find a match for the criteria
```

```objectivec
.contains(where: { $0.id == 800 || $0.name == "Jinha Park" })

——— Example of: contains(where:) ———
Criteria matches!
```

<br />

### `allSatisfy`

- Clouser 조건을 사용하고 모든 값이 조건에 맞는지에 대한 Boolean 을 emit
- greedy → upstream publisher 가 `.finished` completion 이벤트를 emit 할 때까지 대기

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%208.png](/Combine/Section2_Operators/images/Chapter7-Sequence-Operators/Untitled%208.png)

<br />

```objectivec
example(of: "allSatisfy") {
    let publisher = stride(from: 0, to: 5, by: 2).publisher
    
    publisher
        .print("publisher")
        .allSatisfy { $0 % 2 == 0 }
        .sink(receiveValue: { allEven in
            print(allEven ? "All numbers are even" : "Something is odd...")
        })
        .store(in: &subscriptions)
}

——— Example of: allSatisfy ———
publisher: receive subscription: (Sequence)
publisher: request unlimited
publisher: receive value: (0)
publisher: receive value: (2)
publisher: receive value: (4)
publisher: receive finished
All numbers are even
```

모든 값이 짝수 이기 때문에, `.finished` completion gndp, operator 는 `true` 를 emit 하고 적절한 메세지를 출력한다.

```objectivec
let publisher = stride(from: 0, to: 5, by: 1).publisher

——— Example of: allSatisfy ———
publisher: receive subscription: (Sequence)
publisher: request unlimited
publisher: receive value: (0)
publisher: receive value: (1)
publisher: receive cancel
Something is odd...
```

1 이 emit 되면, 조건이 더이상 통과하지 않아서, `false` 를 리턴하고 구독을 취소한다.

<br />

### `reduce`

- Upstream publisher 의 emission 된 값에 기반해서 새로운 값을 축척할수 있다

![Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%209.png](Chapter%207%20Sequence%20Operators%20fa7a8acbad484c76875cf7f1513bad99/Untitled%209.png)

<br />

```objectivec
example(of: "reduce") {
    let publisher = ["Hel", "lo", " ", "Wor", "ld", "!"].publisher
    
    publisher
        .print("publisher")
        .reduce("") { accumulator, value in
            accumulator + value
        }
        .sink(receiveValue: { print("Reduced into: \($0)") })
        .store(in: &subscriptions)
}

——— Example of: reduce ———
publisher: receive subscription: (["Hel", "lo", " ", "Wor", "ld", "!"])
publisher: request unlimited
publisher: receive value: (Hel)
publisher: receive value: (lo)
publisher: receive value: ( )
publisher: receive value: (Wor)
publisher: receive value: (ld)
publisher: receive value: (!)
publisher: receive finished
Reduced into: Hello World!
```

`reduce` 의 두번째 인자는 어떤 타입의 두개의 값을 가지고 같은 타입을 리턴한다.

```objectivec
.reduce("") { accumulator, value in
  return accumulator + value
}
⏸
.reduce("", +)
```

> Note : Chapter 3, "Transforming Operators" 의 `scan` 과  `reduce` 와 기능적으로는 똑같지만, `scan` 은 모든 emit 된 value 에 대해 축적 된 값을 emit 하고, `reduce` 는 하나의 축적된 값만 emit 한다

<br />

## 🔑 Key points

- Publisher 는 collection 과 sequence 가 하는 것 처럼 값을 생산하기 때문에 Sequence 이다.
- `min` , `max` 를 사용해서 publisher 에서 emit 된 값의 최소값, 최대값을 emit 할 수 있다.
- `first` , `last` , `output(at:)` 은 emit 된 값의 특정 인덱스를 찾을 때 유용하다. `output(in:)` 은 인덱스 범위내의 emit 된 값을 찾을 때 사용한다.
- `first(where:)` , `last(where:)` 는 각각 어떤 값이 통과 될지 결정하는 조건을 갖는다.
- `count` , `contains` , `allSatisfy` 같은 operator 는 publisher 로 부터 emit 된 값을 emit 하는 대신, emit 된 값을 기반으로 다른 값을 emit 한다.
- `contains(where:)` 는 주어진 값에 포함되는지에 대한 조건을 갖는다.
- `reduce` 는 emit 된 값들을 하나의 값으로 축적할 떄 사용한다.
