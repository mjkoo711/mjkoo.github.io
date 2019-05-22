### 테스트 환경

- Xcode 10.2.1
- Swift 5
- iOS 12.2

# #1 - Basic

---

### RxSwift

- Swift로 작성된 Reactive Extensions 버전
- 변수에 정적으로 값을 할당하는 것 ⇒ 미래에 바뀔 수 있는 것을 관찰(observe)하는 것

### 이해 돕기 위한 예문

스마트폰은 관찰이 가능**(observable)**합니다. 스마트 폰은 페이스북 알림, 메시지, 스냅챗 알림 등과 같이 신호**(signal)**를 방출합니다. 우리는 자연적으로 스마트폰을 구독**(subscribe)**하고 있고, 모든 알림을 홈 스크린에서 확인할 수 있습니다. 이제 그 신호**(signal)**로 무엇을 할 지 정할 수 있습니다. 우리는 관찰자**(observer)**입니다. 

### 순서

- Step 1. [프로젝트](https://github.com/mjkoo711/RxSwift-Example/)를 다운로드 하여 해당 폴더에서 터미널을 열고 [example/1 브랜치](https://github.com/mjkoo711/RxSwift-Example/tree/example/1)로 전환
- Step 2. **pod install** 명령어 실행
- Step 3. 폴더 내 **RxSwiftExample.xcworkspace** 오픈
- Step 4. ViewController.swift 내의 viewDidLoad()에 아래의 코드를 입력

        searchBar
              .rx.text // RxCocoa의 Observable 속성
              .orEmpty // optional이 아니게 만듬 (String? => String)
              .subscribe(onNext: { [unowned self] query in // 이 코드로 인해 모든 새로운 값에 대한 알림을 받을 수 있다
                self.showCities = self.allCities.filter { $0.hasPrefix(query) } // 도시를 찾기 위해 "API 요청" 작업을 수행
                self.tableView.reloadData()
              }).disposed(by: disposeBag)

    ⇒ 이 코드를 통해 반응형으로 테이블 뷰가 새로고침 되는 것을 확인할 수 있다.

    - **Subscribe**함수를 이용해서 **"Signal을 발생하는 Observable 속성을 구독"**할 수 있다. (onNext 이외에도 onError, onCompleted 등을 구현할 수 있다)
    - disposeBag는 dinit과정에서 구독 해제하려는 모든 것을 보관하고 있는 객체

- Step 5. 과도한 API 요청, 빈 키워드, 딜레이 들에 대해 대응하려면?
    - 시나리오를 하나 가정해보자

        "O"를 입력해서 검색 결과가 나왔다. 그런 다음 "Oc"를 입력한 후에 마음이 바뀌어서 딜레이 끝나고 API 요청이 시작되기 전에 "O"로 바뀌었다고 가정해보자. 그 경우 똑같은 요청을 두번 할 것이다. Rx없이 이 작업을 하려면 플래그와 최근 검색한 쿼리를 추가하고 새로운 것과 비교 ... ⇒ 로직 비대해짐

    - 그럼 어떻게 할 수 있을까? RxSwift에서는 두 줄의 코드로 작업을 할 수 있음
        - debounce() : 주어진 스케줄러에 맞춰 딜레이를 만듦
        - distinctUntilChanged() : 같은 값을 입력하는 것을 막아줌

        [Step 4]에 코드를 추가해보자.

            searchBar
                  .rx.text // RxCocoa의 Observable 속성
                  .orEmpty // optional이 아니게 만듬 (String? => String)
                  .debounce(RxTimeInterval.seconds(Int(0.5)), scheduler: MainScheduler.instance)
                  .distinctUntilChanged() // 새로운 값이 이전과 같은지 확인
                  .subscribe(onNext: { [unowned self] query in
                    self.showCities = self.allCities.filter { $0.hasPrefix(query) }
                    self.tableView.reloadData()
                  }).disposed(by: disposeBag)

- Step 6. 빈 값에 대해서 API 요청을 막으려면?
    - filter() 함수를 사용하자. 아래의 코드 참고!

        searchBar
              .rx.text // RxCocoa의 Observable 속성
              .orEmpty // optional이 아니게 만듬 (String? => String)
              .debounce(RxTimeInterval.seconds(Int(0.5)), scheduler: MainScheduler.instance)
              .distinctUntilChanged() // 새로운 값이 이전과 같은지 확인
              .filter{ !$0.isEmpty } // 빈 값일 때는 필터로 막음
              .subscribe(onNext: { [unowned self] query in
                self.showCities = self.allCities.filter { $0.hasPrefix(query) }
                self.tableView.reloadData()
              }).disposed(by: disposeBag)

# #2 - Observable & Bind

---

### 시작하기 전에

- **Subject**

    Observable & Observer를 한꺼번에 부르는 용어 

- **BehaviorSubject**

    구독하면 Subject에 의해 반환한 가장 최근 값을 가져오고, 구독 **이후**에 반환하는 값을 가져옴

- **PublishSubject**

    구독하면 구독 **이후**에 반환하는 값을 얻게 됨

- **ReplaySubject**
    - 구독하면 구독 이후에 반환하는 값 뿐만 아니라, 구독 이전에 반환한 값을 가져옴.
        - ReplaySubject의 버퍼 사이즈에 따라 이전의 값을 얼마나 가져올지 알 수 있음.
        - 버퍼 사이즈는 Subject의 초기화 할 때 알 수 있음.

- **Variable ⇒** Swift 5에서는 이 타입으로 선언이 안됨 위의 subject 중 하나로 선택해야함

    BehaviorSubject를 감싸는 onNext() 이벤트만 제공하는 Wrapper. (BehaviorSubject를 사용하면 onError(), onCompleted() 전송도 가능) 

    또한, Varible은 할당 해제될 때 자동으로 onCompleted 이벤트를 보냄

### 시나리오

원의 위치에 따라서 배경 뷰의 색이 다르게 보여지는 것을 만들 것

- Step 1. [프로젝트](https://github.com/mjkoo711/RxSwift-Example/)를 다운로드 하여 브랜치를 [example/2](https://github.com/mjkoo711/RxSwift-Example/tree/example/2) 브랜치로 전환
- Step 2. 새로운 변수를 UI와 관련된 것들을 계산할 때 사용될 **ViewModel**에서 생성할 것

    ⇒ 이렇게 되면 새로운 위치를 받을 때마다, 원의 배경 색을 계산할 수 있음

- Step 3. 어떻게 ViewModel을 구성할까? (2개 속성)
    - **centerVariable** : 데이터를 저장하고 가져오는데 사용 (observer, observable 역할)
    - **backgroundColorObservable** (실제 Variable은 아니고 그저 Observable)

    - 의문점
        - 왜 centerVariable은 Variable인데, backgroundColorObservable은 Observable일까?
            - ~~관찰 가능한 원의 중앙은 centerVariable과 연결되어 있다. 이 말은 시간이 지나서 중앙 위치가 바뀌면, centerVariable도 변한다는 것을 의미한다. 그래서 그것은 Observer이다. 또한, ViewModel에서는 centerVariable을 Observable로 사용하는데, 이것은 Observer와 Observable 둘 다로 사용하는 것이기 때문에 그냥 Subject이다. 그렇다면 왜 PublishSubject나 ReplaySubject가 아닌 Variable일까? 그 이유는 단지 우리가 구독한 원의 가장 최근 중앙 지점만을 필요로 하기 때문~~
            - backgroundColorObservable은 그저 Observable이다. 다른 것들에 영향을 주ㅜ지 않기 때문에.
- Step 4. ViewModel 작성

        import Foundation
        import ChameleonFramework
        import RxSwift
        import RxCocoa
        
        class CircleViewModel {
          var centerVariable = BehaviorSubject<CGPoint?>(value: .zero)
          var backgroundColorObservable: Observable<UIColor>!
        
          init() {
            setup()
          }
        
          func setup() {
        
          }
        }

- Step 5. ViewModel의 setup() 작성

        func setup() {
            backgroundColorObservable = centerVariable.asObservable().map { center in
              guard let center = center else { return UIColor.flatten(.black)() }
        
              let red: CGFloat = ((center.x + center.y).truncatingRemainder(dividingBy: 255.0)) / 255.0 // We just manipulate red, but you can do w/e
              let green: CGFloat = 0.0
              let blue: CGFloat = 0.0
        
              return UIColor.flatten(UIColor(red: red, green: green, blue: blue, alpha: 1.0))()
            }
          }

    1. Variable을 Observable로 변경해야함. Variable은 Observer와 Observable 둘 다 될 수 있기 때문에, 하나를 결정해야 함. 우리는 이것을 **관찰**하고 싶기 때문에, asObservable()로 변경함. 
    2. 모든 새로운 CGPoint의 값을 UIColor로 연결함. 우리는 Observable이 생성하는 새로운 중앙 값을 받게되고, 로직을 통해 새로운 UIColor를 만듬
    3. Observable이 옵셔널 CGPoint이다. 그래서 guard let 사용해서 예외처리로 검은색 반환 
- Step 6. 이제 원의 위치에 따라 새로운 배경 색을 반환하는 Observable을 가지게 됨. 이젠 새로운 값을 원에 업데이트 하는 것 (Observable을 subscribe()할 것)

    ViewController의 setup()함수에 추가 작성

        circleView
              .rx.observe(CGPoint.self, "center")
              .bind(to: circleViewModel.centerVariable)
        
        circleViewModel.backgroundColorObservable.subscribe(onNext: { [weak self] (backgroundColor) in
              UIView.animate(withDuration: 0.1, animations: {
                self?.circleView.backgroundColor = backgroundColor
                // 주어진 배경색의 상호 보완적인 색을 구합니다
                let viewBackgroundColor = UIColor(complementaryFlatColorOf: backgroundColor)
                if viewBackgroundColor != backgroundColor {
                  // 원의 배경색으로 새로운 배경색을 할당합니다
                  // 우린 그저 뷰에서의 원이 보일 수 있는 다른 색을 원합니다
                  self?.view.backgroundColor = viewBackgroundColor
                }
              })
            })

색상을 조정하는 모든 작업은 델리게이트, 노티피케이션 그리고 온갖 상용구 코드 없이 만들 수 있다. 

### 참고자료

[](https://pilgwon.github.io/blog/2017/09/26/RxSwift-By-Examples-1-The-Basics.html)
