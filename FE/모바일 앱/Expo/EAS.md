# EAS
----

**EAS(Expo Application Services)** 는 [[Expo]]를 만든 회사가 운영하는 **앱 빌드·배포용 클라우드 서비스 모음**이다.

Expo는 앱을 **만드는** 오픈소스 [[프레임워크]](무료)이고, EAS는 만든 앱을 **빌드하고 내보내는** 유료(무료 구간 있음) 서비스다. 둘은 이름이 붙어 다닐 뿐 별개다. Next.js와 Vercel의 관계와 같다 — Next.js 앱을 Vercel 없이 어디든 배포할 수 있듯, Expo 앱도 EAS 없이 개발·빌드할 수 있다(`npx expo run:ios` 로 로컬 빌드).

> 기준: EAS 공식 문서 · expo.dev/pricing (2026-10-15 확인)
> 출처: https://docs.expo.dev/eas/ · https://expo.dev/pricing

## 1. 왜 따로 서비스가 필요한가

웹은 빌드 결과물이 정적 파일이라 서버에 올리면 끝이다. 앱은 다르다.

* iOS 빌드는 **macOS + Xcode** 가 있어야만 된다
* 결과물(IPA/APK)에 **[[코드 서명(Code Signing)]]** 을 해야 기기에 설치된다. 인증서·프로필·키스토어를 팀이 나눠 관리하기가 번거롭다
* 배포 경로가 서버가 아니라 **스토어 심사**다

EAS는 이 세 가지(빌드 환경, 서명, 배포 경로)를 클라우드로 대신해 준다.

## 2. 구성

| 서비스 | 하는 일 | 대신하는 것 |
| ----------- | ----------- | ----------- |
| [[EAS Build]] | 클라우드 macOS/Linux에서 iOS·Android 빌드 + 서명 | 로컬 Mac·Xcode·Android Studio |
| [[EAS Submit]] | 빌드 결과물을 App Store / Play Store에 업로드 | Transporter, Play Console 수동 업로드 |
| [[EAS Update]] | JS·에셋 변경을 심사 없이 설치된 앱에 전달 ([[OTA 업데이트]]) | 스토어 재심사 |
| EAS Workflows | 위 셋을 엮는 앱 전용 CI/CD → [[지속적 통합(CI)]] | GitHub Actions 등 범용 CI |
| EAS Hosting | Expo Router 웹 출력 배포 | Vercel 등 |

셋을 따로 골라 쓸 수 있다. 예를 들어 빌드는 로컬에서 하고 [[EAS Update]]만 쓰는 것도 된다.

## 3. 다루는 방법

| 도구 | 역할 |
| ----------- | ----------- |
| `eas-cli` | `eas init` · `eas build` · `eas submit` · `eas update` · `eas credentials` |
| `eas.json` | **빌드 프로필** 설정 (`development` / `preview` / `production` 등). 프로필마다 배포 방식·채널·환경변수를 다르게 |
| expo.dev 대시보드 | 빌드 기록·로그·설치 링크(QR)·업데이트 이력·자격 증명 |
| `extra.eas.projectId` | 앱 설정 안에 박힌 UUID. **코드와 대시보드 프로젝트를 잇는 유일한 연결고리** |

```ts
// app.config.ts
export default {
  name: 'petbody',
  slug: 'petvatar',
  extra: {
    eas: { projectId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' },
  },
};
```

