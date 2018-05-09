## Playback

asset들의 재생을 컨트롤하기 위해 AVPlayer객체를 사용해야합니다, 재생하는 동안 AVPlayerItem 인스턴스를 사용하여 asset의 프레젠테이션 상태를 전체적으로 관리할 수 있고, AVPlayerItemTrack 객체는 개별 트랙의 프레젠테이션 상태를 관리하기 위한 객체입니다.

- ### Playing Assets

  AVPlayer인스턴스를 사용하여 단일 자산을 재생할 수 있습니다. AVQueuePlayer개체를 사용하여 여러 항목을 순서대로 재생할 수 있습니다. (Media Playback Programming에도 나온 내용)

  싱글 AVPlayer인스턴스에서 많은 AVPlayerLayer개체를 만들 수 있지만 가장 최근에 만든 레이어만 비디오 콘텐츠를 화면에 표시합니다.

  ![avplayerLayer_2x](/Users/igwang-yong/Programming Guide/avplayerLayer_2x.png)

  여러 플레이어를 사용하여 지정된 자산을 동시에 재생할 수 있지만 각 플레이어가 다른 방식으로 렌더링 할 수 있음을 의미합니다. 예를 들어 항목 트랙을 사용하여 재생 중에 특정 트랙을 비활성화할 수 있습니다(예:사운드 구성 요소를 재생하지 않을 수 있음).

  ![playerObjects_2x](/Users/igwang-yong/Programming Guide/playerObjects_2x.png)


- ### Handling Different Types of Asset

  재생할 asset을 구성하는 방법은 file-based asset과 stream-based asset 두가지 유형이 있습니다.

  - File-based asset 

    1. AVURLAsset 생성


    2. asset을 이용해 AVPlayerItem 생성
    3. AVPlayerItem을 AVPlayer와 연관시킴
    4. 아이탬의 status가 준비될 때까지 기다림(KVO를 이용하여 status의 변화를 알림으로 받는다)


  - Stream-based asset

    ```swift
    playerItem = AVPlayerItem(url: url)
    playerItem.addObserver(self, forKeyPath: "status", options: [], context: ItemStatusContext)
    player = AVPlayer(playerItem: playerItem) 
    ```

    player item을 player와 연관시키면, 재생할 준비가 시작됩니다. 재생할 준비가 시작될 때, player item은 AVAsset과 AVAssetTrack을 생성합니다. 이 인스턴스를 사용하여 라이브 스트림의 내용을 검사할 수 있습니다.

    스트리밍 아이템의 duration을 얻기 위해, player item의 duration 속성을 관찰할 수 있습니다. item을 재생할 준비가 됬을 때, duration 속성은 올바른 값으로 업데이트됩니다.

    ```swift
    playerItem.tracks[0].assetTrack.asset?.duration
    ```

    **플레이어를 초기화한다고 해서 재생할 준비가 된 것은 아닙니다.** player의 재생할 준비가 되면 `AVPlayerStatusReadyToPlay`로 변하는 status 속성을 관찰해야합니다.  

    URL 종류를 모르는 경우 다음 단계를 수행합니다.

    1. URL을 사용하여 AVURLAsset을 초기화한 다음, track key를 로드합니다. track이 성공적으로 로드되면, player item을 생성합니다.
    2. '1.'이 실패한다면, AVPlayerItem을  URL을 이용해 직접 만듭니다. 재생이 가능한지 안한지, player의 status 속성을 관찰합니다.


- ### Playing an Item

  ```swift
  @IBAction func play(_ sender: Any) {
      player.play()
  }
  ```

  재생 속도 및 재생 헤드 위치와 같은 다양한 측면을 관리할 수 있습니다. 플레이어의 재생 상태를 모니터링할 수도 있습니다.

  - #### Changing the Playback Rate

    ```swift
    aPlayer.rate = 0.5;
    aPlayer.rate = 2.0;
    ```

  - #### Seeking—Repositioning the Playhead

    (Media Playback Programming에도 나온 내용)

    재생 후, player의 head는 item의 끝으로 설정되고, 추가로 재생을 요청해소 효고가 없습니다. 다시 item의 시작으로 playhead를 위치시키기 위하여 AVPlayerItemDidPlayToEndTimeNotification 알림을 받을 수 있습니다.