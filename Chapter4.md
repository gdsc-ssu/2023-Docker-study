작업용 노트북에서 빌드를 하려면 개발 팀원 모두가 같은 도구, 같은 버젼을 사용해야해서 이 도구를 설치하는데에만 시간을 다 보내게 된다.

이를 해결해주는 방법!!! 바로 도커.

개발에 필요한 모든 도구를 배포하는 Dockerfile 스크립트를 작성한 다음 이를 이미지로 만들기

이 소스 코드를 컴파일하면 애플리케이션 패키징이 가능!!!

**멀티 스테이지 빌드를 적용한 Dockerfile 스크립트**

![image](https://user-images.githubusercontent.com/58413633/228717000-43c4fbe9-cad3-48cf-bc4b-9e121710b1fa.png)
![image](https://user-images.githubusercontent.com/58413633/228717032-b119d947-9e6a-4204-9653-e98028f83d91.png)

각 빌드 단계는 `FROM` 인스트럭션으로 시작

위 스크립트는 3단계로 나뉘어진 멀티 스테이지 빌드 예시

상위 두 단계는 `build-stage` `test-stage` 라는 이름이 붙어있다

→ `AS` 파라미터를 사용하면 빌드 단계에 이름 붙이기가 가능!!

최종 산출물은 마지막 단계의 내용물을 담고 있는 도커 이미지

두번째 단계에서의 `COPY` 인스트럭션을 보면 `--from` 인자를 사용하여 해당 파일이 호스트 컴퓨터의 파일 시스템이 아닌 앞선 빌드 단계의 파일 시스템에 있는 파일임을 알려주고 있다.

첫번째 단계에서 파일 하나가 생성되고, 생성된 파일을 `test-stage`로 복사하고 다시 `test-stage` 에서 생성한 파일을 마지막 단계에서 복사한다.

`RUN` 인스트럭션은 파일을 생성하기 위해 사용

- 빌드 중에 컨테이너 안에서 명령을 실행한 다음 그 결과를 이미지 레이어에 저장
- 실행할 수 있는 명령에는 제한이 없지만, `FROM`에서 지정한 이미지에서 실행할 수 있는 것이어야만

위에서는 `diamol/base` 기반 이미지로 지정. 그 이미지가 `echo` 명령을 포함하고 있기 때문에 정상적으로 작동

각 빌드 단계는 서로 격리되어 있다.

빌드 단계별 기반 이미지가 다를 수 있으므로 사용할 수 있는 도구도 달라진다

### 실습

> 예제 코드가 있는 디렉터리에서 멀티 스테이지 빌드가 적용된 Dockerfile 스크립트를 사용해 이미지 빌드하기
> 

```python
docker image build -t multi-stage .
```
![image](https://user-images.githubusercontent.com/58413633/228717162-dea9b3bd-e28c-4f12-b34c-d2cc308c0248.png)

 `build-stage` 에서  build.txt 파일 생성

→ 빌드 도구가 설치된 기반 이미지 사용

 `test-stage` 앞서 만든 build.txt파일을 이미지에 추가

→ 앞에서 빌드한 바이너리를 복사해 단위 테스트 수행

`[stage-2 2/2]` 부터 최종 결과물이 될 이미지를 생성하고  `test-stage` 단계에서 build.txt 복사

→ 애플리케이션을 실행할 런타임이 들어 있는 기반 이미지로 시작

→ `build-stage`에서 빌드하고 `test-stage`에서 테스트까지 성공적으로 마친 바이너리를 이 이미지에 복사해서 넣는다.

![image](https://user-images.githubusercontent.com/58413633/228717209-047cc4c4-35ec-49d1-81df-1a8a09cfb1a0.png)

⇒ 애플리케이션의 진정한 이식성 확보!!!!

### 실습: 자바 소스 코드로 빌드하기

**메이븐**

- 빌드 절차와 의존 모듈의 입수 방법을 정의하는 도구
- XML문서 사용
- mvn 명령을 실행하여 사용

**OpenJDK**: 자유로이 재배포가 가능한 자바 런팀이자 개발자 키트

![image](https://user-images.githubusercontent.com/58413633/228717245-ca09733e-f18a-4645-8d9b-b6b2a8c3f874.png)
1단계 (builder 단계)

기반 이미지: `diamol/maven` (메이븐과 OpenJDK 포함)

이미지에 작업 디렉터리를 만든다 → `pom.xml` 파일 복사

`RUN`에서 메이븐이 실행되어 필요한 의존 모듈 다운 (시간 오래걸림 → 별도 단계로 분리)

`COPY ..` 에서 나머지 소스코드 복사됨

→ 도커 빌드가 실행 중인 디렉터리에 포함된 모든 파일과 서브 디렉터리를 현재 이미지 내 작업 디렉터리로 복사하라

마지막: `mvn package` 명령 실행 (= 애플리케이션을 빌드하고 패키징하라)

입력은 자바 소스 코드, 출력은 JAR 포맷으로 패키징된 자바 애플리케이션

2단계

컴파일된 애플리케이션이 해당 단계의 파일 시스템에 생성

기반이미지: `diamol/openjdk` (자바 11런타임 포함, 메이븐 포함 X)

작업디렉터리 생성 후 builder단계에서 만든 JAR파일을 복사

JAR파일: 모든 의존 모듈과 컴파일된 애플리케이션을 포함하는 단일 파일

80번 포트를 `EXPOSE`를 통해 외부로 공개하기

ENTRYPOINT: CMD와 같은 기능. 해당 이미지로 컨테이너가 실행되면 도커가 인스트럭션에 정의된 명령 실행. java명령으로 빌드된 JAR 파일 실행

**실습**

> 이미지 빌드하기
> 

```python
docker image build -t image-of-the-day .
```

굉장히 많은 로그를 볼 수 있는데 의존 모듈을 내려받고 자바 빌드를 실행하는 내용
![image](https://user-images.githubusercontent.com/58413633/228717337-33ec2246-fdf4-4eec-9d0b-f2b5fbd15705.png)

![image](https://user-images.githubusercontent.com/58413633/228717445-a067c929-9fdb-442e-b276-31bcbfb8687d.png)
`[stage-1 3/3]` builder단계에서 패키징된 JAR 파일을 빌드 마지막단계에서 복사

> 컨테이너 간 통신에 사용되는 도커 네트워크 생성하기
> 

```python
docker network create nat
```

![image](https://user-images.githubusercontent.com/58413633/228717478-036f2316-d9df-48f3-8819-d724bc710c31.png)

> 80번 포트를 호스트 컴퓨터를 통해 공개하고 nat 네트워크에 컨테이너를 접속하라!
> 

```python
docker container run --name iotd -d -p 800:80 --network nat image-of-the-day
```

![image](https://user-images.githubusercontent.com/58413633/228717561-07f8d9a1-796b-4032-af02-f7e1cbfd2937.png)

![image](https://user-images.githubusercontent.com/58413633/228717607-36669f6a-38dc-4904-8d9b-0585a7d15599.png)


JSON 포맷으로 확인 가능

<aside>
💡 최종적으로 생성되는 애플리케이션 이미지에 빌드 도구 포함 X

포함하고 싶은 내용이 있다면 최종 단계에서 명시적으로 해당 콘텐츠 복사해오기!!!

</aside>

### Node.js

자바스크립트로 구현되어 있고 인터프리터형 언어라서 별도의 컴파일 절차가 필요 X

컨테이너화된 Node.js 애플리케이션을 실행하려면 Node.js 런타임과 소스 코드가 애플리케이션 이미지에 포함되어야 한다

Node.js는 npm 패키지 관리자를 사용하여 의존 모듈을 관리

![image](https://user-images.githubusercontent.com/58413633/228717636-1511a7d9-8826-46b9-ba75-47c660930e23.png)

`diamol/node` 이미지: Node.js 런타임과 npm이 설치된 이미지

상위 4개의 줄이 빌드 과정 전부임!!!

> 이미지 빌드하기
> 

```python
docker image build -t access-log .
```

기존보다는 빌드가 매우 빠르게 진행됨!!!!

![image](https://user-images.githubusercontent.com/58413633/228717660-017b36af-7562-46da-91fa-020fe223590d.png)

`[stage-1 3/4]` 앞 builder 단계에서 내려받은 의존 모듈을 application 단계에서 복사해오기

`[stage-1 4/4]` 호스트 컴퓨터 src 에서 자바스크립트 파일 복사해오기

이 애플리케이션은 다른 서비스로부터 호출을 받아 로그를 남기는 REST API

> access-log이미지로 컨테이너를 실행하고, nat 네트워크에 연결하여 80번 포트 공개하기
> 

```python
docker container run --name accesslog -d -p 801:80 --network nat access-log
```

![image](https://user-images.githubusercontent.com/58413633/228717693-99f9d87d-457c-4ff7-81c4-a209ec27c649.png)

![image](https://user-images.githubusercontent.com/58413633/228717719-80d0b773-970a-4f5b-b2f5-c97f424c1ff4.png)

남긴 로그 건수 확인 가능!!!

0이 아니라면 다른 서비스가 API를 사용하고 있다는 것!

### GO

: 네이티브 바이너리. 컴파일되는 현대적인 크로스 플랫폼 언어

→ 원하는 어떤 플랫폼이든 해당 플랫폼에서 동작하는 바이너리를 컴파일할 수 있다

→ 별도의 런타임 필요X

→ 도커 이미지 크기 매우 작아진다

![image](https://user-images.githubusercontent.com/58413633/228717767-ddfbafa3-b1dd-4318-bfed-44bab3ddd536.png)

Go는 네이티브 바이너리로 컴파일되므로 각 빌드 단계는 서로 다른 기반 이미지를 사용

builder 단계에서는 `diamol/golang` 이미지 사용

애플리케이션 단계에서는 `diamol/base` 이미지 사용

몇가지 설정값을 환경 변수 형태로 설정하고 컴파일된 바이너리를 실행하여 앱을 시작

앱 단계에서는 builder 단계에서 빌드한 웹 서버 바이너리와 이 웹 서버가 제공할 HTML 파일을 복사하는 과정으로 마무리.

`chmod` 명령을 통해 명시적으로 실행 권한을 부여 받으

> 이미지 빌드하기
> 

```python
docker image build -t image-gallery .
```

![image](https://user-images.githubusercontent.com/58413633/228717804-cfd1115b-f9f6-40bc-8904-837cad339d6e.png)

`[stage-1 3/5]` 호스트 컴퓨터에 있는 HTML 파일을 최종 이미지로 복사

`[stage-1 4/5]` builder 단계에서 빌드한 웹서버 바이너리를 복사

> 빌드에 사용된 Go 빌드 도구 이미지와 빌된 Go 앱 이미지 크기 비교해보기
> 

```python
docker image ls -f reference=diamol/golang -f reference=image-gallery
```

![image](https://user-images.githubusercontent.com/58413633/228717847-cce3c69a-6559-48e1-9def-9ab2ec7a6df1.png)


> Go 앱 이미지로 컨테이너를 실행하되, 컨테이너를 nat 네트워크에 접속하고 80번 포트를 호스트 컴퓨터의 포트를 통해 공개하기
> 

```python
docker container run -d -p 802:80 --network nat image-gallery
```
![image](https://user-images.githubusercontent.com/58413633/228717899-9ad2b8bf-f030-4aa7-8fd3-687434229ee9.png)
![image](https://user-images.githubusercontent.com/58413633/228717914-56ccc6f3-8fb5-4ae0-bc77-b19367eb24f0.png)




현재 3개의 컨테이너에 걸쳐 실행되는 분산 앱이 실행.

Go로 구현된 웹앱이, 자바로 구현된 API를 호출해 이미지의 상세 정보를 얻은 후, Node.js로 구현된 API에 접근 로그를 남긴다

```jsx
curl -X GET localhost:어쩌구
```

### 컨테이너 안에서 애플리케이션을 빌드하는 것이 유용한 이유

1. **표준화**
    
    어느 도구를 설치했던지와 관계 없이 모든 빌드 과정은 도커 컨테이너 내부에서 이루어진다
    
    실무에 적용하면 빌드 실패 크게 줄일 수 있다
    
2. **성능 향상**
    
    Dockerfile 스크립트를 세심하게 최적화하여 작성하면 이후 캐시 재사용을 통해 90% 이상의 빌드 단계에서 시간 절약 가능
    
3. **최종 산출물인 이미지를 가능한 작게 유지 가능**
    
    최종 산출물 이미지에 불필요한 도구 빼기가 가능.
    
    애플리케이션 시작 시간 단축 가능
    
    애플리케이션의 의존 모듈 자체를 줄여 취약점을 이용한 외부 공격 가능성 차단 가능
