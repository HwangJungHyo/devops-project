# 로컬 개발 환경 실행

> 이 문서는 `api-server`를 로컬에서 처음 실행하거나, 코드 변경 후 컨테이너 동작을 검증할 때 사용한다.

## 사전 요구사항

이 문서는 다음 환경에서 따라하기 검증됐다. 이보다 오래된 버전에서는
일부 명령(`--wait-timeout` 등)이 없을 수 있다.

- Docker Desktop 4.x.x (Engine 2x.x.x)
- Docker Compose v2.xx.x

다음 도구가 설치되어 있어야 한다.

* Git
* Docker Desktop
* Docker Compose v2
* HTTP 요청 확인용 `curl`

Docker Desktop이 실행 중인지 확인한다.

```bash
docker version
docker compose version
```

두 명령 모두 Client와 Server 또는 Compose 버전 정보를 출력해야 한다. Docker Server 연결 오류가 발생하면 Docker Desktop을 먼저 실행한다.

저장소가 없다면 다음과 같이 clone한다.

```bash
cd ~/Desktop/git-devops

git clone https://github.com/mentoring-devops/api-server.git

cd api-server
```

이미 clone한 경우 저장소 루트로 이동한다.

```bash
cd ~/Desktop/git-devops/api-server
```

현재 위치에 필요한 파일이 있는지 확인한다.

```bash
ls
```

최소한 다음 파일과 디렉터리가 보여야 한다.

```text
Dockerfile
docker-compose.yml
go.mod
go.sum
cmd/
internal/
```

## 환경변수 준비

애플리케이션에는 다음 환경변수가 필요하다.

| 변수           | 필수 | 설명                     | 로컬 예시   |
| ------------ | -- | ---------------------- | ------- |
| `PORT`       | 필수 | 컨테이너 내부 HTTP 서버 포트     | `8080`  |
| `HOST_PORT`  | 선택 | PC에 공개할 포트             | `8080`  |
| `JWT_SECRET` | 필수 | JWT 생성 및 검증에 사용하는 비밀 키 | 랜덤 64글자 |

`.env.example`을 복사해 실제 로컬 설정 파일을 만든다.

```bash
cp .env.example .env
```

JWT 비밀 키를 생성한다.

```bash
openssl rand -hex 32
```

출력된 값을 복사한 후 `.env`를 수정한다.

```bash
vim .env
```

구조는 다음과 같다.

```dotenv
PORT=8080
HOST_PORT=8080
JWT_SECRET=openssl로_새로_생성한_64글자
```

> `.env`에는 실제 비밀값이 들어 있으므로 절대 Git에 커밋하지 않는다. `.env.example`에는 실제 비밀값 대신 설명용 예제값만 기록한다.

`.gitignore`와 `.dockerignore`에 `.env`가 포함돼 있는지 확인한다.

```bash
grep -n '^\.env$' .gitignore .dockerignore
```

Git이 `.env`를 무시하는지 확인한다.

```bash
git check-ignore -v .env
```

정상이면 `.env`를 제외한 `.gitignore` 규칙이 출력된다.

JWT 값을 노출하지 않고 길이만 확인한다.

```bash
awk -F= '$1=="JWT_SECRET" {print "JWT_SECRET length=" length($2)}' .env
```

예상 결과:

```text
JWT_SECRET length=64
```

Compose 파일과 환경변수 설정을 검증한다.

```bash
docker compose config --quiet
```

정상이면 아무것도 출력하지 않고 종료한다.

`docker compose config`를 옵션 없이 실행하면 보간된 `JWT_SECRET`이 터미널에 출력될 수 있으므로 공유 화면이나 로그에서는 사용하지 않는다.

## 기동

이미 8080 포트를 사용하는 기존 컨테이너가 있는지 확인한다.

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

WSL 내부에서 실행 중인 서버도 확인한다.

```bash
wsl.exe -e sh -lc 'ss -ltnp | grep ":8080" || true'
```

Compose로 이미지를 빌드하고 컨테이너를 백그라운드에서 실행한다.

```bash
docker compose up -d --build
```

컨테이너 상태를 확인한다.

```bash
docker compose ps
```

기동 직후에는 다음처럼 표시될 수 있다.

```text
STATUS
Up ... (health: starting)
```

현재 healthcheck 설정은 다음 시간을 사용한다.

```text
start_period: 5초
interval:     5초
retries:      3회
```

최대 약 20초 동안 기다린 후 다시 확인한다.

```bash
docker compose ps
```

성공 기준:

```text
STATUS
Up ... (healthy)
```

자동으로 `healthy` 상태까지 기다리려면 다음 명령을 사용할 수 있다.

```bash
docker compose up \
  -d \
  --build \
  --wait \
  --wait-timeout 20
```

