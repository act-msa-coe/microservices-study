# 7장. ‘스프링 클라우드 컴포넌트를 활용한 마이크로서비스 확장’

### 스프링클라우드 프로젝트
1) 유레카 : 서비스 등록 및 탐색 / MS 자동 등록 및 서비스 탐색
2) 주울 : 프록시 및 게이트웨이 역할
3) 스프링 컨피그 : 환경설정 외부화 담당
4) 비동기 리액티브 MS 구성에 필요한 스프링 클라우드 메시징

### 스프링 클라우드 
분산 시스템 개발에 필요한 공통 패턴을 모은 스프링 라이브러리
- 분산, 장애 대응, 자체 치유 기능 담당 / 스프링 부트를 바탕으로 비지니스 기능 개발만 집중 가능
- 클라우드 벤더 제품에 비종속적
- 컴포넌트 간 상호 독립적

### #1 Spring cloud Config (sc)
- sc 서버는 애플리케이션과 서비스의 모든 환경설정 속성 정보를 저장, 조회, 관리할 수 있게 해주는 외부화된 환경설정 서버
- 환경설정 정보의 version관리 기능도 지원
#### 1-1) 등장배경
- 기존에는 프로젝트에 application.properties/yml 로 함께 패키징해서 관리해왔으나, MS가 환경이 바뀔 경우 해당 속성 정보가 변경되어야 할 때, 해당 애플리케이션 전체를 다시 빌드해야 함. 이는 환경설정 외부화 원칙 위반
- application-{profile}.properties로 개발 환경에 특화된 환경설정 속성 정보를 담는 건 좀 더 나은 접근방식이지만, a)와 마찬가지로 재빌드가 필요.
- 환경설정 정보를 배포 패키지에서 분리하여 외부화하고 외부 소스에서 설정 정보를 읽어오는 방법은 다음과 같은 4가지가 있음

    (1) JNDI 네임스페이스를 사용하는 외부의 JNDI 서버로 외부화 
   
     >- 단점 : JNDI연산 비용이 높고 유연성이 부족해서 복제하기 어려우며, 버전 관리가 안됨


    (2) Java의 시스템 환경설정 정보 또는 java application 실행시 -D옵션을 통한 외부화
   
     >- 단점 : 대규모 배포시 유연성 보장이 힘듬

    (3) PropertySource 환경설정을 위한 외부화
   
    (4) Java 어플리케이션 실행시 환경 설정 파일 위치 지정을 통한 외부화
   
     >- 단점 : 서버에 mount된 로컬, 공유 파일 시스템에 의존
     
- 결론적으로, 대규모 배포시에는 단순하고 강력한 중앙 집중형 환경설정 정보 관리 솔루션이 필요

#### 1-2) 구조
- 모든 MS 서비스는 환경설정 서버에서 설정값을 읽어와 로컬에 캐싱하여 사용
- 환경설정 서버는 설정값이 변경시 자신을 바라보는 모든 MS에 변경사항을 전파
- 환경설정 서버는 개발 환경별 프로파일 기능 지원

#### 1-3) 종류
- Spring config server, ZooKeeper Configuration, Consul Configuration

#### 1-4) 기본 방식

- Sc 서버는 설정값을 버전 관리도구(git, svn 등) 에 저장한다. 

  단, 대규모 분산 MS 배포에서는 고가용성의 원격 git 저장소를 사용하는 것을 추천
- 설정값의 예시 :  일일거래한도, URL, 각종 인증 정보 등
- 스프링 클라우드에서는 부트스트랩 컨텍스트를 사용 : application.properties/yml => bootstrap.properties/yml 로 바꾸어 사용


#### 1-5) Config 서버 생성

>(1)  깃 저장소에 application.properties 파일 생성, 저장

>(2) config server, Actuator dependency 를 추가한 메이븐 프로젝트 A (config 서버)생성

>(3) 프로젝트 A의 application.properties 파일명을 bootstrap.properties로 변경, 아래 내용 추가
    
    server.port=${A서버의 포트}  => 단, 지정하지 않으면 8888이 기본 포트로 지정
	spring.cloud.config.server.git.uri=${저장소 uri}  => 윈도우 환경에선 URL 끝에 "/" 추가 필요

>(4) A 서버의 Application.java에 @EnableConfigServer 어노테이션 추가

