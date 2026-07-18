# 서버 스터디 2주차 — Docker로 PetClinic 운영하기

> 지난주엔 PetClinic을 **내 컴퓨터에서 직접** 실행하고, 포트·프로세스·로그로 상태를 확인했다.
> 이번 주는 같은 앱을 **Docker 컨테이너**에 담아 실행한다.
> 핵심 질문은 하나다 — "완성된 서버를, 어디서든 똑같이, 안정적으로 굴리려면 어떻게 해야 하는가?"

이 문서는 발표를 따라오지 않고 **혼자 읽어도 이해되도록** 썼다. 개념 → 왜 필요한가 → 실습에서 어떻게 확인하는가 순서로 이어진다.

---

## 0. 이번 주를 관통하는 질문

지난주에 우리는 PetClinic이 "잘 돈다"는 걸 확인했다. 그런데 실제 서버 운영은 여기서 시작이 아니라 그다음부터가 진짜다.

- 내 노트북에선 되는데, **다른 사람 컴퓨터에서도** 똑같이 될까?
- 서버가 한밤중에 **갑자기 죽으면** 누가 다시 켜주나?
- 서버가 **메모리를 다 먹어서** 컴퓨터 전체가 멈추면?
- 서버가 안 열릴 때 **어디를 봐야** 원인을 아나?

이 질문들에 답하는 도구가 Docker다. 이번 주는 명령어를 외우는 게 목적이 아니라, **위 질문들을 하나씩 직접 확인**하는 게 목적이다.

---

## 1. 왜 Docker인가 — "내 컴퓨터에선 되는데요?" 문제

### 문제 상황

개발하다 보면 반드시 겪는 일이 있다. 내 노트북에선 완벽하게 돌아가는 서버가, 팀원 컴퓨터나 실제 배포 서버에선 안 뜬다. 자바 버전이 다르거나, 환경변수가 없거나, OS가 달라서다. 이걸 흔히 **"내 컴퓨터에선 되는데요(works on my machine)"** 문제라고 부른다.

### 컨테이너라는 해결책

Docker는 앱을 **컨테이너**라는 상자에 담는다. 이 상자 안에는 앱뿐 아니라 앱이 돌아가는 데 필요한 것(자바 런타임, 라이브러리, 설정)이 **전부** 들어 있다. 그래서 이 상자만 통째로 옮기면, 받는 쪽 컴퓨터에 자바가 깔려 있든 없든 상관없이 **똑같이** 돈다.

> **비유**: 이사할 때 짐을 컨테이너 박스에 다 넣어서 옮기는 것과 같다. 새 집에 가구가 있든 없든, 내 박스만 열면 내 물건이 그대로 있다.

### 컨테이너 vs 가상머신(VM)

비슷한 걸로 VM이 있는데, 결정적 차이가 있다.

| | 가상머신(VM) | 컨테이너 |
|---|---|---|
| **OS** | 각자 운영체제를 통째로 포함 | 호스트 OS의 커널을 **공유** |
| **크기** | 보통 GB 단위 | 보통 MB 단위 |
| **시작 속도** | 느림(OS 부팅) | 빠름(프로세스 실행 수준) |
| **비유** | 집 안에 작은 집을 통째로 짓기 | 큰 방을 칸막이로 나눠 쓰기 |

VM은 운영체제를 통째로 들고 다녀서 무겁다. 컨테이너는 호스트 컴퓨터의 OS를 빌려 쓰기 때문에 가볍고 빠르다. 그래서 서버 한 대에 컨테이너 수십 개를 띄우는 게 흔하다.

> **심화 각주 — 그런데 Windows/macOS에서는?** 컨테이너가 공유하는 커널은 정확히는 **리눅스 커널**이다. 그래서 macOS·Windows에서는 Docker Desktop이 내부에 가벼운 리눅스 VM(Windows는 WSL2)을 하나 띄우고, 모든 컨테이너는 그 VM 안에서 돈다. "그럼 결국 VM 아닌가?" 싶겠지만, VM은 딱 한 대만 두고 그 위의 컨테이너 수십 개가 커널을 공유하는 구조라서 컨테이너의 가벼움은 그대로 유지된다. 스터디에서 이 질문이 나오면 이렇게 답하면 된다.

### 정리: Docker의 3대 이점

