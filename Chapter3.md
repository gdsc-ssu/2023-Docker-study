# 도커 이미지 만들기

레지스트리 : 이미지를 제공하는 저장소

도커 이미지 내려 받기

```python
docker image pull diamol/ch03-web-ping
```

내려받은 이미지로 컨테이너를 실행하고 실행된 애플리케이션의 기능 확인

```python
docker container run -d --name web-ping diamol/ch03-web-ping
```

- 이 컨테이너의 특이점이라 한다면 네트워크를 통해 요청을 받지 않는다.
- -d 플래그 : —detach의 축약형 (백그라운드에서 동작)
- —name 플래그 : 컨테이너에 원하는 이름을 붙이고 이 이름으로 컨테이너 지칭

### Dockerfile 작성하기

- 일련의 인스트럭션으로 구성됨

```python
FROM diamol/node
ENV TARGET="blog.sixeyed.com"
ENV METHOD="HEAD"
ENV INTERVAL="3000"

WORKDIR /web-ping
COPY app.js

CMD ["node","/web-ping/app.js"]
```

FROM

: 모든 이미지는 다른 이미지로부터 출발한다. 이 이미지는 diamol/node 이미지를 시작점으로 지정함. 

ENV

: 환경 변수값을 지정하기 위한 인스트럭션 . key-value 형식을 따름

WORKDIR

: 컨테이너 이미지 파일 시스템에 디렉터리를 만들고 해당 디렉터리를 작업 디렉터리로 지정하는 인스트럭션

/web-ping 디렉터리를 만든다.

COPY

: 로컬 파일 시스템의 파일 혹은 디렉터리를 컨테이너 이미지로 복사하는 인스트럭션 

[원본경로] [복사경로] 형식으로 지정하면 됨

CMD

: 도커가 이미지로부터 컨테이너 실행했을 때 실행할 명령을 지정하는 인스트럭션

Node.js런타임이 애플리케이션을 실행하도록 app.js지정

### 컨테이너 이미지 빌드하기

```python
docker image build --tag web-ping .
```

- —tag의 인잣값(web-ping)은 이미지의 이름이다.
- 이어지는 인자는 Dockerfile 및 이미지에 포함시킬 파일이 위치한 경로이다 ⇒ 도커에서는 이 디렉터리를 컨텍스트르라고 한다.
- 마지막의 . (점)은 현재 작업 디렉터리를 의미한다.

### 질문사항

![image](https://user-images.githubusercontent.com/85864699/227063510-38191952-4058-4e54-b53a-13f4aea389de.png)

컨테이너 내부에서 도커 이미지 빌드가 안되는지…

흐엏 예제 코드를 git clone해야했음..

![image](https://user-images.githubusercontent.com/85864699/227063571-c08c191a-ed77-4f1f-827d-f9a488a3cfe1.png)

### 도커 이미지와 이미지 레이어

- 도커 이미지에는 우리가 패키징에 포함시킨 모든 파일이 들어있다
- 이들 파일은 나중에 컨테이너의 파일 시스템을 형성한다.
- 이미지에는 자신에 대한 여러 메타데이터 정보들도 들어있다.
- 이 정보를 이용하면 이미지를 구성하는 각 레이어는 무엇이고 이들 레이어가 어떤 명령으로 빌드됐는지 알 수 있음.
- 도커이미지 ⇒ 이미지 레이어가 모인 논리적 대상
    - 레이어 : 도커 엔진의 캐시에 물리적으로 저장된 파일
    - 이미지 레이어는 여러 이미지와 컨테이너에서 공유되기 때문
    - 실제 차지하는 것 보다도 45% 이상 줄일 수 있음
- 예시
    - Node.js 애플리케이션이 실행되는 컨테이너를 여러게 실행한다면 이들 컨테이너는 모두 Node.js 런타임이 들어있는 이미지 레이어를 공유
    

### 이미지 레이어 캐시를 이용한 Dockefile 스크립트 최적화

- 이미지 레이어를 여러 이미지가 공유한다면 공유되는 레이어는 수정할 수 없어야 함.
- 만약 이미지 레이어를 수정할 수 있다면 ⇒ 그 수정이 레이어를 공유하는 다른 이미지에도 영향을 미치게 됨
- 도커 이미지 레이어를 읽기전용으로 만들어 두어 문제 해결 가능
- 도커는 캐시에 일치하는 레이어가 있는지 확인하기 위해 해시값 이용

### 질문사항 : 이미지와 이미지 레이어 차이?

이미지 레이어도 이미지 근데 이미지 레이어는 아키텍쳐라고 생각하면 됨

레이어 : 기존 이미지에 추가적인 파일이 필요할 때 다시 다운로드 받는 방법이 아닌 해당 파일을 추가하기 위한 개념이다.

아무튼 이미지가 근본이다!

컨테이너: 이미지를 실행한 상태

### 연습문제 풀이

-it : 컨테이너에 접속해서 컨테이너 조작하겠다는것

```python
docker container run diamol/ch03-lab
docker container run --interactive --tty diamol/ch03-lab # 컨테이너에 들어가겠다/diamol  
ls
#ch03.txt
/diamol vi ch03.txt

```

vi 편집기로 ch03.txt를 고친다

(의도 : 이미지도 건들이지 않고, Dockerfile을 통해 새로 빌드하지 않는다)

프로그램이 커지면 → 빌드할 때 시간이 오래 걸림 (commit 을 쓰면 내가 원하는 부분만 바꿀 수 있음 ⇒ 빌드 시간 줄여줌 git이랑 비슷하다고 보면 됨)

```python
Lab solution, by: jungmin
```

간단하게 txt파일을 바꿔주고

기존의 이미지를 건들이지 않고 새롭게 빌드한다 ⇒ commit 명령어를 사용한다.

```python

docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
docker commit d8 diamol/ch03-lab:0.1
```

- d8이 컨테이너 diamol/ch03-lab이 이미지 이름

[도커(Docker) : 이미지 커밋(업데이트)하기](https://tttsss77.tistory.com/230)

이미지 실행시켜주고 싶으면

```python
docker container run -it Repository:tag
```