>(5) 실행 시, http://localhost:8888/application/default/master에 접속하면 application.properties 파일 내용을 JSON 형식으로 볼 수 있음

#### 1-6) 컨피그 서버 URL의 이해
>(1) http://localhost:8888/"application"/default/master
>
>- bootstrap.properties 파일에 spring.applicaetion.name속성으로 지정된 이름
>- 각 어플리케이션마다 고유한 이름 ( 서비스 ID ) 을 가져야 함
>- config 서버는 저장소에서 해당 이름에 맞는 환경 설정 정보를 가져옴.
	
    myapp 어플리케이션은 저장소에서 myapp.properties 파일 내의 설정값을 가져옴.
    
>(2) http://localhost:8888/application/"default"/master
>
>- 프로파일 : 기본값은 ‘default’ ( 프로파일이 없는 URL도 유효 )
>- 프로파일 별로 서로 다른 git저장소 지정 가능.

     - Dev, Test, Stage, Prod로 나누는 방식 : 동일한 애플리케이션의 서로 다른 환경
     
     - Primary, Secondary 로 나누는 방식 : 애플리케이션이 배포될 서로 다른 서버를 의미

	 ex) application-development.properties
         application-production.properties
         
>(3) http://localhost:8888/application/default/"master"
>
>- 레이블 : 기본값은 ‘master’
>- 필수 아닌 옵션 정보로서, git의 branch 이름을 붙여서 사용

#### 1-7) Config 서버에 접근할 MS 서비스 생성
>(1)  Config Client,  Actuator 를 선택해 프로젝트 B 생성

>(2) B 프로젝트의 application.properties => bootstrap.properties로 이름 변경

>(3) B-service.properties git 저장소에 추가 (가져올 설정값 입력)

>(4) B 프로젝트 의 bootstrap.properties에 다음 값 입력
    
    spring.application.name=B-service
	server.port=8090
	spring.cloud.config.uri= ${config서버 uri}

>(5) @RefreshScope 어노테이션을 클래스에 추가 : 환경 설정값 변경시 새로운 값 자동으로 가져오기 위함

>(6) 실행시 해당 설정값을 가져옴.

#### 1-8) 환경설정 정보 변경 전파 및 반영
	localhost:8090/refresh
    - POST 호출시 Actuator의 새로고침 기능이 실행되어 설정값의 변경 사항 반영

#### 1-9) 환경 설정 변경을 전파하는 스프링 클라우드 버스

> - 5개의 인스턴스가 실행되고 있다면, 설정값이 갱신될 때마다 각 인스턴스별로 /refresh 호출이 필요
> - ${인스턴스1의 uri}/bus/refresh 
> - 변경값이 인스턴스1에 반영되면, 수정값이 Cloud Bus에 전달되고, 인스턴스 2~5는 Message Broker를 통해 이를 구독하여 변경된 설정값으로 로컬에 캐시된 설정값을 갱신함. 

#### 1-10) 컨피그 서버에 고가용성 적용
	config 서버, git 저장소, RabbitMQ 는 각기 단일 장애지점이므로 고가용성 확보가 필요

> (1) config 서버
> - 다운되었을 때를 대비해 여분의 config 서버가 필요.
> - 하지만, 다운되더라도  각 MS 서비스는 각기 로컬 캐시의 설정값으로 동작할 수 있으므로 다른 MS의 가용성과 동일한 수준으로 취급될 필요는 없음
> - 상태가 유지되지 않는 HTTP 서비스이므로 다수의 config 서버 인스턴스는 병렬적으로 실행 되도 좋음. 단, 각 MS 서비스의 bootstrap.properties에는 하나의 config 서버 uri만 지정 가능하므로, 로드밸런서나 로컬 DNS를 설정하여 해당 uri를 사용하는 것이 필요.


> (2) git 저장소
> - git이든 svn이든 고가용성을 보장하는 서비스 사용 필요
> - 하지만, config 서버도 로컬 설정값이 사용 가능하므로, config 서버 확장시에 고가용성은 더 큰 의미를 갖는다. MS 서비스 수준의 고가용성을 요구하진 않음.

>(3) RabbitMQ
> - /refresh 메시지 전달을 위해 필요
> - 단, /refresh 를 직접 호출하는 것도 가능하므로 다른 컴포넌트 수준의 고가용성을 요구하진 않음.


#### 1-11) 컨피그 서버 상태 모니터링
	http://localhost:8888/health에 접속하여 상태 모니터링 가능
    
