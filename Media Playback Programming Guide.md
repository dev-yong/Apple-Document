## Exploring AVFoundation

- ### Understanding the Asset Model

  AVFoundation : Media asset의 재생 및 처리, AVAsset 클래스를 사용

  - **AVAsset**
    - **로컬 파일 기반 미디어**(QuickTime동영상 또는 MP3오디오 파일)를 모델링 할 수 있으며, 원격 호스트에서 다운로드하거나 **HTTP 실시간 스트리밍**(HLS)을 사용하여 스트리밍되는 asset
    - media format에 대한 독립성을 제공. 일관된 인터페이스
    - media의 위치로부터 독립성을 제공. **URL로 초기화** 하여 생성
    - AVAssetTrack인스턴스로 구성된 컨테이너 개체
    - Video, Audio, Subtitles


- ### Creating an Asset

  ```swift
  var anAsset = AVURLAsset(url: url, options: nil)
  ```

  HLS스트림에 대한 자산을 생성하는 경우 사용자가 셀룰러 네트워크에 연결되어 있을 때 해당 미디어가 해당 미디어를 검색하지 못하도록 할 수 있습니다. `AVURLAssetAllowsCellularAccessKey`

  ```swift
  let url: URL = // Remote Asset URL
  let options = [AVURLAssetAllowsCellularAccessKey: false]
  let asset = AVURLAsset(url: url, options: options)
  ```


- ### Preparing Assets for Use

  unloaded property value를 검색하는 요청은 오랫동안 블락되면,  타임아웃이 발생합니다. 이러한 일을 방지하기 위해, asset properties를 **asynchronously 로드**해야합니다.

  AVAsset과 AVAssetTrack은 AVAsynchronousKeyValueLoading protocol (property의 현재의 loaded state를 묻고 asynchronously 로드)를 사용합니다.

  ```swift
  public func loadValuesAsynchronously(forKeys keys: [String], completionHandler handler: (() -> Void)?)
  public func statusOfValue(forKey key: String, error outError: NSErrorPointer) -> AVKeyValueStatus
  ```

  ```swift
  asset.loadValuesAsynchronously(forKeys: [playableKey]) {
      var error: NSError? = nil
      let status = asset.statusOfValue(forKey: playableKey, error: &error)
      switch status {
      case .loaded:
          // Sucessfully loaded. Continue processing.
      case .failed:
          // Handle error
      case .cancelled:
          // Terminate processing
      default:
          // Handle all other cases
      }
  }
  ```

  `statusOfValueForKey:error:`를 사용하여 완료 콜백에서 속성 상태를 확인합니다.

   모든 경우에 완료 콜백은 임의의 백그라운드 대기 열에서 호출됩니다.


