# 實作- 如何建立iOS12 shortcuts 

- [Siri Shortcuts Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)

- [Soup Chef: Accelerating App Interactions with Shortcuts](https://developer.apple.com/documentation/sirikit/soup_chef_accelerating_app_interactions_with_shortcuts)
- [Building for Voice with Siri Shortcuts](https://developer.apple.com/videos/play/wwdc2018/214/)
- [A beginner’s guide to developing Custom Intent Siri Shortcuts for iOS 12](https://medium.com/@pietropizzi/a-beginners-guide-to-developing-custom-intent-siri-shortcuts-for-ios-12-a3627b7011af)



## 流程

- 開啟Siri Capability

![1](https://github.com/htaiwan/study_note/blob/master/Assets/1.png)

- 新增Target
    - 選擇“Intents Extension”。
    - 預設 “Include UI Extension” 勾選。 
    - 接受 wizards的建議啟用相關的scheme。

- Intent Definition File
    - 在project中新增“SiriKit Intent Definition File”

        - 確保需要此檔案的target，都在此檔案的target membership被勾選了，且預設"Intent Classes"。

      ![2](https://github.com/htaiwan/study_note/blob/master/Assets/2.png)

        - **(雷！！)這步很重要，因為Xcode會自動產生對應所需的class到對應的target中**
        
            - 這裡真的很雷，有時候會突然發生Xcode就是莫名抓不到這些自動產生的class，一直編譯錯誤。
            - [stackoverflow](https://stackoverflow.com/questions/51054236/the-auto-generated-code-for-an-intent-defined-in-a-intentdefinition-throws-no)
            - 解決方式是在project settings中，找到 Intent Class Generation Language 然後設定成 Swift
        
          ![3](https://github.com/htaiwan/study_note/blob/master/Assets/3.png)

  - 建立自己的客製化intent

    - 根據自己的需求定義所需要的intent的內容。

      - **(雷！！)Intent的命名，不需要再用intent做結尾，因為Xcode會自動在你的命名名稱後再補上intent，然後就會發現所產生class名稱XXIntentIntent。**

      - 至於Description，我到現在還是不清楚是用在哪裡，煩請高手指教。

        ![4](https://github.com/htaiwan/study_note/blob/master/Assets/4.png)

- Intent的貢獻

  -  簡單寫個extension，讓所有view controller都有能力可以簡單置放shortcut Button，並貢獻intent。
  -  **(Bridge ！！)**這一步需要把我寫好Swift extention bridge到oc view controller
     -  加上`@objc`
     -  `@available(iOS 12.0, *)`，因為shortcut function只有ios12才有支援。
     -  在需要的oc檔案裡，`#import "ECAuctionApp-Swift.h"`
     -  接下來就是快樂使用swift extension

- 處理Intent Extension

  -  這裡是精華，畢竟Extension才是讓app更靈活被使用的方式。

  -  在新增target的那一步驟後，Xcode會自動幫我們生成這個檔案 “IntentHandler.swift”。

     -  修改 `handler(for:)`，確保只處理我們所定義的intent
     -  ECAuctionBiddingIntent就是Xcode自動生成的很雷的class，有時候就是很莫名的找不到，編譯錯誤。

     ```swift
         override func handler(for intent: INIntent) -> Any {
             guard intent is ECAuctionBiddingIntent else {
                 fatalError("無法處理未定義的intent: \(intent)")
             }
     
             return ECAuctionBiddingIntentHandler()
         }
     ```

     - ECAuctionBiddingIntentHandler是我們需要自己建立的一個class中，建立在Intent Extension這個target中，主要是要實作ECAuctionBiddingIntentHandling這個protocol，而這個很雷protocol也是Xcode自動產生的。

       - 這個protocol定義兩個metod，  `handle(intent:completion:)` 和`confirm(intent:completion)` ，handle是必須要實作，confirm是不一定。

       - 在handle中通常都是串API取得需要的資料回來，在WWDC影片中，將串API的動作跟回傳的內容都包成framework，好讓不同的target可以共用，但這裡對大多數的公司的現成專案就是痛苦的開始，讓我一一到來。

         - 我們公司是透過cocoapods管理第三方的跟公司內部專用的套件，並不是透過Xcode建立framework方式來管裡，而是要修改podfile，在podfile新增剛剛兩個target，並透過pod安裝需要套件到target中。在執行pod intall又出現了另外[Xcode的雷]([Xcode 10 Error: Multiple commands produce](https://stackoverflow.com/questions/50718018/xcode-10-error-multiple-commands-produce))，辛苦搞定cocoapods問題，接下來面就是另外一個問題。

         - **(Bridge ！！)**, OC bridge Swift，因為原本專案是OC，新增的target是Swift，如果想要直接使用原本target中已經寫好的service勢必須要處理bridge的問題。

           - 新增一個bridge檔案到原本專案target中，就在原本專案下建立一個Swift檔案，就會有一個建立視窗跳出來，按下確定後就會自產生一個bridge檔案。`ECAuctionApp-Bridging-Header.h`。
           - 在header中把等會swift中所需要oc class中都放進去，我這裡把service和service所對應的需要的data model都放進去，這真的是苦工啊。
           - **(雷！！) 不僅僅寫import檔案，還記得要到對應class中的Target membership把所需的target勾選。**
           - **(雷！！) 記得到不同target下的build setting下的Objective-C Bridging Header指向正確bridge file的路徑。**

           ![5](https://github.com/htaiwan/study_note/blob/master/Assets/5.png)

       - 最後，終於可以在extension中重複使用原本專案target已經寫好的service跟data model，上面這步真是重點的重點，不然這個extension就一點都沒用了。

         ```swift
           func handle(intent: ECAuctionBiddingIntent, completion: @escaping (ECAuctionBiddingIntentResponse) -> Void) {
                 ECShortcutClient.sharedInstance.getListingWithId(listingId: intent.listingId!) { (listing, error) in
                     if error != nil {
                         let failure = "對不起，目前查不到資料"
                         completion(ECAuctionBiddingIntentResponse.failure(failure: failure))
                     }
         
                     let ealisting = listing as! EAListing
                     var success = ""
         
                     guard let currentPice = ealisting.currentPrice,
                         let user = ealisting.bid.highestBidders.first else {
                             success =  "目前出價\(ealisting.currentPrice!)元，最高出價者無"
                             completion(ECAuctionBiddingIntentResponse.success(success: success))
                             return;
                     }
         
                     success = "目前出價\(currentPice)元，最高出價者 \((user as! EAUser).nickname ?? "無")"
                     completion(ECAuctionBiddingIntentResponse.success(success: success))
                 }
             }
         ```

- 再來就是處理Intent UI這個target

  - 這步相對簡單，沒有什麼雷，重複使用上一步辛苦搞定可以使用的service。

  - 只是要注意在apple的文件中有說過，這個UI不提供任何互動，也是說裡面不可以放button可以點選或者tableview可以滾動。唯一可以做的就是點選整個UI喚起你的app。

    > - However, your view controllers do not receive touch events or any other events while they are onscreen, and you cannot add gesture recognizers to them. Therefore, never create an interface with controls or views that require user interactions.
    > - [Requirements and Limitations](https://developer.apple.com/documentation/sirikit/creating_an_intents_ui_extension/configuring_the_view_controller_for_your_custom_interface?language=objc)

  - 這個UI只能單純提供訊息給使用者吧了。

```swift
  func configureView(for parameters: Set<INParameter>, of interaction: INInteraction, interactiveBehavior: INUIInteractiveBehavior, context: INUIHostedViewContext, completion: @escaping (Bool, Set<INParameter>, CGSize) -> Void) {
        // Do configuration here, including preparing views and calculating a desired size for presentation.

        guard  interaction.intent is ECAuctionBiddingIntent else {
            completion(false, Set(), .zero)
            return
        }

        let width = self.desiredSize.width
        let desiredSize = CGSize(width: width, height: 110)

        activiityIndicator.startAnimating()
        // fetch API
        ECShortcutClient.sharedInstance.getListingWithId(listingId: (interaction.intent as! ECAuctionBiddingIntent).listingId!) { [weak self] (listing, error) in

            self?.activiityIndicator.startAnimating()
            self?.activiityIndicator.isHidden = true

            let ealisting = listing as! EAListing
            self?.listingTitleLabel.text = ealisting.title

            if let first = ealisting.bid.highestBidders.first {
                let user = first as! EAUser
                self?.listingBiddingInfoLabel.text = "目前出價：\(ealisting.currentPrice!)\n最高出價者： \(user.nickname ?? "無")"
            } else {
                self?.listingBiddingInfoLabel.text = "目前出價：\(ealisting.currentPrice!)\n最高出價者：無"
            }

            if let image  = (listing as! EAListing).images.first {
                let urlString = image.getUrlOfRule("resize00")
                let url = URL(string: urlString!)
                let data =  try? Data(contentsOf: url!)
                self?.listingImageView.image = UIImage(data: data!)
            }
        }

        completion(true, parameters, desiredSize)
    }
```



- 最後，就是處理喚起APP的互動
  - 在appdelege中這個method中取回intent的資訊，並進行相關的處理`- (BOOL)application:(UIApplication *)application continueUserActivity:(nonnull NSUserActivity *)userActivity restorationHandler:(nonnull void (^)(NSArray<id<UIUserActivityRestoring>> * _Nullable))restorationHandler`