#### 1-12) 컨피그 서버 환경설정 파일
	/{애플리케이션 이름}/{프로파일}/{레이블}/{경로 ex) logback.xml과 같은 파일 이름} 를 통해 외부 설정 파일로 접근도 가능.


### #2 유레카를 이용한 서비스 등록 및 탐색
- 각 서비스의 URL을 정적 설정값으로 주는 것은 지양하고 자동화 필요
- MS 서비스가 자신의 서비스를 동적으로 등록하여, 자동으로 사용자의 서비스 탐색 대상에 포함돼 발견되게 해야 함.
>>(1) 동적 등록
>> - 새로운 서비스 시작시, 중앙 서비스 레지스트리에 등록 / 서비스 장애 발생시 자동으로 제외

>>(2) 동적 탐색
>> - 중앙 서비스 레지스트리를 통해 사용 가능한 URL을 탐색 가능
>> - 클라이언트는 더 빠른 서비스를 위해 레지스트리 데이터를 로컬에 캐시 가능

#### 2-1) 유레카의 이해
> - 서비스의 self-registration, 동적 탐색, 부하 분산에 주로 사용
> - 부하 분산을 위해 내부적으로 리본 사용
> - 클라이언트, MS 인스턴스들은 각기 유레카 클라이언트를 가지고 있고, MS  인스턴스들은 유레카 서버에 정보 (일반적으로 서비스 ID, URL)를 등록하고, 클라이언트는 유레카 서버에서 필요한 서비스를 탐색하여 가져온다.
>>>(1) MS 서비스가 시작되면 유레카 서버의 레지스트리에 서비스ID, URL 등의 정보를 등록하고, 30초 간격으로 ping요청을 날리면서 자신이 살아있다는 것을 알린다. 만약 ping 요청이 몇번 전송되지 않으면 죽은 것으로 간주되어 레지스트리에서 제외된다.

>>>(2) 모든 유레카 클라이언트는 레지스트리 정보를 복제하여 로컬에 캐싱한다. 30초마다 주기적으로 갱신되어 delta updates 방식으로 갱신된다. (최근 정보와 현재 레지스트리 정보의 차이를 가져옴)

>>>(3) 클라이언트가 MS의 API를 호출하려 하면, 유레카 클라이언트는 요청된 서비스 ID를 기준으로 현재 사용 가능한 서비스 목록을 제공한다. 이때 리본 클라이언트는 유레카 클라이언트가 알려주는 사용 가능한 MS 인스턴스들에게 요청을 분산해서 보내는 방식으로 부하 분산을 실행한다. 

>>>(4) 유레카 클라이언트와 유레카 서버 사이의 통신은  REST/JSON이 사용된다.

#### 2-2) 유레카 서버 생성
> (1) 스프링 스타터 프로젝트에서 Config Client, Eureka Server, Actuator를 선택하여 유레카 프로젝트 C를 생성

> (2) 프로젝트 C의 application.properties => bootstrap.properties로 변경
	
    spring.applicaetion.name=eureka-server1
	server.port:8761
	spring.cloud.config.uri=http://localhost:8888  => config 서버 uri
> (3) 기본 설정에서는 유레카 서버 == 유레카 클라이언트

> (4) 유레카 클라이언트는 eureka.client.serviceUrl.defaultZone 값이 같은 다른 유레카 클라이언트와 동료(peer) 관계를 형성한다. 독립 설치에선 이 값은 자기 자신의 인스턴스를 가리킨다.

> (5) eureka-server1.properties 파일은 config 설정값 저장소에 생성한다. ( 2) 에서 spring.applicaetion.name로 지정한 이름 )

	spring.application.name=eureka-server1
	eureka.client.serviceUrl.defaultZone= ${유레카서버 URI}/eureka/
	eureka.client.registerWithEureka=false
	eureka.client.fetchRegistry=false

> (6) 프로젝트 C의 Application.java에 @EnableEurekaSever 어노테이션 추가

> (7)  프로젝트 C를 실행하여 http://localhost:8761에 접속하면 유레카 콘솔 화면을 볼 수 있다.
>> - Instances currently registered with Eureka 에서 등록된 인스턴스 목록을 볼 수 있음.

#### 2-3) 유레카 서비스 등록