1. **이식성** — 어디서든 똑같이 실행된다. (오늘 실습의 `save`/`load`가 이걸 증명한다)
2. **격리** — 컨테이너끼리, 그리고 호스트와 서로 간섭하지 않는다.
3. **일관성** — "매번 똑같은 환경"이 보장돼서 배포·협업이 쉬워진다.

---

## 2. Docker의 3가지 핵심 개념 — Dockerfile · Image · Container

이 셋은 늘 헷갈리는데, **요리에 비유**하면 한 번에 정리된다.

| 개념 | 비유 | 한 줄 정의 |
|---|---|---|
| **Dockerfile** | 레시피(조리법) | 이미지를 **어떻게 만들지** 적은 설계도 |
| **Image** | 밀키트(냉동 완제품) | 실행에 필요한 모든 걸 담은 **불변의 패키지** |
| **Container** | 요리해서 접시에 담긴 음식 | 이미지를 **실제로 실행한 인스턴스** |

흐름은 이렇다.

```
Dockerfile  --(build)-->  Image  --(run)-->  Container
  레시피                    밀키트              실제 음식
```

- **이미지는 한 번 만들면 바뀌지 않는다(불변).** 그래서 누가 언제 실행하든 같은 결과가 나온다.
- **컨테이너는 이미지로 여러 개** 찍어낼 수 있다. 밀키트 하나로 음식을 여러 접시 만들 수 있는 것과 같다.

### 잠깐 — PetClinic엔 Dockerfile이 없다?

맞다. Spring PetClinic 프로젝트에는 `Dockerfile`이 **없다**. 대신 Spring Boot가 제공하는 빌드 도구가 이미지를 자동으로 만들어 준다.

```bash
./gradlew bootBuildImage --imageName=spring-petclinic:1.0
# Windows(PowerShell): .\gradlew.bat bootBuildImage --imageName=spring-petclinic:1.0
```

이 명령은 내부적으로 **Cloud Native Buildpacks**라는 기술을 써서, 우리가 Dockerfile을 직접 쓰지 않아도 자바 앱에 딱 맞는 이미지를 만들어 준다. 즉, "레시피를 우리가 안 써도 Spring Boot가 알아서 써준다"고 이해하면 된다.

> **처음 실행하면 오래 걸린다.** 빌드에 필요한 재료(베이스 이미지 등)를 인터넷에서 여러 개 받아오기 때문이다. 스터디 당일 처음 돌리면 다 같이 기다리게 되니, **미리 한 번 빌드**해 두는 걸 권한다.

> ⚠️ **최신 PetClinic(Spring Boot 4.x)은 한 가지 손질이 필요하다.** 프로젝트에 GraalVM 네이티브 이미지 플러그인이 들어 있어서, 그대로 실행하면 `bootBuildImage`가 네이티브 빌드를 시도하다 **실패한다**(status code 51). `build.gradle`에서 네이티브 관련 줄들을 주석 처리하고 빌드해야 한다 — 정확한 수정 방법은 `01-환경세팅.md` 4-1에 정리해뒀다.

### 심화 — 그래도 Dockerfile을 한 번은 직접 써보자

`bootBuildImage`는 편하지만, 이대로만 쓰면 이미지가 **블랙박스**로 남는다. 레시피가 실제로 어떻게 생겼는지 한 번은 직접 써보자. 먼저 실행 가능한 jar 파일을 만든다.

```bash
./gradlew bootJar
# Windows(PowerShell): .\gradlew.bat bootJar
```

프로젝트 루트에 `Dockerfile`이라는 이름의 파일을 만들고, 주석 빼면 딱 세 줄을 적는다.

```dockerfile
# 1) 베이스 이미지 — "자바 17이 깔린 주방"에서 시작한다
FROM eclipse-temurin:17-jre

# 2) 빌드한 내 앱(jar)을 이미지 안으로 복사한다
COPY build/libs/*.jar app.jar

# 3) 컨테이너가 시작될 때 실행할 명령
ENTRYPOINT ["java", "-jar", "app.jar"]
```

읽어보면 정말 레시피 그대로다 — **어디서 시작해서(FROM), 뭘 넣고(COPY), 어떻게 조리하는가(ENTRYPOINT)**. 이제 빌드하고 실행해본다.

```bash
docker build -t spring-petclinic:manual .
# 마지막 점(.)은 "현재 폴더를 재료 창고로 쓰라"는 뜻
docker run -d --name petclinic-manual -p 8082:8080 spring-petclinic:manual
```

