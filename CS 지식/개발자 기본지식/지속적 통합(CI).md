# 지속적 통합(CI)
----

**CI(Continuous Integration, 지속적 통합)** 는 코드 변경을 **자주 메인 브랜치에 합치고, 합칠 때마다 빌드·테스트를 자동으로 돌려** 깨진 걸 바로 알아내는 관행이다. 짝인 **CD(Continuous Delivery/Deployment)** 는 그 결과물을 **배포 가능한 상태로 만들거나 실제로 배포까지** 자동으로 이어 간다. 둘을 묶어 CI/CD라 부르며, 이 노트에서 함께 다룬다.

## 1. 왜 필요한가

통합을 미룰수록 충돌이 커진다. 각자 2주 작업하고 한꺼번에 합치면, 무엇이 무엇을 깨뜨렸는지 아무도 모른다. CI의 발상은 단순하다 — **아프면 자주 해서 작게 아프자.**

[[Git 브랜치 전략]]의 GitHub Flow가 "`main` 은 항상 배포 가능하다"는 불변식 하나에 기대는데, 그 불변식을 **사람의 주의력이 아니라 기계가** 지키게 하는 게 CI다. 로컬의 [[Git 훅(Git Hooks)]]은 `--no-verify` 로 건너뛸 수 있는 안전벨트일 뿐이고, 진짜 게이트는 서버에서 도는 CI다.

## 2. CI · CD 구분

| 단계 | 의미 | 자동화 끝점 |
| ----------- | ----------- | ----------- |
| **CI** 지속적 통합 | 머지마다 빌드·린트·테스트 | "합쳐도 안 깨진다"는 확인 |
| **CD** 지속적 전달 (Delivery) | 언제든 배포할 수 있는 결과물을 자동 생성 | **배포 버튼은 사람이** 누름 |
| **CD** 지속적 배포 (Deployment) | 통과하면 운영까지 자동 배포 | 사람 개입 없음 |

같은 약어 CD가 두 의미로 쓰여 헷갈린다. 판단 기준: **"운영 반영을 사람이 승인하나?"** — 승인하면 Delivery, 안 하면 Deployment.

```
push → [빌드] → [린트·타입체크] → [테스트] → [아티팩트 생성] → [스테이징] → [운영]
       └──────────── CI ────────────┘   └──── CD(Delivery) ────┘  └ Deployment ┘
```

## 3. 웹·서버 vs 앱 — 파이프라인 모양이 다르다

**결과물이 무엇이고 어디로 가는가**가 다르기 때문이다.

| | 웹·서버 | 모바일 앱 |
| ----------- | ----------- | ----------- |
| 결과물 | 정적 파일 · Docker 이미지 | **APK/AAB · IPA** |
| 어디로 | 내 서버·클러스터 | **테스터 기기 · 스토어** |
| 배포 후 반영 | 즉시 (서버 교체) | **심사 대기** + 사용자가 업데이트해야 |
| 추가 단계 | — | **[[코드 서명(Code Signing)]] · 스토어 제출 · [[OTA 업데이트]]** |
| 빌드 환경 | Linux면 충분 | iOS는 **macOS 필수** |
| 롤백 | 이전 이미지로 | 스토어 버전은 롤백 불가, OTA로만 빠르게 |

그래서 앱 CI/CD의 단계는 `빌드 → 서명 → (내부 배포 | 스토어 제출) → OTA` 가 된다. Expo 생태계에서는 이걸 [[EAS Build]] · [[EAS Submit]] · [[EAS Update]]가 각각 맡고, **EAS Workflows** 가 이를 YAML로 엮는다.

```yaml
# .eas/workflows/build.yml (공식 문서 예시)
name: Create Production Builds
on:
  push:
    branches: ['main']
jobs:
  build_android:
    type: build
    params: { platform: android, profile: production }
  build_ios:
    type: build
    params: { platform: ios, profile: production }
```

