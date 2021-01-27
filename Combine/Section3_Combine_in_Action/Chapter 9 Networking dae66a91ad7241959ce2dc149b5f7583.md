# Chapter 9 : Networking

## 🐻‍❄️ URLSession extensions

`URLSession` 은 네트워크 데이터 전송 태스크를 수행하는데 권장되는 방법이다. 강력한 구성의 옵션과 투명한 백그라운드 지원을 갖춘 최신 비동기식 API 를 제공한다.

- URL 내용을 검색하는 데이터 전송 태스크
- URL 내용을 검색하고 파일로 저장하는 다운로드 태스크
- URL에 파일과 데이터를 업로드하는 업로드 태스크
- 두 파티 간 데이터 스트리밍 태스크
- 웹 소켓에 연결하는 웹소켓 태스크

이 중,  첫번째인 데이터 전송 데이터가 Combine publisher 를 expose 한다. Combine 은 `URLRequest` 또는 `URL` 을 갖는 두개의 변수로 단일 API 를 사용하여 태스크를 처리한다.

<br />


```swift
guard  let url = URL(string: "https://mysite.com/mydata.json") else {
    return
}

let subscription = URLSession.shared
    .dataTaskPublisher(for: url)
    .sink(receiveCompletion: { completion in
            if case .failure(let err) = completion {
                print("Retrieving data failed with error \(err)")
            }
    }, receiveValue: { data, response in
        print("Retrieved data of size \(data.count), response = \(response)")
    })
```

1. 결과 Subscription 를 유지하는 것은 중요하다. 그렇지 않으면, 구독이 즉시 취소되고 요청이 실행 되지 않는다.
2. URL 을 파라미터로 갖는 `dataTaskPublisher(for:)` 를 오버로드 한것을 사용한다.
3. 에러를 항상 처리해야 한다! 네트워크 연결은 실패하기 쉽다.
4. 결과 값은 `Data` 객체와 `URLResponse` 를 갖는 튜플 값이다.

Combine 은 클로저 대신 publisher 만 노출 하면서,  `URLSession.dataTask` 위에 투명한 bare-bone publisher 를 제공한다. 

<br />


## 🐻‍❄️ Codable support

`Codable` 프로토콜은 최신 강력한 Swift-only 인코딩 디코딩 메카니즘이다. Foundation 은 JSON 인코딩 디코딩을 `JSONEncoder` 과 `JSONDecoder` 로 제공한다.  또한 `PropertyListEncoder` 와 `PropertyListDecoder` 를 사용 할 수 있지만, 네트워크 요청 콘텍스트에는 덜 유용하다.

<br />


이전 예제에서 JSON 을 다운로드 했다. `JSONDecoder` 로 디코딩 할 수 있다.

```swift
let subscription = URLSession.shared
    .dataTaskPublisher(for: url)
    .tryMap { data, _ in
        try JSONDecoder().decode(MyType.self, from: data)
        
    }
    .sink(receiveCompletion: { completion in
            if case .failure(let err) = completion {
                print("Retrieving data failed with error \(err)")
            }
    }, receiveValue: { data, response in
        print("Retrieved data of size \(data.count), response = \(response)")
    })
```

JSON 을 `tryMap` 에서 디코드 할 수 있지만, Combine 을 boilerplate 를 줄일 수 있도록 operator 를 제공한다. → `decode(type:decoder:)`

<br />


```swift
.map(\data)
.decode(type: MyType.self, decoder: JSONDecoder())
```

불행히도, `dataTaskPublisher(for:)` 가 튜플을 emit 하기 때문에, 결과의 `Data` 부분만을 emit 해주는  `map(:)`  을 먼저 사용 하기 전에, `decode(type:decoder:)` 를 직접 사용 할 수 없다.

Publisher를 세팅할 때, `JSONDecoder` 를 한번만 인스턴스화 하면 되는 것이 장점이고, 단 `tryMap(_:)` 클로저마다 생성해 주어야 한다.

<br />


## 🐻‍❄️ Publishing network data to multiple subscribers

Publisher 를 구독 할 때 마다, 작업을 시작한다.

네트워크 요청 케이스에서, 이것은 만약 다수의 subscriber 가 결과를 필요로 할때, 같은 요청을 여러번 보내는 것을 의미한다. 

Combine 에서는 놀랍게도, 다른 프레임워크처럼 이를 쉽게 해주는 operator 가 부족하다. `share()` operator 를 사용 할 수 있지만, 결과를 받기 전에 모든 subscriber 에 구독해야 하기 때문에 다루기 어렵다.

캐싱 메카니즘 사용외에도, `multicast()` operator 를 사용하면 Subject 를 통해 값을 publish 하는 `ConnectablePublisher` 를 생성한다. 이것은 subject 에 여러번 구독하게 해주고, 준비가 되었을 때 publisher 의 `connect()`  메소드를 호출 한다.

```swift
let url = URL(string: "https://www.raywenderlich.com")!
let publisher = URLSession.shared
  .dataTaskPublisher(for: url)
  .map(\.data)
  .multicast { PassthroughSubject<Data, URLError>() }

let subscription1 = publisher
  .sink(receiveCompletion: { completion in
    if case .failure(let err) = completion {
      print("Sink1 Retrieving data failed with error \(err)")
    }
  }, receiveValue: { object in
    print("Sink1 Retrieved object \(object)")
  })

let subscription2 = publisher
  .sink(receiveCompletion: { completion in
    if case .failure(let err) = completion {
      print("Sink2 Retrieving data failed with error \(err)")
    }
  }, receiveValue: { object in
    print("Sink2 Retrieved object \(object)")
  })

let subscription = publisher.connect()
```

이 코드로, 요청을 한번 보내고 두개의 subscriber 에게 두개의 결과를 공유한다.
<br />



> Note : 모든 `Cancellable` 을 저장 해야 한다. 그렇지 않으면 특정 상황에서 즉시 현재 코드 스코프를 나갈때 해제되고 취소된다.

<br />


## 🔑 Key points

- Combine 은 `dataTask(with:completionHandler:)`를 publisher-based 추상화한 `dataTaskPublisher(for:)` 메소드를 제공한다.
- 내제된 `decode` operator 를 사용해서 Publisher 가 emit 한 `Data` 값을  `Codable` 을 conform 한 모델을 디코딩 할 수 있다.
- 여러 subscriber 에 구독의 재생을 공유 하는 operator 는 없지만, `ConnectablePublisher` 와 `multicast` operator 를 사용해서 다시 만들 수 있다.