`http://localhost:8082`에서 같은 PetClinic이 뜬다. 이제 buildpacks가 만든 이미지와 비교해보자.

```bash
docker images                            # 두 이미지의 크기 비교
docker history spring-petclinic:manual   # 내가 쓴 세 줄이 어떤 층으로 쌓였나
docker history spring-petclinic:1.0      # buildpacks가 쓴 레시피는 몇 층인가
```

여기서 두 가지를 눈으로 확인할 수 있다.

- 이미지는 통짜 파일이 아니라 **레이어(층)를 쌓아 만든 것**이다. Dockerfile 명령 한 줄이 대략 한 층이 되고, 층은 캐시로 재사용된다. jar만 바뀌면 마지막 층만 다시 만들면 되니 빌드가 빨라지는 원리다.
- 층 개수 자체는 비슷해도 **내용이 다르다**. buildpacks 이미지에는 JVM 메모리 자동 설정(5장 심화 참고), 비-루트 사용자 실행 같은 운영 노하우가 레이어로 들어 있다. `docker inspect --format '{{.Config.User}}' <이미지>`로 비교해보면 직접 만든 이미지는 root로 돌고, buildpacks 이미지는 일반 사용자로 돈다. **세 줄짜리도 잘 돌지만, 그 차이를 눈으로 보는 게 이 심화의 목적이다.**

> `build/libs`에 jar가 여러 개 있으면(`*-plain.jar` 등) `COPY`가 실패한다. 그럴 땐 와일드카드 대신 정확한 파일명을 적자.

### Docker Hub — 이미지를 공유하는 창고

만든 이미지는 **Docker Hub**(hub.docker.com)라는 공용 저장소에 올리고 내려받을 수 있다. GitHub이 코드를 공유하는 곳이라면, Docker Hub는 이미지를 공유하는 곳이다. `docker pull`로 남이 만든 이미지를 받고, `docker push`로 내 이미지를 올린다.

---

## 3. 실습에서 만나는 명령어들 — 하나씩 뜯어보기

이번 실습의 핵심 명령을 미리 이해하고 가자. 당일에 "이게 뭐였지?" 하지 않으려면 여기를 읽어두면 된다.

### 3-1. 이미지 만들기

```bash
./gradlew bootBuildImage --imageName=spring-petclinic:1.0
# Windows(PowerShell): .\gradlew.bat bootBuildImage --imageName=spring-petclinic:1.0
```

PetClinic 소스로부터 `spring-petclinic:1.0`이라는 이름표가 붙은 이미지를 만든다. 콜론 뒤 `1.0`은 **태그(tag)**로, 버전 이름표라고 보면 된다. 안 붙이면 자동으로 `latest`가 붙는다.

### 3-2. 컨테이너 실행하기 (오늘의 핵심 한 줄)

```bash
docker run -d --name petclinic-app --restart unless-stopped --memory 1g --cpus 1 -p 8081:8080 spring-petclinic:1.0
```

길어 보이지만 옵션을 하나씩 끊으면 쉽다.

| 옵션 | 뜻 | 왜 쓰나 |
|---|---|---|
| `-d` | 백그라운드로 실행(detached) | 터미널을 닫아도 서버가 계속 돈다 |
| `--name petclinic-app` | 컨테이너에 이름 부여 | 이후 명령에서 이름으로 지목하려고 |
| `--restart unless-stopped` | 재시작 정책 | 죽으면 자동으로 다시 켜지게 (4장 참고) |
| `--memory 1g` | 메모리 상한 1GB | 서버가 메모리를 폭주시키지 못하게 (5장) |
| `--cpus 1` | CPU 1개만 사용 | CPU 독점 방지 (5장) |
| `-p 8081:8080` | 포트 연결 | 내 8081 → 컨테이너 8080 (아래 상세) |
| `spring-petclinic:1.0` | 실행할 이미지 | 어떤 밀키트를 요리할지 |

### 3-3. 포트 매핑 `-p 8081:8080` — 가장 많이 헷갈리는 부분

컨테이너는 **격리된 상자**라, 그 안에서 8080 포트로 서버가 떠 있어도 밖(내 컴퓨터)에서는 바로 접속할 수 없다. 상자에 **구멍을 뚫어 연결**해 줘야 하는데, 그게 `-p`다.

```
-p 8081:8080
   ↑     ↑
내 컴퓨터  컨테이너 내부
(호스트)   (서버가 실제로 뜬 곳)
```

