# 重構- API Clients的美麗與哀愁

------

### 前言

api的串接，我想是每個app幾乎都會用到，越複雜的app或者畫面串接api的數目就越多，因此把api串接抽離出來成一個獨立的class來進行處理，也是理所當然，但要如何設計一個好的class來應付日漸龐大的api數目？

------

### Clients的哀愁

先來看看，目前工作上專案在client設計出了哪些問題

- 違背了Open Close Principle

  - 每次新增api時，都要修改client的介面，新增一個method,。一個好的設計應該對close“修改”，open"擴充"
  - 不應該在client反覆的修改，已達到擴充的目的。

- 違背了Dependency Inversion Principle

  高層模組(client)過度依賴底層模組(data model)，每次api返回的資料，都會先轉成data model，因此不同的api串接都有可能依賴到不同的data model，導致client亦賴過多data model, 有太多的dependency了。

  這樣壞處就是，就是不易重複使用， 如果哪天要在不同的專案或不同的target重複使用這個client的某個method時，卻需要把其他一堆沒用到data object都要複製到另外一邊去。

  ------

  ### Clients的美麗

  - Open Close Principle - Client應該是由抽象搭建出來的框架。
    - 透過實作框架來達到擴充的目的。
  - Dependency Inversion Principle- 高層模組(client)跟底層模組(data model)都要依賴抽象。
  - 透過抽象來當作橋樑，連接clinet跟data model之間的關聯性，又可以減少clinet跟data model之間的耦合性，讓整個系統更易於擴充，也更易維護跟重複使用。

  ------

  ### 例子

  這是目前client的.h中宣告的部分method, 每次要串接新的api就會在這.h中，新增一個method。然後再import這個method所需要的data model。所以就會看到這個client import了很多很多的data model。造就一個怪獸級的client，也很難重複使用。

![Screen Shot 2018-11-20 at 11.09.04 AM](https://github.com/htaiwan/study_note/blob/master/Assets/8.png)

![Screen Shot 2018-11-20 at 11.09.04 AM](https://github.com/htaiwan/study_note/blob/master/Assets/9.png)

#### 	UML

目前client的架構圖，每次在client增加新method，都有可能增加新的data model的dependency。

![Screen Shot 2018-11-20 at 11.09.04 AM](https://github.com/htaiwan/study_note/blob/master/Assets/6.png)

#### 修改

- Client只有一個method，這個method只依賴抽象。(EAYQLRequest, EADataContainer都是抽象)

```swift
public func send<T: EAYQLRequest>(_ request: T, completion: @escaping ResultCallback<EADataContainer<T.Response>>) 
```

- 擴充這個Client，不在是直接修改這個client，而是新增一個class來實作抽象。

```Swift
public struct EAGetOrders: EAYQLRequest {
    public typealias Response = EAOrederList
    
    public var query: YQLQuery? {
        let table = "ecauction.orders(\(range!.location), \(range!.length))"
        let select = YQLQuery.select(from: table)
        let wherePredict = (criteria.dictionary as NSDictionary).predicate(withIdentifierName: nil)
        
        return select?.where(wherePredict)
    }
    
    public let criteria: EAOrderCriteria
    public let range: NSRange?

    public init(criteria: EAOrderCriteria,
                range: NSRange? = NSRange(location: 0, length: 10)) {
        self.criteria = criteria
        self.range = range
    }
}
```

- 所有的response的data model，都是依賴EADataContainer，實作Decodable。

```Swift
public struct EADataContainer<Results: Decodable>: Decodable {
    public let count: Int?
    public let created: String?
    public let lang: String?
    public let results: Results
}
```

- 使用方式，由於client的send是依賴抽象，所以只要根據傳入的不同的實作，就可以達到呼叫不同api的目的。

```Swift
        EAIntentYQLClient.sharedInstance.send(EAGetOrders(criteria: criteria)) { (response) in
            switch response {
            case .success(let dataContainer):
                var amount = 0
                for order in dataContainer.results.orderList.order {
                    if order.payment.status == EAOrederPaymentStatus.canceled.rawValue {
                        amount += order.summary.finalTotalPrice
                    }
                }
                completion(EACalculateOrderIntentResponse.success(success: self.getSuccessString(amount: amount)))
            case .failure(let error):
                completion(EACalculateOrderIntentResponse.failure(failure: "網路不穩，請稍後再試"))
            }
        }
    }
```

#### 	UML	

- 現在高層模組client只依賴抽象，底層模組EAGetOrders, EAorderList也是依賴抽象。如此一來，高層模組跟底層模組就解耦。
- 每次新增API,都是新增Class(EAGetOrders)並實作EAYQLRequest，而不是直接修改client的介面。
- 產生新的data model(EAOrederList) , 也是透過EADataContainer這個抽象跟client產生關係。
- 這樣設計的好處，一但將來如果想要重複使用這個client中某個api，只需要將client跟抽象介面，以及所需要的api class跟data model拿過去直接使用，而不會像之前的client綁了一大堆其他不需要的method跟data model。

![Screen Shot 2018-11-20 at 2.32.19 PM](https://github.com/htaiwan/study_note/blob/master/Assets/7.png)