### 함정: 동적 설정이면 `eas init` 이 projectId를 못 쓴다
`app.json`(정적)이면 `eas init` 이 자동으로 `extra.eas.projectId` 를 써 넣는다. `app.config.ts`/`app.config.js`(동적)는 코드라서 CLI가 고칠 수 없다 → **"can't automatically be added"** 경고가 나오고 직접 붙여 넣어야 한다. 출처: [Expo Forums](https://forums.expo.dev/t/eas-project-id-with-dynamic-configuration/66158) (공식 문서 본문에서는 이 문구를 직접 찾지 못함 — 경고 문구는 포럼 기준)

### slug는 사용자에게 보이지 않는다
`slug` 는 *"계정 안에서 유일한 URL용 이름"* 이다(공식 app config 문서). 홈 화면 앱 이름(`name`), 딥링크 `scheme`, 번들 ID(`ios.bundleIdentifier` / `android.package`)와 **무관**하다. 그래서 대시보드 프로젝트 이름에 맞춰 slug를 바꿔도 사용자 쪽에서는 아무것도 바뀌지 않는다.

## 4. 개인 계정 vs Organization

| | 개인(Personal) | Organization |
| ----------- | ----------- | ----------- |
| 용도 | 개인 프로젝트 | 회사·팀 |
| 멤버 초대·권한 | 불가 | 역할별 부여 |
| 결제 주체 | 개인 | 조직 |

* 프로젝트는 대시보드 **Project settings > General > Transfer project** 로 옮긴다
* 옮겨도 **projectId는 그대로**다. 앱 설정에 `owner` 필드만 새 계정으로 맞추면 된다 (`owner` 를 안 쓰면 로그인한 사용자 계정이 기본값)
* 출처: https://docs.expo.dev/accounts/account-types/

**언제 옮기나**: 회사 Apple Developer 계정을 연결하기 전, 팀원이 합류하기 전, 운영 빌드를 내기 전, 유료 결제를 하기 전. 개인 계정에 회사 자격 증명·결제가 묶이면 퇴사·인수인계 때 문제가 된다.

## 5. 요금 (무료 플랜 한도)

| 항목 | Free | Starter | Production |
| ----------- | ----------- | ----------- | ----------- |
| 월 요금 | $0 | $19 + 사용량 | $199 + 사용량 |
| 빌드 | 월 Android 15 + iOS 15 | $45 빌드 크레딧 | $225 빌드 크레딧 |
| 빌드 대기열 | 저우선 | 고우선 | 고우선 |
| 빌드 타임아웃 | 45분 | | |
| Update 대상 | 1K MAU | 3K MAU | 50K MAU |

> expo.dev/pricing 기준 (2026-10-15 확인). 요금은 자주 바뀌므로 결제 전 다시 확인

## 6. 장단점

### 장점
* Mac이 없어도(Windows 팀원도) iOS 빌드가 된다
* 서명 자격 증명을 EAS가 생성·보관 → 팀원이 Apple 계정 권한 없이도 빌드 가능
* 빌드 → 제출 → OTA가 한 도구·한 대시보드에 모인다

### 단점
* **클라우드 빌드는 소스가 Expo 서버로 올라간다.** 보안 정책상 안 되면 `eas build --local` 로 같은 빌드 과정을 내 기기에서 돌린다 (캐시·비밀 환경변수 미지원 등 제약 있음)
* 무료 플랜은 빌드 횟수·업데이트 MAU 한도가 있고, 저우선 대기열이라 바쁠 때 오래 기다린다
* 서명 키를 EAS에 맡기면 **벤더 종속**이 생긴다. Android 키스토어는 `eas credentials` 로 받아서 따로 백업해 둘 것
* `extra.eas.projectId` 한 줄을 지우면 연결이 끊긴다. 반대로 말하면 이 줄이 실수로 다른 프로젝트 ID로 바뀌면 엉뚱한 프로젝트로 빌드·업데이트가 간다

## 7. 펫바디 적용 사례

* 2026-10-15 `petbody-client-app` 을 대시보드 프로젝트 `@seraaa/petvatar` 에 연결
* `app.config.ts`(동적 설정)라 `eas init` 이 projectId를 못 써서 **수동 추가**
* slug는 대시보드에 맞춰 `petvatar`. 홈 화면 이름 `petbody`·scheme·번들 ID와는 무관하므로 영향 없음
* 현재 **개인 계정 소속** → 회사 Apple 계정 연결·팀원 합류·운영 빌드·유료 결제 전에 Organization으로 이전 예정 (projectId·slug 유지, `owner` 만 추가)

## 정리
* Expo = 앱을 만드는 프레임워크, EAS = 빌드·서명·제출·OTA를 대신하는 클라우드. 별개다
* 코드와 대시보드를 잇는 건 `extra.eas.projectId` 한 줄. 동적 설정이면 직접 써 넣는다
* slug는 내부 식별자. 앱 이름·번들 ID와 헷갈리지 말 것
* 회사 일이면 처음부터 Organization 계정으로. 늦어도 운영 빌드 전에 옮긴다

#펫바디