읽는 법: **"내 컴퓨터의 8081로 온 요청을, 컨테이너 안 8080으로 전달하라."**

- 순서가 **호스트:컨테이너**다. 절대 헷갈리면 안 된다. 뒤집으면 엉뚱하게 동작한다.
- 그래서 브라우저에서는 `http://localhost:8081`로 접속한다. (컨테이너 안의 8080이 아니라!)

> **왜 굳이 8081로?** 지난주에 로컬에서 직접 띄운 PetClinic이 이미 8080을 쓰고 있을 수 있다. 컨테이너 서버를 8081로 빼면, **로컬 서버와 컨테이너 서버를 동시에** 띄워놓고 비교할 수 있다. "같은 앱인데 포트만 다르게 두 개가 뜬다"는 걸 눈으로 확인하는 게 이번 실습의 재미 포인트다.

### 3-4. 심화 — 같은 이미지, 다른 설정 (환경변수 주입)

2장에서 "이미지는 불변"이라고 했다. 그런데 이상하지 않은가? 개발 서버와 운영 서버는 포트도 DB 주소도 다른데, 그럼 환경마다 이미지를 새로 구워야 하나?

아니다. 설정은 이미지 안에 굽지 않고, 실행하는 순간 **밖에서 주입**한다. 그 통로가 `-e`(환경변수) 옵션이다.

```bash
docker run -d --name petclinic-env -e SERVER_PORT=9090 -p 8083:9090 spring-petclinic:1.0
```

- Spring Boot는 `SERVER_PORT` 환경변수가 있으면 그 포트로 뜬다. **같은 이미지**인데 이번엔 컨테이너 안에서 8080이 아니라 **9090**으로 떴다. 그래서 `-p`도 `8083:9090`으로 맞췄다 — 포트 매핑 개념 복습이 저절로 된다.
- `docker logs petclinic-env`에서 `Tomcat started on port 9090`을 확인하고, `http://localhost:8083`으로 접속해보자.
- 같은 원리로 `-e SPRING_PROFILES_ACTIVE=mysql`을 주면 같은 이미지가 MySQL을 바라보게 만들 수도 있다(다음 주에 다룬다).

핵심 원칙: **"빌드는 한 번, 설정은 실행할 때."** 이미지 하나를 개발·스테이징·운영 어디서든 그대로 돌리는 것이 실무 배포의 기본이다(12-Factor App 원칙). 이걸로 1장의 "이식성" 이야기가 완성된다 — 앱은 이미지에, 환경별 차이는 환경변수에.

---

## 4. 서버가 죽으면? — 재시작 정책(Restart Policy)

### 왜 필요한가

실제 서버는 별의별 이유로 죽는다. 앱에 버그가 있거나, 메모리가 부족하거나, 서버 컴퓨터가 재부팅되거나. 사람이 24시간 지켜보다가 다시 켤 수는 없으니, **"죽으면 알아서 다시 켜지게"** 설정해두는 것이 재시작 정책이다.

### 4가지 정책

| 정책 | 동작 | 언제 쓰나 |
|---|---|---|
| `no` | 자동 재시작 안 함 (기본값) | 일회성 작업 |
| `on-failure` | **오류로** 죽었을 때만 재시작 | 정상 종료는 놔두고 싶을 때 |
| `always` | 항상 재시작 (내가 stop해도, 도커 재시작 시 다시 켜짐) | 무조건 떠 있어야 하는 서버 |
| `unless-stopped` | 항상 재시작하되, **내가 직접 멈춘 건 존중** | 실무에서 가장 무난 (오늘 이걸 쓴다) |

`always`와 `unless-stopped`의 차이가 포인트다. 둘 다 죽으면 살리지만, 내가 **의도적으로** `docker stop`으로 멈춘 컨테이너를 도커 재시작 후 다시 켜느냐(`always`) 마느냐(`unless-stopped`)가 다르다. 내가 끈 건 이유가 있어 끈 거니 존중하는 `unless-stopped`가 대체로 합리적이다.

### 흔한 오해 — `docker kill` 하면 정책이 되살려줄까?

직관적으로는 `docker kill`(강제 종료)이 "서버가 갑자기 죽는 상황"의 흉내일 것 같다. 하지만 실험해보면 **되살아나지 않는다.** Docker는 `stop`이든 `kill`이든 **사용자가 명령으로 멈춘 것은 전부 "의도적 정지"로 기록**하고, 그 컨테이너에는 재시작 정책을 적용하지 않는다. 공식 문서에도 "수동으로 멈춘 컨테이너는 데몬이 재시작되거나 사용자가 직접 다시 시작하기 전까지 재시작 정책이 무시된다"고 명시돼 있다.

