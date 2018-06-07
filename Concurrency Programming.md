# Concurrent Programming With GCD in Swift 3

> https://developer.apple.com/videos/play/wwdc2016/720/

- ### **Main Thread**에서 User Interface의 모든 코드를 실행한다. 

  Data transform이나 image processing과 같은 작업을 main thread에서 진행하면, User Interface는 느려지거나 중지된다.

![image-20180607161856008](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607161856008.png)

![image-20180607162516697](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607162516697.png)

- ### Concurrency를 이용하자

  thread를 이용한다. 하지만, 코드의 불변성 유지가 어렵다.

  ![image-20180607162055807](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607162055807.png)
  - ### GCD(Grand Central Dispatch)

    - ##### Concurrency Library 

    - ##### DispatchQueue : to submit items of work to that queue

      - **c 기반** 메커니즘

      - Dispatch가 thread와 service가져온다.

      - Queue에 제출된 순서 = Dispatch에서 실행되는 순서 (Dispatch Queues excute FIFO)

      - submit work : asyncronous, syncronous

      - 다른 Dispatch Queue에서 transform 등을 처리한 후, data만 다시 main thread로 보낸다.

        ![image-20180607163048659](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607163048659.png)

        ```swift
        let queue = DispatchQueue(label: "com.example.imageTransform")
        
        queue.async {
            let smallImage = image.resize(to: rect)
            //DispatchQueue.main : Main Thread에서 실행하는 모든 항목 처리
            DispatchQueue.main.async {
                imageView.image = smallImage
            }
        }
        ```

- ### Structuring Your Application

  1. ##### Data flow 식별

  2. ##### Subsystem으로 나눈다.

  3. ##### 각 Subsystem에 Dispatch Queue를 부여한다.

  ![image-20180607163809964](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607163809964.png)

  ​	너무 많은 queue와 thread는 성능 저하의 주범이다.

  - ##### Asyncronous

    ![image-20180607163955267](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607163955267.png)

    ##### ![image-20180607164053644](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607164053644.png)

    `group.notify(queue: DispatchQueue.main){}` 
    : group에서의 작업이 완료되면, 선택한 queue에서 작업을 완료하도록 지시한다.

  - ##### Synchronous

    subsystem들을 직렬화로 처리한다.

    안전하게 property에 접근할 수 있다. (**mutual exclusion**) ex) mutex, semaphore
    하지만, **Deadlock**이 발생할 수도 있다.

    ![image-20180607164255911](/var/folders/q5/jd37stc93wnbg03qc25l_yc00000gp/T/abnerworks.Typora/image-20180607164255911.png)	

    ```swift
    class MyObject {
    	private let internalState: Int
    	private let internalQueue: DispatchQueue
    	var state: Int {
    		get {
    			return internalQueue.sync { internalState }
    		}
    		set (newState) {
    			internalQueue.sync { internalState = newState }
    		}
    	}
    }
    ```



# Concurrency Programming Guide

> https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1

- The term ***thread*** is used to refer to a separate path of execution for code. 
  코드 실행을 위한 <u>별도의 실행 경로</u>
- The term ***process*** is used to refer to a running executable, which can encompass multiple threads.
  여러 thread를 포함할 수 있는, 실행 파일(<u>running executable</u>)
- The term ***task*** is used to refer to the abstract concept of work that needs to be performed.
  수행해야할 <u>작업의 추상적 개념</u> 



> https://www.appcoda.com/grand-central-dispatch/