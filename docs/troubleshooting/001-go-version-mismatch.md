# 001: docker build가 go mod download 단계에서 Go 버전 에러로 실패

- **날짜**: 2026-08-14
- **Phase**: Phase 1
- **소요 시간**: (진단~해결까지 — 직접 기입)

## 증상 (Symptom)

로컬에서 `go run`은 정상 동작하지만, `docker build`는 `RUN go mod download`
단계에서 실패했다.

```
 => ERROR [builder 4/6] RUN go mod download                                          0.3s
------
 > [builder 4/6] RUN go mod download:
0.281 go: go.mod requires go >= 1.26.5 (running go 1.22.12; GOTOOLCHAIN=local)
------
Dockerfile:7
ERROR: failed to build: failed to solve: process "/bin/sh -c go mod download"
did not complete successfully: exit code: 1
```

## 원인 (Root Cause)

`go.mod`는 Go 1.26.5 이상을 요구하는데, 빌더 이미지 `golang:1.22-alpine`에는
Go 1.22.12가 설치되어 있었다.

공식 golang 이미지는 `GOTOOLCHAIN=local`로 설정되어 있어, 요구 버전이 더 높아도
새 툴체인(컴파일러 등)을 네트워크로 자동 다운로드하지 않고 빌드를 거부한다.

로컬에서 문제를 못 느낀 이유는 로컬 Go의 기본값이 `GOTOOLCHAIN=auto`이기
때문이다. go.mod가 더 높은 버전을 요구하면 해당 툴체인을 자동으로 내려받아
조용히 그걸로 실행하므로, 같은 버전 불일치가 로컬에서는 드러나지 않았다.

참고: `go.mod`의 `go` 선언은 "정확히 이 버전"이 아니라 "최소 이 버전 이상"이며,
어떤 설정에서도 무시되지 않는다. `auto`는 다운로드로 요구를 충족시키고,
`local`은 실패로 알려준다는 차이만 있다.

## 해결 (Resolution)

빌더 이미지의 Go 버전을 go.mod의 요구에 맞춰 올렸다.

```diff
-FROM golang:1.22-alpine AS builder
+FROM golang:1.26.5-alpine AS builder
```

go.mod를 낮추는 방향은 택하지 않았다. go.mod는 앱 코드가 전제하는 요구
선언이므로, 낮추면 1.26 기능을 사용한 코드가 깨질 위험이 있어 빌드 환경
쪽을 요구에 맞췄다. 조직 차원의 런타임 표준을 도입하는 경우는 별도
정책으로 다룬다(→ 골든 이미지, Phase 3 CI 검사).

## 검증 (Verification)

재빌드 성공 확인:

```
$ docker build -t api-server:test ./
[+] Building 8.1s (16/16) FINISHED
 => [builder 4/6] RUN go mod download                                                0.9s
 => [builder 6/6] RUN CGO_ENABLED=0 GOOS=linux go build -o /server cmd/server/main.go
 => => naming to docker.io/library/api-server:test
```

재발 방지: go.mod와 Dockerfile의 Go 버전을 함께 올려야 함을 Dockerfile 주석으로
명시했다. Phase 3에서 두 값의 일치를 CI 검사로 강제하는 것을 검토한다.

## 배운 것

1. 에러 메시지를 문자 그대로 읽는다. 이번 에러는 요구 버전, 실제 버전,
   원인 설정(`GOTOOLCHAIN=local`)까지 한 줄에 전부 담고 있었다.
2. "로컬에선 되는데 컨테이너 빌드는 실패"를 만나면 두 환경의 차이부터
   본다 — 설치된 버전과 환경변수(`GOTOOLCHAIN` 등)가 다를 수 있다.