그럼 정책은 언제 발동하나? 딱 두 경우다.

1. **앱이 스스로 죽었을 때** — 크래시, 메모리 부족 등
2. **Docker 데몬(호스트)이 재시작됐을 때** — 여기서 `always`와 `unless-stopped`의 차이가 갈린다

한 줄로 기억하자: **"사람이 끈 건 존중하고, 스스로 죽은 것만 살린다."**

### 실습에서 확인하는 법 — 진짜 크래시를 만들어보자

사람이 죽이는 건 정책이 무시하니, 앱이 **스스로 죽는 상황**을 만들어야 한다. 가장 확실한 방법은 메모리 상한을 앱이 뜰 수 없을 만큼 낮게 거는 것이다(자원 제한은 5장에서 자세히 다룬다).

```bash
docker rm -f petclinic-app    # 기존 컨테이너 정리
docker run -d --name petclinic-app --restart unless-stopped --memory 128m -p 8081:8080 spring-petclinic:1.0

docker ps -a                                                 # STATUS: Restarting (1) ... 반복
docker inspect --format '{{.RestartCount}}' petclinic-app    # 재시작 횟수가 계속 올라간다
docker logs petclinic-app                                    # 왜 죽는지 확인 (5장 심화 참고)
```

앱이 128MB로는 뜰 수 없어 곧바로 죽고 → 정책이 되살리고 → 또 죽고를 반복한다. `Restarting` 상태와 계속 올라가는 `RestartCount`가 "정책이 일하고 있다"는 증거다. 재시도 간격은 100ms에서 시작해 2배씩 늘어난다(무한 폭주 방지).

> **함께 보면 좋다**: 두 번째 터미널에서 `docker events --filter container=petclinic-app`을 켜두면 die → start 이벤트가 실시간으로 흘러가는 게 보인다. 확인이 끝나면 `docker rm -f petclinic-app`으로 지우고 정상 옵션(3-2의 명령)으로 다시 띄워두자.

> **참고**: 셸이 있는 이미지라면 `docker exec <이름> kill 1`로 컨테이너 안의 프로세스를 직접 죽여 크래시를 흉내 낼 수도 있다. 하지만 **우리가 만든 PetClinic 이미지(tiny 계열)에는 셸 자체가 없어서 이 방법이 안 된다** — `docker exec petclinic-app ls`조차 "executable file not found"로 실패한다. 그래서 메모리 제한 방식을 쓰는 것이다. (덤: 셸이 없다는 건 공격자가 침투해도 할 수 있는 게 없다는 뜻이라, 보안상 장점이기도 하다)

한편 `docker stop`으로 멈춘 컨테이너는 사람이 끈 것이므로 당연히 되살아나지 않는다. `always`였다면? stop한 컨테이너도 **Docker 데몬이 재시작될 때** 다시 켜진다. Docker Desktop을 껐다 켜보면 두 정책의 차이를 직접 확인할 수 있다.

---

## 5. 서버가 자원을 폭주시키면? — CPU·메모리 제한

### 왜 필요한가

컨테이너는 기본적으로 호스트의 자원을 **무제한**으로 쓸 수 있다. 앱에 메모리 누수 버그라도 있으면, 컨테이너 하나가 서버 전체 메모리를 다 먹어치워서 다른 서비스까지 다 같이 죽을 수 있다. 그래서 **"너는 여기까지만 써"** 하고 상한선을 그어준다.

```
--memory 1g   → 메모리는 최대 1GB까지만
--cpus 1      → CPU는 1코어 분량까지만
```

메모리 상한을 넘으면 컨테이너 안의 프로세스가 강제 종료된다. 이걸 **OOM Kill**(Out Of Memory Kill)이라고 부른다.

### 실습에서 확인하는 법

```bash
docker stats petclinic-app
```

`docker stats`는 CPU·메모리 사용량을 **실시간으로** 보여준다. 화면이 계속 갱신되니, 다 봤으면 `Ctrl + C`로 빠져나오면 된다. 여기서 `MEM USAGE / LIMIT`이 우리가 건 `1g` 안에서 움직이는 걸 확인할 수 있다.

