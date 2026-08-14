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

<!-- 직접 작성. 아래 세 질문에 답하는 문장이면 된다:
  ① go.mod가 요구한 버전과 빌더 이미지에 깔려 있던 버전은 각각 무엇인가?
  ② 에러 메시지의 GOTOOLCHAIN=local은 무엇을 막았는가?
  ③ 왜 로컬 go run에서는 이 문제를 못 느꼈는가? -->

1. 1.26.5 : go.mod
   1.22 : docker file
2.  Go 언어가 네트워크를 통해 최신 버전의 Go 툴체인(컴파일러 등)을 자동으로 다운로드하여 실행하는 것을 막았습니다
3. 로컬에선 GOTOOLCHAIN=auto가 자동적으로 내포되어있어, 높은 버전을 요구하면 자동으로 내려받아서 쓴다. 

## 해결 (Resolution)

빌더 이미지의 Go 버전을 go.mod의 요구에 맞춰 올렸다.

```diff
-FROM golang:1.22-alpine AS builder
+FROM golang:1.26.5-alpine AS builder
```

go.mod를 낮추는 방향은 택하지 않았다.
<!-- 왜 이미지 쪽을 바꿨는지 한 문장 직접 작성.
     힌트: go.mod의 버전은 누구(어느 역할)의 요구 선언인가? -->

: "go.mod는 앱 코드가 전제하는 요구 선언이고, 낮추면 코드가 깨질 위험이 있어 빌드 환경을 맞췄다. 조직 차원의 런타임 표준을 도입한다면 별도 정책으로 다룬다(→ 골든 이미지, Phase 3 CI 검사).

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
명시. Phase 3에서 두 값의 일치를 CI 검사로 강제하는 것을 검토한다.

## 배운 것

<!-- 직접 작성: 다음에 "로컬에선 되는데 컨테이너 빌드는 실패"를 만나면
     어디부터 볼 것인가? -->
1. 에러 메시지를 그대로 읽자
2. 로컬과 컨테이너의 환경 변수가 다르니 유의하자