> EAS Workflows 기준 (2026-10-15 확인) — [Get started](https://docs.expo.dev/eas/workflows/get-started/) · [Syntax](https://docs.expo.dev/eas/workflows/syntax/)
> 잡 종류: `build` · `submit` · `update` · `fingerprint` · `maestro`(E2E 테스트) · `slack` 등
> **트리거(`on`)는 GitHub 이벤트(push·PR·PR 코멘트 등)와 스케줄·App Store Connect 이벤트.** 문서에 GitLab 연동 언급은 없다. 대신 `eas workflow:run` 으로 `on` 과 무관하게 수동 실행이 가능하므로, GitLab CI에서 이 명령을 호출하는 식으로 붙일 수는 있다 (이 조합은 미검증)

## 4. 장단점

### 장점
* 깨진 코드가 `main` 에 들어가기 전에 막힌다
* "내 컴퓨터에선 되는데"가 사라진다 — 깨끗한 환경에서 매번 빌드
* 배포가 버튼 하나(또는 0개)가 되어 배포 공포가 줄고, 자주 배포하게 된다

### 단점·함정
* **느린 CI는 아무도 안 기다린다.** 20분 걸리면 사람들이 결과를 안 보고 머지한다. 캐시·병렬화가 필수 → [[Docker 멀티스테이지 빌드]]의 레이어 캐시가 여기서 값을 한다
* **테스트가 없으면 CI는 "빌드 되네" 확인기**일 뿐이다. 안전망의 품질은 테스트의 품질
* 불안정한(flaky) 테스트가 하나만 있어도 신뢰가 무너진다. "또 그거네" 하고 재실행 버튼을 누르는 순간 CI는 무력화
* 비밀값(서명 키·API 키·Service Account)이 CI에 모인다 → CI 자체가 공격 대상. 권한 최소화·마스킹
* 앱은 빌드 한 번이 비싸다(수 분~수십 분, 무료 플랜 횟수 제한). **PR마다 네이티브 빌드를 돌리면 금방 한도 소진** — PR에는 린트·테스트·OTA 미리보기만, 네이티브 빌드는 `main` 이나 태그에서만

### 판단 기준

| 상황 | 권장 |
| ----------- | ----------- |
| 팀 3명, 테스트 거의 없음 | CI부터: 린트 + 타입체크 + 빌드만이라도 PR에 걸기 |
| 운영 반영에 책임자 승인 필요 | CD는 Delivery까지. 배포는 수동 승인 |
| 스토어 앱 | 스토어 제출은 Delivery(사람이 심사 제출), JS 수정은 OTA로 Deployment에 가깝게 |

## 5. 펫바디 사례

**웹·서버**: 현재 CI는 없고, Makefile 기반 **수동 파이프라인**이다.

```
make (배포 타깃 — 타깃 이름은 생략)
  → docker build        (멀티스테이지)
  → docker save         (이미지를 tar로)
  → containerd import   (사내 노드에 직접 적재 — 레지스트리 없음)
  → kubectl apply       (사내 [[쿠버네티스(Kubernetes)]])
```

CD 관점에서 보면 "배포 가능한 결과물을 만드는 단계"는 스크립트로 자동화돼 있지만 **트리거가 사람**이고, 머지 전 검증(CI)은 비어 있다. 사내 GitLab을 쓰므로 GitLab CI에 이 Makefile을 그대로 태우는 게 가장 작은 첫걸음이다 (팀 미논의).

**앱(펫바타)**: 사내 k8s는 Linux 노드라 **iOS 빌드를 못 한다.** 그래서 앱 파이프라인은 사내 인프라가 아니라 [[EAS]] 쪽에 둔다. 다만 EAS Workflows의 자동 트리거가 GitHub 전용이라, GitLab에서 `eas build` / `eas update` 를 호출하는 방식을 검토해야 한다.

## 정리
* CI = 자주 합치고 매번 자동 검증. CD = 그 결과를 배포 가능 상태(Delivery) 또는 운영(Deployment)까지
* 브랜치 전략의 "main은 항상 배포 가능"을 기계가 지키게 하는 장치
* 앱은 결과물이 스토어로 가므로 **빌드 → 서명 → 제출 → OTA**가 단계. iOS는 macOS 필수
* 느린 CI·flaky 테스트·비밀값 집중이 3대 함정

#펫바디