> **심화(선택)**: 4장에서 `--memory 128m`로 "뜨지 못하고 죽는" 장면을 이미 봤다. 그때 `docker logs`를 보면, 흔히 기대하는 OOM Kill이 아니라 `unable to calculate memory configuration — fixed memory regions require 636104K which is greater than 128M ...` 같은 **"메모리 계산 실패"** 에러가 보인다(필요한 최소 메모리까지 친절하게 알려준다). buildpacks로 만든 이미지는 시작할 때 **컨테이너의 메모리 상한을 읽어 JVM 메모리를 자동 배분**하는데(memory calculator), 128MB로는 그 계산이 성립하지 않아 앱이 시작조차 못 하는 것이다. "이미지가 자원 제한을 인식하고 스스로 맞춘다"는 것 자체가 buildpacks의 정교함을 보여주는 포인트다. 어느 쪽이든 교훈은 같다 — 제한을 잘못 걸면 서버가 뜨다 만다.

---

## 6. 서버가 안 열리면? — 관찰(Observability) 3종 세트

서버가 응답이 없을 때, 무작정 껐다 켜지 말고 **어디를 볼지** 순서가 있다. 이 3개가 실무에서 제일 먼저 치는 명령이다.

| 명령 | 무엇을 보나 | 언제 |
|---|---|---|
| `docker ps` | 컨테이너가 **살아 있나?** | 제일 먼저 |
| `docker logs` | 앱이 **뭐라고 하다 죽었나?** | 떠 있는데 응답이 없을 때 |
| `docker inspect` | 설정이 **내 의도대로 걸렸나?** | 포트·재시작·자원 확인 |

```bash
docker ps                        # 실행 상태 (죽은 건 docker ps -a 로)
docker logs --tail 80 petclinic-app   # 최근 로그 80줄
docker inspect petclinic-app     # 포트/재시작/자원 등 전체 설정(JSON)
```

- `docker ps`에 안 보이면? → 컨테이너가 죽은 것. `docker ps -a`로 확인하고 `docker logs`로 원인을 본다.
- 떠 있는데 브라우저가 안 열리면? → 포트 매핑을 잘못 걸었거나, 앱이 아직 **부팅 중**일 수 있다.
- `docker inspect`는 출력이 매우 길다(JSON 전체). 특정 값만 보려면 필터를 쓴다.
  ```bash
  docker inspect --format '{{.State.Status}}' petclinic-app
  ```

### 컨테이너가 Up이어도 앱은 아직 준비 안 됐을 수 있다

중요한 개념이다. `docker ps`에 `Up`으로 떠도, 그건 **컨테이너(상자)가 실행됐다**는 뜻이지 **안의 앱이 요청받을 준비가 끝났다**는 뜻이 아니다. 자바 앱은 부팅에 수십 초 걸리기도 한다. 그래서 컨테이너 상태와 애플리케이션 상태는 **다르다.**

이걸 확인하는 게 헬스체크 엔드포인트다.

```
http://localhost:8081/actuator/health
```

여기서 `"status":"UP"`이 포함된 JSON(예: `{"groups":["liveness","readiness"],"status":"UP"}`)이 나와야 **앱까지** 준비된 것이다. "컨테이너 Up ≠ 앱 UP"을 이 화면으로 눈에 익히면 된다.

---

## 7. 이미지를 남에게 넘기려면? — save / load

만든 이미지를 다른 사람 컴퓨터에서 실행하는 두 가지 방법이 있다.

### 방법 A — 파일로 주고받기 (오늘 기본 실습)

```bash
# 내 컴퓨터: 이미지를 tar 파일 하나로 뽑는다
docker save -o spring-petclinic-1.0.tar spring-petclinic:1.0

# 다른 컴퓨터: 그 tar 파일을 이미지로 불러온다
docker load -i spring-petclinic-1.0.tar
docker run -d --name petclinic-app -p 8081:8080 spring-petclinic:1.0
```

`save`는 이미지를 **파일 한 개**(tar)로 묶어서 뽑는다. 압축은 아니라서 파일이 꽤 큰데, 줄이고 싶으면 zip 등으로 한 번 더 싸면 된다. USB나 메신저로 그 파일을 넘기면, 받는 쪽은 `load`로 되살려 바로 실행할 수 있다. 인터넷 없이도 이미지를 통째로 전달할 수 있다는 게 장점이다. **이게 곧 "이식성"의 증명이다** — 소스코드도, 자바 설치도 없이 이미지 파일 하나로 같은 서버가 뜬다.

