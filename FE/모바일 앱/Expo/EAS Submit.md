# EAS Submit
----

**[[EAS]]의 스토어 제출 서비스.** 서명된 앱 바이너리를 **App Store Connect / Google Play Console에 업로드**하는 과정을 명령 한 줄로 자동화한다.

> 기준: EAS Submit 공식 문서 (2026-10-15 확인)
> 출처: https://docs.expo.dev/submit/introduction/ · https://docs.expo.dev/submit/android/

## 1. 왜 쓰나

스토어 업로드는 원래 플랫폼마다 다른 도구로 손으로 한다.

| 플랫폼 | 원래 방법 | 불편한 점 |
| ----------- | ----------- | ----------- |
| iOS | Xcode Organizer · Transporter 앱 | **macOS에서만** 동작 |
| Android | Play Console 웹에서 AAB 업로드 | 매번 수동, 자동화 불가 |

EAS Submit은 **어느 OS에서든** 같은 명령으로 올리고, [[EAS Build]] 결과물을 바로 집어 가므로 [[지속적 통합(CI)|CI/CD]]에 끼우기 쉽다.

```bash
eas submit --platform ios                       # 최근 빌드 선택
eas submit --platform android --latest --non-interactive   # CI용
eas submit --platform ios --path ./app.ipa      # 로컬에서 만든 파일도 가능
```

## 2. 필요한 것

| 플랫폼 | 바이너리 | 서명 | 계정·인증 |
| ----------- | ----------- | ----------- | ----------- |
| iOS | `.ipa` | 배포 인증서 + 프로비저닝 프로필 | Apple Developer 계정 · App Store Connect (API 키 권장) |
| Android | `.aab` | 업로드 키스토어 | Play Console 계정 + **Google Service Account 키** |

서명이 왜 필요한지는 [[코드 서명(Code Signing)]] 참고.

### 함정: Android 첫 제출
예전에는 Google Play API 제약 때문에 **첫 번째 업로드는 Play Console에서 손으로** 해야 했다. 현재 공식 문서는 *"첫 제출이면 기본 `eas submit` 이 그대로 동작해 internal testing 트랙에 첫 릴리스를 만든다"* 고 한다. 직접 하고 싶으면 수동 업로드도 여전히 가능하다. 오래된 블로그 글을 보면 "첫 업로드는 수동 필수"라고 나오니 헷갈리지 말 것.

## 3. 하지 않는 것

공식 문서가 명시한다 — **스토어 등록정보(설명·스크린샷·릴리스 노트)는 관리하지 않는다.** 바이너리를 올릴 뿐이고, 심사 제출 버튼·스토어 페이지 꾸미기·개인정보 라벨은 여전히 각 콘솔에서 해야 한다.

즉 EAS Submit은 "업로드 자동화"이지 "출시 자동화"가 아니다. 업로드 후에도 심사가 남는다([[앱 출시 프로세스 (iOS · Android)]]) → 그래서 JS만 바뀐 수정은 [[EAS Update]]로 심사 자체를 건너뛴다.

## 4. 장단점

### 장점
* Mac 없이 iOS 업로드
* 빌드 → 제출을 한 도구로 이어 CI에서 자동화
* Service Account 키를 한 번 등록하면 팀원이 Play Console 권한 없이도 제출

### 단점
* 메타데이터·스크린샷은 안 해 준다 (그쪽은 fastlane `deliver`/`supply` 같은 도구가 더 강함)
* 결국 심사 대기는 그대로. 제출이 빨라져도 출시가 빨라지는 건 아니다
* Service Account 키·App Store Connect API 키를 EAS에 맡기는 것 자체가 권한 위임이다. 키 권한 범위를 최소로

## 정리
* EAS Submit = 서명된 바이너리를 스토어에 **올리기만** 하는 자동화
* iOS는 Mac 없이, Android는 Service Account 키로
* 스토어 페이지·심사 제출은 여전히 콘솔에서

#펫바디
