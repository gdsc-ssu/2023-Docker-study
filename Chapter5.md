# Registry, Repository, Image Tag
지금까지 Image를 가져올 때는 `diamol/base` 처럼 가져왔다.  
URL 표현처럼 이는 default 값을 이용한 짧은 이용 방식이며, 실제로는 아래와 같이 구성되어 있다.  
`REGISTRY/USER/REPOSITORY:TAG`

* REGISTRY : Image가 저장된 Domain으로, Docker Hub가 default이다.
* USER : 해당 Image의 소유자로, 개인 혹은 단체 뭐든 가능하다. Github의 User 혹은 Organization이라고 생각하면 된다.
* REPOSITORY : Github의 Repository라고 생각하면 된다. Docker에서는 Repository 이름 자체로 이용할 수 있다는 차이점이 있다.
* TAG - Image의 버전으로, Github의 Branch와 비슷한 개념이라고 생각하면 된다.

개인 혹은 팀 프로젝트에서는 Docker Hub를 이용하겠으나, 프로덕션 환경에서는 자체적인 Registry가 존재할 것이다.  
또한 TAG는 이후 다시 보겠으나, 대충 작성하는 것은 유지 보수에 좋지 않다.

# Push Image To Docker Hub
우선 `https://hub.docker.com/` 에서 계정을 생성한다.  
이후 터미널에서 로그인을 해야 하는데 서적에서는 환경 변수로 저장하여 이용하는 예제를 보여준다.  
환경 변수 대신 직접 ID를 입력하는 방식으로 실습을 진행한다.

```bash
docker login --username USER
```

