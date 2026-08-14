# ADR-0001: 런타임 베이스 이미지로 alpine을 사용한다 (Phase 8 재검토)

- **상태**: Accepted
- **날짜**: 2026-08-14
- **관련 Phase**: Phase 1
- **영향받는 후속 Phase**: Phase 8 (distroless 전환 재검토), Phase 11 (취약점 스캔 범위)

## 맥락 (Context)

컨테이너화(Phase 1)를 시작하며 런타임 베이스 이미지를 정해야 한다.
앱은 `CGO_ENABLED=0`으로 빌드되는 정적 Go 바이너리라 런타임에 OS
라이브러리가 필요 없고, 따라서 선택지는 scratch / distroless/static /
alpine 세 가지다.

## 결정 (Decision)

alpine 이미지를 사용한다. Phase 8(로그 중앙화) 완료 후 distroless 전환을 재검토한다.

## 고려한 선택지 (Options Considered)

### scratch — 기각

- 내용물: 완전히 빈 이미지 (0MB)
- 쉘 접속 (`docker exec sh`): 불가
- HTTPS 외부 호출: CA 인증서 직접 복사해야 함
- 취약점 표면: 없음 (스캔할 것 자체가 없음)
- non-root: 유저를 직접 만들어야 함
- **기각 사유**: distroless/static이 같은 보안 수준에 CA 인증서·nonroot
  유저를 기본 제공하므로, scratch를 직접 구성하여 시간을 소모할 필요 없음

### distroless/static — 기각 (Phase 8 재검토 대상)

- 내용물: CA 인증서 + 타임존 + 유저 정보만 (~2MB)
- 쉘 접속 (`docker exec sh`): 불가
- HTTPS 외부 호출: 기본 포함
- 취약점 표면: 거의 없음
- non-root: `:nonroot` 태그 제공
- **기각 사유**: 현재는 모니터링 시스템이 갖춰지기 전이므로 기각하지만,
  Phase 8 이후 추후 도입 예정

### alpine — 채택

- 내용물: 미니 리눅스 — 쉘, 패키지 매니저 포함 (~7MB)
- 쉘 접속 (`docker exec sh`): 가능
- HTTPS 외부 호출: `apk add ca-certificates`
- 취약점 표면: apk 패키지들이 CVE 대상이 됨
- non-root: 유저를 직접 만들어야 함
- **채택 사유**: 즉시 운영 환경에서 사용한다 가정할 경우 distroless가
  이상적이지만, 현재 모니터링 시스템이 갖춰지기 전에 쉘 없는 이미지를
  사용하게 되면 간단한 확인 작업도 하기 어려운 상태로 바뀌어버린다.

## 결과 (Consequences)

- **좋아지는 것**: 디버깅 편의, 학습 속도
- **감수하는 것**: apk 패키지의 CVE 노출, 쉘 존재, 전환을 미룰수록 리스크 증가
- **후속 작업**:
  - 쉘 의존 최소화를 유지한다 (헬스체크 등을 쉘 명령에 기대지 않게 구성 —
    전환 비용을 "FROM 한 줄"로 유지하기 위함)
  - Phase 8 이후 런타임 베이스를 distroless로 교체하는 작업

## 재검토 트리거 (Revisit When)

- Phase 8 로그 중앙화 완료 시 distroless 전환 검토

## 참고 자료

- [GoogleContainerTools/distroless](https://github.com/GoogleContainerTools/distroless) — distroless의 목적과 구성
- [Docker 공식 Dockerfile best practices](https://docs.docker.com/build/building/best-practices/) — 멀티스테이지, 베이스 이미지 선정
- [alpine 공식 이미지](https://hub.docker.com/_/alpine)