>(1) Config Client, Actuator, Web, Eureka discovery를 선택하여 동적 등록 및 탐색 기능을 위한 유레카 서비스 프로젝트 D를 생성

>(2) git 저장소에 있는 각 MS 의 properties 파일에 유레카 서버에 연결 가능케 하도록 다음 내용 추가

	eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka => 유레카 서버 URI

>(3) MS  의 springboot 메인 클래스에 @EnableDiscoveryClient 어노테이션을 추가하면 MS 실행 시 자동으로 서비스의 가용성을 유레카 서버에 알리게 됨.


#### 2-4) 클라이언트 생성

>(1) "3) 유레카서비스 등록" 과 와 동일한 방식으로 구성

>(2) 메인 클래스에 AppConfiguration 클래스 추가하고 @Configuration 어노테이션 추가

>(3) @LoadBalanced 어노테이션이 붙은 RestTemplate을 @Bean 어노테이션으로 Bean으로 등록하여 부하 분산이 적용된 RestTemplate을 사용할 수 있도록 함.

>(4) RestTemplate 인스턴스를 사용하여 모든 서비스를 호출. 하드 코딩되어 있던 MS URL을, 유레카 서버에 등록된 각 MS 의 서비스 ID ( ex) search-service, book-service 등)로 대체.

>(5) 실행

#### 2-5) 고가용성 유레카 서버

- 실제 운영 시스템에서 유레카 서버는, 독립 설치형 모드보다 클러스터 형태로 구성
>>(1) 유레카 클라이언트는 서버에 연결해 레지스트리 정보를 가져와 로컬에 캐시하여, 언제나 로컬 캐시 내용을 기반으로 동작.

>>(2) 유레카 클라이언트는 주기적(30초) 유레카 서버 정보를 체크하여 변경사항을 로컬 캐시에 재반영.

>>(3) 동작에는 문제가 없으나, 클라이언트가 최신 정보를 반영하지 않게 되었을 때 일관성 문제 발생 가능

- 유레카 서버는 P2P 방식으로 데이터 동기화 메커니즘을 바탕으로 만들어짐.

- 런타임 상태 정보는 DB가 아닌, 인메모리 캐시에서 관리

- 유레카 서버의 고가용성 구현은 CAP 정리( 분산 시스템에서 일관성, 가용성, 분할 용인을 모두 갖는 것은 불가능) 에 따라 가용성과 분할 용인을 선택하고 일관성을 포기.

- 유레카 서버 인스턴스는 비동기적인 메커니즘으로 peer 관계에 있는 다른 유레카 서버 인스턴스와 동기화하기 때문에 모든 유레카 서버 인스턴스의 상태 정보가 언제나 동일하지는 않다. 

- 하나 이상의 유레카 서버가 있다면 각 유레카 서버는 peer 관계에 있는 서버 중 최소한 하나의 서버와는 연결되어야 한다.


- 유레카 서버에 고가용성을 적용하는 방법은 여러대의 유레카 서버를 클러스터링하여 로드밸런서나, 로컬 DNS 뒤에 두는 것. 
>> (1) 유레카 서버의 spring.application.name=eureka 로 변경 (기존 eureka-server1)

>> (2) git 저장소에 eureka-server1.properties / eureka-server2.properties 파일을 생성

		ex) eureka-server1.properties의 경우  
		    eureka.client.serviceUrl.defaultZone= ${eureka2 URI}/eureka/
			eureka.client.registerWithEureka=false
			eureka.client.fetchRegistry=false

>> (3) 프로파일을 지정하여 유레카 서버 2개 실행하면 동기화 되어 실행


### #3 주울 프록시 API 게이트웨이
- 간단한 게이트웨이 서비스, 에지 서비스

- 다른 기업용 API 게이트웨이 제품과 달리 개발자가 특정한 요구 사항에 알맞게 설정하고 프로그래밍할 수 있게 개발자에게 완전한 통제권을 준다.

- 라우팅, 모니터링, 장애복구 관리, 보안 등 다양한 기능 수행 가능

- API 계층에서 서비스의 기능을 재정의해서 뒤에 있는 서비스의 동작을 바꿀 수도 있다.

- 사전 필터, 라우팅 필터, 사후 필터, 에러 필터 등 여러 필터를 제공