20초 후에도 `starting`이거나 `unhealthy`라면 [로그 확인](#로그-확인)과 [문제 발생 시](#문제-발생-시)를 확인한다.

## 상태 확인

### Compose 상태 확인

```bash
docker compose ps
```

정상 기준:

* 서비스가 `api`로 표시된다.
* 컨테이너가 `Up` 상태다.
* Health 상태가 `healthy`다.
* `8080` 포트가 컨테이너의 `8080`으로 연결돼 있다.

### 컨테이너 내부 사용자 확인

```bash
docker compose exec api id
```

예상 결과:

```text
uid=10001(appuser) gid=10001(appgroup)
```

`uid=0(root)`가 나오면 non-root 설정이 적용되지 않은 것이다.

### IPv4 healthz 확인

```bash
curl -4 -i http://localhost:8080/healthz
```

예상 결과:

```text
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8

ok
```

### IPv6 healthz 확인

```bash
curl -6 -i http://localhost:8080/healthz
```

IPv4와 동일하게 `200 OK`와 `ok`가 반환돼야 한다.

`curl -4`는 성공하지만 `curl -6`이 404를 반환한다면 IPv4와 IPv6 요청이 서로 다른 프로세스로 전달되고 있을 수 있다. 자세한 조사 방법은 [Troubleshooting 002](../troubleshooting/002-localhost-ipv6-listener.md)를 참고한다.

### 컨테이너 내부 healthcheck 확인

Compose healthcheck는 Git Bash가 아니라 Alpine 컨테이너 안에서 실행된다.

컨테이너 내부에 `wget`이 존재하는지 확인한다.

```bash
docker compose exec api sh -c 'command -v wget'
```

healthcheck 명령을 직접 실행한다.

```bash
docker compose exec api sh -c \
  'wget --spider -q http://127.0.0.1:$PORT/healthz && echo "healthcheck success"'
```

예상 결과:

```text
healthcheck success
```

## 로그 확인

전체 로그를 확인한다.

```bash
docker compose logs api
```

최근 100줄만 확인한다.

```bash
docker compose logs --tail=100 api
```

로그를 실시간으로 확인한다.

```bash
docker compose logs -f api
```

실시간 로그 확인은 `Ctrl+C`로 종료한다. 이때 로그 조회만 종료되며 컨테이너는 계속 실행된다.

정상 기동 로그 예시:

```text
Server starting on :8080
```

컨테이너가 비정상 종료됐다면 정지된 컨테이너까지 확인한다.

```bash
docker compose ps -a
```

## 종료

컨테이너와 Compose 네트워크를 종료하고 제거한다.

```bash
docker compose down
```

`docker compose down`은 다음 항목을 제거한다.

* Compose가 생성한 컨테이너
* Compose가 생성한 기본 네트워크

빌드된 `api-server:local` 이미지는 유지하므로 다음 실행에서 재사용할 수 있다.

컨테이너만 일시 정지하고 나중에 다시 시작하려면 다음 명령을 사용한다.

```bash
docker compose stop
```

```bash
docker compose start
```

이미지까지 제거하려면 다음 명령을 사용한다.

```bash
docker compose down --rmi all
```

차이는 다음과 같다.

| 명령                              | 컨테이너 | 네트워크 | 이미지 |
| ------------------------------- | ---- | ---- | --- |
| `docker compose stop`           | 유지   | 유지   | 유지  |
| `docker compose down`           | 제거   | 제거   | 유지  |
| `docker compose down --rmi all` | 제거   | 제거   | 제거  |

일반적인 개발 종료에는 `docker compose down`을 사용하고, 이미지를 처음부터 다시 검증해야 할 때만 `--rmi all`을 사용한다.

## 문제 발생 시

### `PORT must be set`

예시:

```text
required variable PORT is missing a value
```

Compose가 `.env`를 찾지 못했거나 `PORT`가 비어 있는 상태다.

```bash
ls -la .env
grep -E '^(PORT|HOST_PORT)=' .env
docker compose config --quiet
```

`.env.example`은 자동으로 로드되지 않는다. 실제 실행에는 파일명이 정확히 `.env`인 파일이 필요하다.

### 8080 포트 선점

Windows에서 8080 리스너를 확인한다.

```bash
netstat -ano | findstr ":8080"
```

출력 마지막 열의 PID를 확인한다.

```text
TCP  0.0.0.0:8080  ...  LISTENING  1234
```

PID의 프로세스를 조회한다.

```bash
powershell.exe -NoProfile -Command \
  'Get-CimInstance Win32_Process -Filter "ProcessId = 1234" | Select-Object ProcessId,Name,CommandLine | Format-List'
```

WSL 내부 리스너도 확인한다.

```bash
wsl.exe -e sh -lc 'ss -ltnp | grep ":8080" || true'
```

확인하지 않은 PID를 임의로 종료하지 않는다. 프로세스의 정체와 실행 경로를 확인한 뒤 기존 개발 서버나 컨테이너만 종료한다.

호스트 포트를 임시로 변경하려면 `.env`에서 다음 값을 수정한다.

```dotenv
HOST_PORT=18080
```

이 경우 확인 주소도 변경한다.

```bash
curl -4 -i http://localhost:18080/healthz
curl -6 -i http://localhost:18080/healthz
```

### Health 상태가 `starting` 또는 `unhealthy`

애플리케이션 로그를 확인한다.

```bash
docker compose logs --tail=100 api
```

컨테이너 내부 healthcheck를 직접 실행한다.

```bash
docker compose exec api sh -c \
  'wget --spider -S http://127.0.0.1:$PORT/healthz'
```

컨테이너 내부 환경변수는 값을 노출하지 않고 존재 여부만 확인한다.

```bash
docker compose exec api sh -c \
  'printf "PORT=%s\n" "$PORT"; test -n "$JWT_SECRET" && echo "JWT_SECRET=set"'
```

필요하면 컨테이너를 재생성한다.

```bash
docker compose down

docker compose up -d --build
```

추가 사례는 다음 문서를 참고한다.

* [Troubleshooting 001 — Go 버전 불일치로 빌드 실패](../troubleshooting/001-go-version-mismatch.md)
* [Troubleshooting 002 — localhost IPv4/IPv6 응답 불일치](../troubleshooting/002-localhost-ipv6-listener.md)

## 참고 문서

* [Docker Compose 시작하기](https://docs.docker.com/compose/gettingstarted/)
* [Docker Compose 환경변수 보간](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
* [Docker Compose 서비스와 healthcheck](https://docs.docker.com/reference/compose-file/services/#healthcheck)
* [`docker compose up`](https://docs.docker.com/reference/cli/docker/compose/up/)
* [`docker compose down`](https://docs.docker.com/reference/cli/docker/compose/down/)

