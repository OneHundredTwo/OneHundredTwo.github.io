# Non-Activex Player 개발일지

> ITX 근무중에 주요 업적이라고 할 수 있는 Non-Activex 플레이어 모듈을 개발하는 과정 및 그 사이 문제점,해결방법,개선을 위해 적용한 방법 등을 정리.

1. 2020년도 중순 IE 지원 종료가 다가옴에 따라 Non-Activex 개발을 주요 이슈로 올림.
   * 기존 AI박스 및 S1 사업자 대상으로 [media-stream-library](https://github.com/AxisCommunications/media-stream-library-js) [^1]를 사용한Non-Activex 플레이어 프로젝트를 제공했으나, [H.265](https://www.koreascience.or.kr/article/JAKO201435051109807.pdf)[^3] 미지원하므로 자사 제품에 탑재할 수 없었음.
   * 따라서 H265 플레이를 지원할 수 있도록 [ffmpeg](https://www.ffmpeg.org/)[^4] 를 포팅한 [WebAssembly](./Web/Frontend/WebAssembly.md) 및 [Javascript Worker](./Web/Frontend/Javascript_Worker.md)를 이용한 Non-Activex 플레이어를 개발해야함. 
   * 이 시기의 녹화기에 올라간 Web Server( Nginx ) 및 Host엔 RTSP 통신을 위한 채널이 없었고, 이를 위해 Host에  RTSP송수신을 위한 WebSocket 서버가 추가되었고, Nginx에 WebSocket 요청을 받고 Host의 WebSocket 서버에 프록시 하기 위해 WebSocket 모듈을 추가하여 리빌드함.
   * 이 시기에 ffmpeg을 빌드 및 운용 경험이 있는 선임이 주도적으로 스트리밍 모듈을 설계하고 개발함.
2. 아래와 같은 구조로 NAC라 불리는 RTSP 통신을 이용한 H264/H265 지원 플레이어 모듈의 골자가 완성됨. 

   * ![image-20220227172448604](TYPORA_IMG_TMP/Non-Activex 개발일지/image-20220227172448604.png){: width="100" height="100"}
   * Controller : Javascript GlobalWorker( UI Worker)에서 각 Worker 사이의 가교역할( Worker끼리 직접 통신할 수 없는 구조적 한계 때문에 ) 및  UI Thread 의 역할로 Draw canvas에 디코딩 된 Frame 데이터를 그린다. 
     * 각 Worker에 명령을 내린다.
     * WS Worker에서 수신한 
     * DrawCanvas는 WebGL을 이용해 yuv frame데이터를 추가함.
   * WS Worker :  Web페이지에 연결한 세션에 의존적이지 않은 RTSP 세션 체결 및 메세지 송수신, RTP Streaming을 위해 WebSocket을 이용한 연결을 추가하고 관리하는 Worker 이다.
     * RTSP 세션 체결 : OPTIONS, DESCRIBE, SETUP 명령을 이용해 RTSP 표준으로 메세지를 송수신하여 플레이 가능한 세션을 생성한다.
     * Play Control 메세지 송수신 : PLAY, PAUSE, TEARDOWN 메세지 등을 송수신하여 PLAY를 컨트롤한다. 
     * Streaming 데이터 패킷( WebSocket 패킷 )수신시 Controller로 Throwup하여 패킷데이터를 분석해 Frame으로 조립될 수 있도록 한다. 
     * 위 기능과 관련된 인터페이스( postMessage/onmessage )정의.
   * Frame Worker : WS Worker에서 수신한 Streaming 데이터 패킷을 분석해 각 채널별 수신한 Frame 데이터를 조립하는 Worker. 
     * Packet 데이터엔 RTP 데이터가 포함되어 있고, ITX RTSP는 여러 채널 데이터를 한 연결에 송신하는 커스텀 프로토콜 이므로, 패킷을 분석하고 파싱해야한다.
     * RTP 표준 헤더 및 ITX RTSP 확장 RTSP를 데이터를 영상 출력에 필요한 encoded frame 데이터 및 관련 메타데이터로 파싱하여 Controller에 넘겨준다.
     * 위 기능과 관련된 인터페이스( postMessage/onmessage )정의.
   * Decoding Worker : Frame Worker에서 조립한 프레임 데이터 및 메타데이터를 수신하여 WebAssembly로 포팅된 FFMPEG 라이브러리( ffmpeg.wasm )를 이용해 디코딩한다.
     * FFMPEG 라이브러리는 ffmpeg을 이용해 디코딩하는 인터페이스 c 소스코드를 포함해 [emsdk](https://emscripten.org/docs/getting_started/downloads.html)[^5]로 크로스컴파일하여 얻어낸 WebAssembly파일이다.
     * emsdk 컴파일시 wasm의 인터페이스로 추가한 c언어 함수를 config 옵션으로 정의하여 외부에서 접근할 수 있도록한다.
     * Decoding Worker는 `ffmpeg.wasm` 및 emsdk에서 `ffmpeg.wasm` 생성시 같이 생성한 인터페이스 모듈 `js`파일 등을 이용해 Frame Worker에서 수신한 Encoded Frame 데이터를 디코딩하여 yuv 프레임 데이터를 만든다.
     * yuv 프레임 데이터 및 해당 프레임의 메타데이터를 포함하여 Controller로 송신한다.
     * 위 기능과 관련된 인터페이스( postMessage/onmessage )정의.  
3. 2021년 1월에 나를 포함하여 Web Application에 기능 추가를 위한 모듈 개발 시작.

   * NAC 개발과 모듈 개발 시작 사이에 기존 프로젝트에서 사용하던 media-streaming-library를 이용한 외부 모듈 프로젝트를 맡아 외부인터페이스 모듈을 제공했음.( 추후 정리 )

   * 성능테스트
   * 한계 및 기능 파악
   * 기존에 제공하던 Activex를 이용한 플레이어와 비교하여 Non-Activex의 스펙시트 작성.
4. 2021년 5월 단일 채널 영상 스트리밍( 카메라 ) 기능개발 완성.

   * 플레이어 영상이 보여지는 DOM 및 Layout을 담당할 VideoLayoutManager 모듈 추가 개발
     * Aspect Ratio 기능 개발.
   * Web Application이 SAP 프레임워크를 사용함에 따라 Player가 포함된 UI의 동적 Load시 NAC 모듈의 갱신에 따라 발생하는 지연을 줄이기 위해 NAC를 독립적인 모듈로 설계. 
     * Web Application이 Load에 맞춰, Player가 보여지는 UI가 첫번째 동적 Load시 NAC를 생성하고, 이후 WebApplication이 유지되는 동안 Singleton으로 생성된 NAC 오브젝트를 참조함. 
     * Player가 보여지는 UI의 생명주기에 따라 NAC의 Statement를 컨트롤 할 수 있도록 함. 
       * ![image-20220227185542818](TYPORA_IMG_TMP/Non-Activex_개발일지/image-20220227185542818.png)
       * 크게 INIT -> READY -> WORKING -> TERMINATE로 나뉘며, INIT은 처음 NAC 객체가 생성될때만 수행되고 Web Application이 유지되는 동안 READY~TERMINATED 상태를 순환한다.
     * NAC가 미지원하는 기능을 Activex을 이용하여  maintainence 하기 위해 IE에서 인터프리팅 되더라도 동작할 수 있도록  브라우저를 체크한 후에만 동작하는 독립적인 스크립트로 작성함.
       * 표준적인 방식은 `type=module`을 이용하는 것이지만, 이 당시 이를 몰라 원시적인 `js` 모듈러 방식( 막 코드 )을 구현하였다.
       * Web Application에선 `nac.js`파일만 포함해도 NAC 모듈과 관련된 파일을 import할 수 있도록 `nac.js`에 의존관계에 있는 `js`파일들을  browser check, 그외 실행환경을 보고 동적으로 document에 `script` 노드를 추가하는 방식으로 작성하였다.
       * 이 방식의 문제점
         * 의존관계에 따라 load 타이밍이 선행되어야 하는 모듈들의 경우 일일히 모듈관계를 스크립트로 작성해야해서 모듈 로딩 타이밍을 컨트롤해야한다.
         * Worker 및 WebAssembly는 ES6이상을 지원하는 브라우저에서만 지원되므로 동적으로 load되는 스크립트 내부에선 ES6이상 문법을 써도된다. 하지만 직접 WebApplication에 스크립트가 추가해야하는 `nac.js`에 ES6문법을 쓰게되면 IE에서 에러가 발생하게 된다.
         * 이로 인해, NAC 모듈을 구성하는 모든 모듈들이 같은 Context를 유지할 수 없다.
         * 이로인해 `nac.js`는 의도적으로 es5이하 문법을 유지해야했고, 유지보수에 어려움이 있었다.
         * 이를 개선할 방법으로 NAC모듈을 로드하는 `nac_importer.js`와 같은 추가 인터페이스 모듈을 선행하여 로드하고, es6 문법으로 작성된 본 모듈파일인 `nac.js`를 추가로드하게 하는 방법을 생각했으나 실행에 옮기진 않았다.
     * 문제점 
       * `SharedArrayBuffer`를 사용하는 wasm 코드가 포함되어 있어, `cross-origin isolated` 상태로 만들기 위해 COEP, COOP 헤더를 추가했다. ( 이후 emsdk config 옵션에 thread pool 갯수를 1로 변경하여 SharedArrayBuffer를 사용하지 않도록해 해결함. )
         * 일정 Chrome 버전에서 멜트다운 및 Specture 이슈로 인해 막혔다, Chrome에서만 94버전 이상에서 위 헤더를 추가하면 사용할 수 있도록 허용했다. 
         * HTTPS에서만 동작 가능하는 문제가있으며 자사 제품의 경우 HTTPS를 사설 인증서를 사용하여 관련경고가 뜨는 문제로 인해 보안이슈를 제기받는 입장이여서 HTTP지원이 필요했다.
         * 뿐만 아니라 `SharedArrayBuffer`사용으로 인한 멜트다운 및 Specture문제는 여전히 남아있는 상태이며, 관련 공격을 제한하기 위한 임시방편으로 `cross-origin isolated`상태를 만들라고 하는 것 뿐이여서 결과적으론 언제 제한될지 모르는 기능이라 다른 방법으로 강구해야했다.  
       * `nac.js`파일에 아래의 기능을하는 코드가 전부 포함되어있어 유지보수가 힘들어짐.
         * WebApplication에서 명령 및 응답을 송수신하는 인터페이스 및 관련 명령을 Worker와 VideoLayoutManager에 분배하는 코드. 
         * 각 Worker를 컨트롤하는 코드( onMessage/ postMessage / statement 관리 )
         * VideoLayoutManager에 관련 명령및 statement 관리코드.
       * 불완전한 모듈 로드 방식 및 의존성 관리로 인한 비표준성 및 유지보수의 어려움.
       * 디자인패턴을 적용한게 아니라, 범용적인 구조로 작성되지 않아, 이후 내가 아닌 사람이 보기에 구조파악을 쉽게 할 수 없는 문제점.
         * 개발당시엔 OOP 원칙에 따른 의존성 부여, 결합도 낮춤, 관점에 따른 모듈 분리에 맞춰 개발했으나 이를 구조적으로 적용한 이론이 없어 다른사람한테 설명할때 일일이 설명해야함.
         * 설계의 부재.
5. 2021년 10월 다채널 플레이어 기능 완성 및 NAC 모듈 유지보수성 개선.

   1.  `type=module` 사용.
   1.  레코더 우선 적용으로 변경됨에 따라 다채널 지원하도록 개발함.


[^1]: [Media Source Extensions]()기술을 이용해 Axis[^2]에서 배포한 RTSP 연결로 스트리밍할 수 있는 Non-Actviex library 
[^2]: 스위스에 위치한 세계적인 보안장비 소프트웨어 및 인프라 기업 

[^3]: 영상 압축 방식(Codec) 중 한가지. 윈도우/안드로이드 진영에선 통상 HEVC라고 불린다 . Apple 진영에선 HEIF라고도 부르지만 같은 코덱이다.
[^4]: 가장 널리 알려진 H264/H265 코덱 지원 오픈소스 라이브러리(라이센스 : LGPL/GPL)
[^5]: c언어로 작성된 소스를 WebAssembly로 크로스컴파일하는데 쓰이는 대표적인 llvm 계열 오픈소스 크로스컴파일러.