#### 3-1) 주울 설정
- 유레카, config 서버와 달리 특정 MS 단위에서 구성되지만 하나의 API게이트웨이가 여러 MS를 담당하는 경우도 있다.
> (1) Zuul, Config Client, Actuator, Eureka Discovery를 선택해 프로젝트 E 를 생성한다.

> (2) git 저장소에 ms-apigateway.properties 파일을 생성한다.
		
        spring.application.name=ms-gateway
        zuul.routes.ms-apigateway.serviceId=ms
        zuul.routes.ms-apigateway.path=/api/**
        eureka.client.serviceUrl.defaultZone=${유레카서버uri}/eureka/ 
        
        => API 게이트웨이의 /api 로 들어오는 모든 요청은  ms로 전송된다.

> (3) 프로젝트 E 의 springboot의 메인 클래스에 @EnableZuulProxy 어노테이션을 추가해서 어플리케이션에 Zuul 프록시 역할을 주입한다.

> (4) config 서버, 유레카 서버, 해당 ms를 실행하고, 프로젝트 E 를 실행한다.

> (5) 클라이언트에서 API 게이트웨이를 호출하도록 수정한다.


#### 3-2) zuul 프록시가 더욱 유용한 경우
- 인증이나 보안을 게이트웨이 한 곳에서 수행할 경우
- 비즈니스 insight 및 모니터링 기능 : 실시간 통계 데이터를 수집하고, 수집한 데이터를 외부 분석시스템에 전달할 경우
- 세밀한 제어를 필요로 하는 동적 라우팅 : 특정 값에 따라 요청을 분류해 다른 ms 인스턴스로 보내는 경우
- 부하 슈레딩/스로틀링이 필요한 상황 :일일 요청 수와 같은 한계값을 기준으로 부하를 제어하는 경우
- 세밀한 제어를 필요로 하는 부하 분산 처리 :  zuul, 유레카 클라이언트, ribon을 함께 사용하면 세밀한 조절 능력으로 부하 분산 요구 사항에 대처 가능
- 데이터 집계가 필요한 상황

#### 3-3) 고가용성 주울

- 모든 트래픽이 zuul 프록시를 통해 들어오므로 zuul의 고가용성은 매우 중요. 하지만, 탄력적인 확장 기능은 ms 만큼 필수적이지 않다.

- 전형적인 사용 사례

>> (1) 클라이언트측 js mvc 프레임워크가 zuul 서비스 호출

>> (2) zuul을 통해 ms 또는 일반 서비스에 접근하는 서비스

- PL/SQL로 작성된 레거시 어플리케이션처럼 상황에 따라 클라이언트가 유레카 클라이언트 라이브러리를 사용할 수 없을 수도 있다. 이런 경우 조직의 정책에 따라 인터넷 클라이언트가 클라이언트 단에서 부하 분산 처리하는 것을 허용하지 않는다.

- 브라우저 기반 클라이언트인 경우 3rd 파티에서 만든 유레카용 자바스크립트 라이브러리도 사용할 수 있다.

#### 3-4) 클라이언트가 유레카 클라이언트이기도 할 때의 고가용성 주울
- zuul 도 자신의 서비스 ID 를 유레카에 등록하면, 클라이언트는 해당 서비스 ID를 통해 zuul에 접근 가능하다.

#### 3-5) 클라이언트가 유레카 클라이언트가 아닐 때의 고가용성 주울
- 유레카 서버에 의한 부하 분산 처리를 쓸 수 없다. 이때는 별도의 DNS/로드 밸런서를 통해 zuul 서버를 접근하여 부하 분산 처리를 한다.

### #4 리액티브 마이크로서비스를 위한 스트림  
- 스프링 클라우드 스트림은 래빗엠큐, 레디스, 카프카 등 다양한 말단 구현체를 제공하는 라이브러리이다.
- 스프링  5.0 마이크로서비스 2판 p381 참조

### #5 스프링 클라우드 시큐리티를 활용한 마이크로 서비스 보호
> (1) 기본 패턴 : 게이트웨이를 watchdog로 두고 경계를 넘지 못하게 보안을 구현
		   
>>- ms를 향한 모든 요청이 API 게이트웨이를 거쳐가도록 보장하는 것이 중요 
>>- 엔터프라이즈 사이버 보안 차원에서 용납되지 않는 패턴

> (2) 네트워크를 격리하고, 서비스가 게이트웨이에만 열려있게 하는 것이 좋음.

> (3) 토큰 릴레이를 활용하는 방식