![02_Docker_Login](https://user-images.githubusercontent.com/18185906/231041497-d30d1414-d127-4faa-b69d-1cae0bd7f593.png)

로그인에 성공하면 위와 같이 추가적인 출력이 이루어진다.  
`./.docker/config.json`에는 아래와 같이 암호화된 방식으로 인증 정보가 저장된다.

![03_Docker_Credential](https://user-images.githubusercontent.com/18185906/231041511-ab59efe7-1794-4460-b481-3825cde2489e.png)

개인 계정에 Push 실습을 하기 전, 이전에 사용한 `image-gallery`에 개인 계정을 명시한다.  
해당 작업이 진행되어야 개인 Repository에 Push가 가능하다.  
이를 명시하지 않을 경우 `library` Registry를 이용하게 되며, 이는 Docker Hub에서 공식으로 제공하는 Registry이다.  
본 Registry에는 다양한 공식 Image들이 존재하며, 일반 유저는 Push할 수 없다.  
Push를 시도하여도 아래와 같이 권한이 없어 Push할 수 없다.

![04_Permission_Denied](https://user-images.githubusercontent.com/18185906/231041536-5c6bba1e-dbae-4f8a-8396-66483c6a546b.png)

이전에 사용하였던 `image-gallery`에 계정을 명시하여 Tag를 붙인다.

```bash
docker image tag image-gallery USER/image-gallery:v1
```

아래 명령어를 이용하면 정상적으로 계정을 명시하였는지 확인할 수 있다.

```bash
docker image ls -f reference=*/image-gallery
```

이제 Image가 준비되었으니 개인 계정에 Image를 Push한다.

```bash
docker image push USER/image-gallery:v1
```

![05_Image_Push](https://user-images.githubusercontent.com/18185906/231041565-31dcbd14-87e1-438d-ba81-0e85f8ddab78.png)

![06_Docker_Hub_Repository](https://user-images.githubusercontent.com/18185906/231041579-f9e51534-ed20-432a-bb3e-240bd3fa7dad.png)

Image가 Push되며, `https://hub.docker.com/r/USER/image-gallery/tags`에 들어가서 정상적으로 Push가 이루어진 것을 확인할 수 있다.

# Running Own Docker Registry
Docker Hub에서 Private Repository를 이용할 수 있다.  
하지만 Image를 Push하고 Pull하는 과정에서 번거로울 수 있으며, Image가 크다면 통신 속도로 인하여 작업이 지연될 수 있다.  
이를 해결할 수 있는 방법이 Local Registry이다.

Local에서 Registry를 띄우는 것이 가능하다.  
Docker Enterprise Edition은 Github Enterprise와 비슷하며, 외부와 격리된 독자적인 Registry를 제공한다.  
다만 구독제이기에 초기 스타트업에서는 사용하기에 부담이 될 수 있다. 이때 Local Registry를 이용하는 것이 유용하다.  
Docker는 공식적으로 쉽게 Registry를 띄울 수 있는 [registry Image](https://hub.docker.com/_/registry)를 제공한다.

이번 실습 과정에서 Registry를 자체적으로 띄워서 사용한다.  
다만, Docker에서 제공하는 Registry가 아닌 저자가 간단하게 구성한 Registry를 이용한다.

```bash
docker container run -d -p 5000:5000 --restart always diamol/registry
```

Registry 또한 하나의 Container이다. 5000번 Port끼리 맵핑하였으며, `--restart` 옵션을 통해 Docker가 다시 시작될 경우 자동으로 실행한다.  
Registry Server를 띄웠으며, 간편한 사용을 위해 Alias를 추가한다. 물론 추가하지 않아도 이용할 수 있다.

```bash
echo $'\n127.0.0.1 registry.local' | sudo tee -a /etc/hosts
```

`tee` 명령어는 Output(앞의 명령어)을 받아 Console 및 명시한 파일에 작성한다.  
`/etc/hosts` 파일의 최하단에 registry.local을 등록하였다.  
Alias를 사용하지 않는다면 `127.0.0.1`을 입력해주면 된다.

```bash
docker image tag image-gallery registry.local:5000/gallery/ui:v1
```

기존 `image-gallery`의 Registry를 Local Registry로 명시하였다.  
다만 현재는 이를 실행할 수 없다. 기본적으로 Docker에서는 암호화되지 않은 Registry와 통신을 할 수 없다.  
구성한 Local Registry는 HTTPS가 아닌 HTTP 통신을 하므로 별도의 설정이 필요하다.  
Docker Desktop을 쓰면 쉽게 설정할 수 있으나, 그렇지 않다면 아래와 같이 설정해주어야 한다.

```bash
echo $'{\n\t"insecure-registries": [\n\t\t"registry.local:5000"\n\t]\n}' \
| sudo tee -a /etc/docker/daemon.json
```

위 명령어를 그대로 입력하거나, `/etc/docker/daemon.json`에 아래 내용을 적으면 된다.

```bash
{
        "insecure-registries": [
                "registry.local:5000"
        ]
}
```

Docker의 설정을 바꾸었기에 재시작을 해주어야 적용이 된다.  
재시작 후, 아래 명령어를 통해 `registry.local:5000`의 안전하지 않은 통신이 허용되었는지 확인할 수 있다.

```bash
docker info
```

![07_Docker_Info](https://user-images.githubusercontent.com/18185906/231041609-1d486957-8343-4f7a-aa45-293d5bb19f33.png)

`registry.local:5000`에 대한 안전하지 않은 통신이 허용되었다.  
이제 아래 명령어를 통해 Image를 Push한다.

```bash
docker image push registry.local:5000/gallery/ui:v1
```

# Using Tag Effectively
이전에 Tag를 대충 작성하는 것은 좋지 않다고 했다.  
서적에서는 아래와 같은 방식을 추천하였다.

![08_Tag_Suggestion](https://user-images.githubusercontent.com/18185906/231041628-f764766b-a87f-4439-8646-4d79fbcbe504.png)

* `patch` - 사소한 버그 수정과 같은 작은 변화
* `minor` - 기존 기능 보완 및 개선
* `major` - 새로운 기능 추가, API 변경 등 큰 변화

`major`와 `minor`의 차이가 조금 애매할 수 있으나, 해당 프로젝트 혹은 조직에서 적절한 규칙을 명시하면 된다.

해당 규칙을 적용하여 아래 명령어를 통해 다양한 버전의 Image를 생성한다.  
생성된 Image들은 이후 Lab에서 사용한다.

```bash
docker image tag image-gallery registry.local:5000/gallery/ui:latest
docker image tag image-gallery registry.local:5000/gallery/ui:2
docker image tag image-gallery registry.local:5000/gallery/ui:2.1
docker image tag image-gallery registry.local:5000/gallery/ui:2.1.106
```

# Golden Image
Docker Hub에는 누구나 Container를 배포할 수 있다. 비트코인 채굴 및 전송하는 백그라운드 프로그램을 설정하여 배포할지도 모른다.  
이러한 우려를 막기 위해 검증된 Publisher와 공식 Image가 존재한다.

Publisher는 본인들의 기술을 배포하며, 악성 프로그램에 대한 우려는 지울 수 있다.  
공식 Image는 보통 오픈 소스 S/W로 해당 팀과 Docker측에서 Image를 같이 관리한다.  
공식 Image를 이용해도 되나, 실제로 활용하기 위해서는 다양한 설정을 해야 한다.  
이러한 설정을 어느 정도 처리해준 Image가 바로 Golden Image이다.  
Golden Image는 일부 설정이 적절하게 되어 있는 것이지, 추가적인 설정을 아예 안 해도 된다는 것은 아니다.

# Lab
registry.local Registry를 명시한 다양한 Image가 존재한다.  
REST API만을 이용하여 이를 Push, 모든 Image가 존재함을 확인한 후 다시 모든 Image를 삭제한다.  
명령어를 사용하지 않더라도 REST API를 이용하여 Docker 이용이 가능하다는 것을 파악하고 사용해보기 위한 목적이다.  
REST API에 대한 정보는 [공식 문서](https://docs.docker.com/registry/spec/api/)에서 확인한다.

## Solution
우선 Image들을 registry.local에 Push한다.  
이 부분은 사전 작업에 해당하기에 `docker image` 명령어를 이용하여 간단하게 작업한다.

```bash
docker image push -a registry.local:5000/gallery/ui
```

`Client.Timeout`이 발생하였다면 `registry.local` 방식을 이용하였을 것이다.  
`/etc/hosts`에 Alias가 정상적으로 구성이 되지 않았을 가능성이 높다.

Push가 완료되었다면 Tag를 조회한다.  
[공식 문서의 Listing Image Tags](https://docs.docker.com/registry/spec/api/#listing-image-tags)를 이용하였다.

```bash
curl http://registry.local:5000/v2/gallery/ui/tags/list
```

![09_Listing_Tags](https://user-images.githubusercontent.com/18185906/231041654-4e33a37f-66f5-44ef-8826-ed4835fd25e2.png)

이후 DELETE를 진행하기 위해서는 [DELETE Manifest](https://docs.docker.com/registry/spec/api/#delete-manifest)를 이용하면 된다.  
다만 Manifest가 필요하므로 우선 Manifest를 받아와야 한다.  
[GET Manifest](https://docs.docker.com/registry/spec/api/#get-manifest)를 이용하여 Manifest를 받아온다.

```bash
curl -I http://registry.local:5000/v2/gallery/ui/manifests/v1
```

![10_Wrong_Request](https://user-images.githubusercontent.com/18185906/231041682-f544b420-06d5-48ff-9c27-875dd26a4338.png)

v2 API를 이용하였으나 v1 Manifest를 받아오는 문제가 있다.  
이를 이용하여 DELETE 요청을 보내면 아래와 같이 `MANIFEST_UNKNOWN` 에러가 발생한다.

![11_MANIFEST_UNKNOWN](https://user-images.githubusercontent.com/18185906/231041724-4310ae0f-d2b7-445a-bf45-fb92e8ef0b02.png)

v2 Manifest를 가져오기 위해 `Accept` Header에 `...v2+json`으로 Type을 명시해주어야 한다.

```bash
curl -I http://registry.local:5000/v2/gallery/ui/manifests/v1 -H 'Accept: application/vnd.docker.distribution.manifest.v2+json'
```

![12_Right_Request](https://user-images.githubusercontent.com/18185906/231041743-778c6ce9-2ba0-4a3c-98d6-e423ae8fd1dd.png)

드디어 올바른 Manifest를 받아왔다. 이를 이용하여 DELETE Request를 보낸다.

```bash
curl -X DELETE http://registry.local:5000/v2/gallery/ui/manifests/MANIFEST
```

아무런 출력이 없다면 DELETE가 정상적으로 수행이 된 것이다.  
다시 curl을 이용하여 Tag를 조회할 경우, 아래와 같이 모든 Tag가 삭제되는 것을 확인할 수 있다.

![13_All_Tags_Deleted](https://user-images.githubusercontent.com/18185906/231041764-73e8bea4-1a70-4e25-a9d6-45c99bc313d0.png)

하나의 Tag에 대한 작업을 진행했으나 모든 Tag가 삭제되었다. 이는 당연하다.  
동일한 Image를 이용하여 다양한 Tag를 생성하더라도 같은 Manifest이다.  
해당 Image가 중간에 바뀌었다면 다른 Manifest로 바뀌나, 본 Chapter에서는 `image-gallery`에 대한 변경이 없었기에 모두 같은 Manifest를 가지는 것이다.