### 방법 B — Docker Hub로 올리고 받기 (시간 되면 선택 실습)

```bash
# 올리기 (미리 docker login 필요, 이미지 이름은 계정명/이름 형식)
docker tag spring-petclinic:1.0 myaccount/spring-petclinic:1.0
docker push myaccount/spring-petclinic:1.0

# 받기 (아무 컴퓨터에서나)
docker pull myaccount/spring-petclinic:1.0
```

GitHub에 코드 올리듯, Docker Hub에 이미지를 올려두면 누구나 `pull`로 받을 수 있다. 실무 배포는 대부분 이 방식(레지스트리)을 쓴다.

---

## 8. 오늘의 전체 흐름 한눈에

```
1) 준비 확인      docker --version / docker info
2) 이미지 빌드    ./gradlew bootBuildImage --imageName=spring-petclinic:1.0
3) 컨테이너 실행  docker run -d --name ... --restart ... --memory ... --cpus ... -p 8081:8080 ...
4) 브라우저 확인  http://localhost:8081  /  /actuator/health
5) 관찰          docker ps / docker logs / docker stats / docker inspect
6) 죽여보기      --memory 128m 로 재실행 → 스스로 죽는 앱을 정책이 되살리는 것 확인
7) 이미지 전달    docker save → (다른 컴퓨터) docker load → run
```

각 단계가 0장의 질문 하나씩에 대응한다.

- 2~4번 → "어디서든 같은 서버를 띄울 수 있는가"
- 3번의 `-p 8081:8080` → "로컬 서버와 컨테이너 서버를 동시에 띄울 수 있는가"
- 6번 → "죽으면 어떻게 되는가"
- 3번의 `--memory/--cpus` + 5번의 stats → "자원을 너무 쓰면 어떻게 막는가"
- 5번 → "안 열리면 어디를 보는가"
- 7번 → "다른 사람 컴퓨터에서도 되는가"

### 실습 후 뒷정리

컨테이너와 이미지는 지우기 전까지 계속 디스크를 차지한다. 실습이 끝나면 정리하자.

```bash
docker rm -f petclinic-app                    # 컨테이너 삭제 (-f: 실행 중이어도 강제로)
docker rm -f petclinic-manual petclinic-env   # 심화 실습을 했다면 이것도
docker rmi spring-petclinic:1.0               # 이미지 삭제 (그 이미지의 컨테이너를 먼저 지워야 한다)
docker system df                              # Docker가 디스크를 얼마나 쓰는지 확인
docker system prune                           # 안 쓰는 것 일괄 정리 (안내문을 읽고 y)
```

---

## 9. 이번 주에 꼭 가져갈 것

명령어를 다 외울 필요는 없다. **이 문장들만 자기 말로 설명할 수 있으면** 이번 주는 성공이다.

- 컨테이너는 앱과 그 실행 환경을 통째로 담은 상자라, 어디서든 똑같이 돈다.
- Dockerfile(레시피) → Image(밀키트) → Container(요리)의 관계. 이미지는 **레이어를 쌓아** 만든다.
- `-p 호스트:컨테이너`는 상자에 구멍을 뚫어 밖과 연결하는 것.
- 재시작 정책은 **스스로 죽은 것만** 살린다 — `stop`이든 `kill`이든 사람이 끈 건 존중한다.
- 설정은 이미지에 굽지 않는다 — **빌드는 한 번, 설정은 실행할 때**(`-e` 환경변수).
- 자원 제한으로 "한 컨테이너가 서버 전체를 마비시키는 것"을 막는다.
- 안 열릴 땐 `ps → logs → inspect` 순서로 본다.
- `Up`(컨테이너)과 `UP`(앱 헬스체크)은 다르다.

---

## 참고 자료

- [Spring PetClinic 공식 리포](https://github.com/spring-projects/spring-petclinic) — Dockerfile 없이 `bootBuildImage`로 이미지를 만든다.
- [Docker 공식 문서 — 컨테이너란](https://www.docker.com/resources/what-container/)
- [Docker restart policies 문서](https://docs.docker.com/engine/containers/start-containers-automatically/)
- [Docker resource constraints 문서](https://docs.docker.com/engine/containers/resource_constraints/)
- 1주차 자료: `likelion-server-study/WEEK1` — PetClinic 로컬 실행과 상태·로그 확인
