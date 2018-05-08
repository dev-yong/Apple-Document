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

  AVAsset과 AVAssetTrack은 **AVAsynchronousKeyValueLoading protocol** (property의 현재의 loaded state를 묻고 asynchronously 로드)를 사용합니다.

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

    ​