- ### Working with Metadata

  Media container format은 미디어 설명 메타데이터를 저장할 수 있습니다. AVFoundation은 AVMetadataItem 클래스를 사용하여 메타 데이터 작업을 간단하게 하였습니다.

  AVMetadataItem은 싱글 메타데이터 값(ex. 영화 제목, 앨범 아트 워크 등)을 나타내는 key-value 쌍입니다. 

  - #### Retrieving a Collection of Metadata

    AVMetadataItem을 효과적으로 사용하려면 AVFoundation에서 메타 데이터를 구성하는 방법을 이해해야 합니다. 메타 데이터 항목의 찾기 및 필터링을 간소화하기 위해 프레임워크는 관련 메타 데이터를 key space(주요 공간? key 공간?)에 그룹화합니다.

    - **Format-specific key spaces** : 프레임워크는 여러가지 Format-specific key space를 정의합니다. QuckTime 또는 MP3와 같은 특정 컨테이너 혹은 파일 형식과 연관되어있습니다. 하지만, 단일 asset은 여러 key space의 메타 데이터 값을 포함할 수 있습니다.

    - **Common key space** : 여러 key space에 존재할 수 있는 영화의 제작 날짜나 설명과 같은 여러가지 일반적인 메타 데이터 값이 있습니다. 프레임워크는 여러 주요 공간에 공통된 제한된 메타 데이터 값 집합에 대한 액세스를 제공하는 공통된 key space를 제공합니다. commonMetadata속성을 사용하여 asset의 공통 메타 데이터의 컬렉션을 검색할 수 있습니다.

      ```swift
      let url = Bundle.main.url(forResource: "audio", withExtension: "m4a")!
      let asset = AVAsset(url: url)
      let formatsKey = "availableMetadataFormats"
      asset.loadValuesAsynchronously(forKeys: [formatsKey]) {
          var error: NSError? = nil
          let status = asset.statusOfValue(forKey: formatsKey, error: &error)
          if status == .loaded {
              for format in asset.availableMetadataFormats {
                  let metadata = asset.metadata(forFormat: format)
                  // process format-specific metadata collection
              }
          }
      }

      ```

      `availableMetadataFormats` 는 메타데이터 형식의 문자열 식별자의 배열을 반환합니다.

  - #### Finding and Using Metadata Values

    메타데이터 컬렉션을 검색한 후, 다음 단계는 관심있는 특정 값을 찾는 것입니다. AVMetadataItem의 다양한 클래스 메소드를 사용하여 메타데이터 데이터를 분리된 값들의 집합으로 필터링할 수 있습니다.

    다음 예는 공통된 key space에서 제목 항목을 검색하는 방법입니다.

    ```swift
    let metadata = asset.commonMetadata
    let titleID = AVMetadataCommonIdentifierTitle
    let titleItems = AVMetadataItem.metadataItems(from: metadata, filteredByIdentifier: titleID)
    if let item = titleItems.first {
        // process title item
    }
    ```

    특정 메타 데이터 아이템을 검색한 후, 다음 단계는 value property를 호출하는 것입니다. NSObject와 NSCopying protocol을 사용하는 객체 타입을 반환합니다. 값을 적절한 타입으로 직접 캐스팅할 수 있지만, 메타데이터 아이탬의 타입 강제 속성을 사용하는 것이 더 안전하고 쉽습니다.

    다음 예는 iThunes 음악 트랙와 관련된 아트워크를 검색하는 방법입니다.

    ```swift
    let metadata = asset.commonMetadata
    // Filter metadata to find the asset's artwork
    let artworkItems =
        AVMetadataItem.metadataItems(from: metadata,
                                     filteredByIdentifier: AVMetadataCommonIdentifierArtwork)
    if let artworkItem = artworkItems.first {
        // Coerce the value to an NSData using its dataValue property
        if let imageData = artworkItem.dataValue {
            let image = UIImage(data: imageData)
            // process image
        } else {
            // No image data found
        }
    }
    ```

- ### <u>Playing Media</u>

  미디어를 재생하는 데 필요한 추가 객체들에 대해 설명하고 재생을 위해 그것들을 구성하는 방법에 대하여 보여 줍니다

  ![image-20180509085808899](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180509085808899.png)

  - #### AVPlayer 

    재생하는 케이스를 주도하는 central 클래스입니다. player는 미디어 asset의 **재생과 타이밍을 관리**하는 컨트롤러 객체 입니다.

    순차적으로 미디어를 재생할 미디어 asset의 큐를 생성하고 관리하기 위해서는 AVQueuePlayer를 사용합니다.

  - #### AVPlayerAsset

    AVAsset은 미디어의 정적인 면만 모델링하기 때문에 AVPlayer를 이용한 재생에는 적합하지 않습니다. asset을 재생하기 위해서는 AVPlayerItem을 사용합니다. AVPlayerItem은 AVPlayer에 의해 재생되는 **asset의 타이밍과 프레젠테이션 상태**를 모델링합니다. 

  - #### AVKit and AVPlayerLayer

    AVPlayer및 AVPlayerItem은 비시각적인 개체이며 단독으로 자산의 비디오를 화면에 표시할 수 없습니다. 

    - **AVKit** : iOS 또는 tvOS에 있는 AVKit 프레임워크의 AVPlayerViewController를 사용하는 것이 동영상을 표시하는 가장 좋은 방법입니다.
    - **AVPlayerLayer** : 플레이어의 커스텀한 인터페이스를 구성하는 경우, AVFoundation에서 제공하는 CALayer의 하위 클래스인 AVPlayerLayer를 사용합니다. 단순 시각적 콘텐츠를 화면에 표시합니다.

    ```swift
    class PlayerViewController: UIViewController {
     
        @IBOutlet weak var playerViewController: AVPlayerViewController!
     
        var player: AVPlayer!
        var playerItem: AVPlayerItem!
     
        override func viewDidLoad() {
            super.viewDidLoad()
     
            // 1) Define asset URL
            let url: URL = // URL to local or streamed media
     
            // 2) Create asset instance
            let asset = AVAsset(url: url)
     
            // 3) Create player item
            playerItem = AVPlayerItem(asset: asset)
     
            // 4) Create player instance
            player = AVPlayer(playerItem: playerItem)
     
            // 5) Associate player with view controller
            playerViewController.player = player
        }
     
    }
    ```

- ### <u>Observing Playback State</u>

  AVPlayer와 AVPlayerItem은 상태가 자주 변경되는 동적인 객체입니다. **Key-Value Observing(KVO)를 사용**하여 상태 변경을 쉽게 관찰하고 변화에 대한 반응을 취할 수 있습니다.

  가장 중요하게 관찰해야하는 AVPlayerItem의 속성은  **status**입니다. status는 플레이어 **아이탬의 재생 준비**와 **사용할 수 있는 지**를 나타냅니다.

  1. 최초 생성 시, `AVPlayerItemStatusUnknown`으로 지정되며, **미디어가 로드되어지지 않았**거나 **재생을 위한 대기열에 있지 않음**을 뜻합니다.
  2. 플레이어 아이탬을 AVPlayer와 연관지으면, 즉시 아이태ㅐㅁ의 미디어는 재생 대기열에 추가되고 재생 준비를 시작합니다.
  3. `AVPlayerItemStatusReadyToPlay`로 변경되면 플레이어 아이탬을 **사용할 준비**가 됩니다.

  `addObserver:forKeyPath:options:context:` 를 사용하여 플레이어 아이탬의 status를 관찰하도록 등록합니다.

  ```swift
  let url: URL = // Asset URL
   
  var asset: AVAsset!
  var player: AVPlayer!
  var playerItem: AVPlayerItem!
   
  // Key-value observing context
  private var playerItemContext = 0
   
  let requiredAssetKeys = [
      "playable",
      "hasProtectedContent"
  ]
   
  func prepareToPlay() {
      // Create the asset to play
      asset = AVAsset(url: url)
   
      // Create a new AVPlayerItem with the asset and an
      // array of asset keys to be automatically loaded
      playerItem = AVPlayerItem(asset: asset,
                                automaticallyLoadedAssetKeys: requiredAssetKeys)
   
      // Register as an observer of the player item's status property
      playerItem.addObserver(self,
                             forKeyPath: #keyPath(AVPlayerItem.status),
                             options: [.old, .new],
                             context: &playerItemContext)
   
      // Associate the player item with the player
      player = AVPlayer(playerItem: playerItem)
  }
  ```

  `observeValueForKeyPath:ofObject:change:context: `를 구현하여 status의 변화에 대한 알림을 받습니다.

  ```swift

  override func observeValue(forKeyPath keyPath: String?,
                             of object: Any?,
                             change: [NSKeyValueChangeKey : Any]?,
                             context: UnsafeMutableRawPointer?) {
   
      // Only handle observations for the playerItemContext
      guard context == &playerItemContext else {
          super.observeValue(forKeyPath: keyPath,
                             of: object,
                             change: change,
                             context: context)
          return
      }
   
      if keyPath == #keyPath(AVPlayerItem.status) {
          let status: AVPlayerItemStatus
          if let statusNumber = change?[.newKey] as? NSNumber {
              status = AVPlayerItemStatus(rawValue: statusNumber.intValue)!
          } else {
              status = .unknown
          }
          // Switch over status value
          switch status {
          case .readyToPlay:
              // Player item is ready to play.
          case .failed:
              // Player item failed. See error.
          case .unknown:
              // Player item is not yet ready.
          }
      }
  }
  ```

  ​