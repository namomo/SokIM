# 속 입력기

<img src="https://github.com/kiding/SokIM/blob/main/SokIM/Assets.xcassets/AppIcon.appiconset/icon_128x128%402x%402x.png" width="128px">
<img src="https://github.com/kiding/SokIM/releases/download/v1.3.1/screenshot.png" width="300px">

빠르고 매끄러운 한영 전환을 위한 새로운 macOS 입력기

---

## 주요 기능 (Key Features)

- **저수준 입력 모니터링 (`IOHIDManager`)**: 가상 키보드 및 하드웨어 키보드의 이벤트를 직접 핸들링하여 반응 속도를 극대화했습니다. 
- **지능적 입력 전략 (`Strategy`)**: 현재 사용 중인 애플리케이션(Xcode, Pages, Chrome 등)을 자동으로 분석하여 가장 적합한 텍스트 처리 방식을 선택합니다. (Direct vs Marked)
- **캡스락(Caps Lock) 한영 전환 지원**: 많은 macOS 사용자가 선호하는 캡스락 기반의 한영 전환 기능을 완벽하게 지원합니다.
- **Karabiner-Elements 호환**: 커스텀 키 매핑 툴과의 충돌을 방지하며 유기적으로 동작합니다.
- **보안 입력 대응**: 암호 입력 필드 등 보안이 필요한 입력 모드를 감지하고 안전하게 대응합니다.

---

## 프로젝트 구조 및 아키텍처 (Architecture)

본 프로젝트는 `InputMethodKit`과 `IOKit`을 결합한 하이브리드 아키텍처를 가집니다.

- **[SokIM](file:///c:/work/github/SokIM/SokIM)**: 모든 소스 코드가 포함된 코어 디렉토리
  - `InputMonitor`: USB HID 레벨의 입력 감지 및 필터링
  - `State`: 입력기의 전체 상태 관리 및 오토마타 제어
  - `Engine`: 한글(TwoSet) 및 영문(Qwerty) 조합 엔진 (오토마타)
  - `Strategy`: 클라이언트 앱의 특성에 맞춘 텍스트 전달 인터페이스
- **[doc](file:///c:/work/github/SokIM/doc)**: 상세 분석 및 문서 보관
  - [상세 아키텍처 분석 문서](doc/architecture.md)

---

## 설치 방법

1. [GitHub Releases](https://github.com/kiding/SokIM/releases)에서 `SokIM.pkg` 다운로드 및 설치 또는 `brew install --cask sokim` 실행 
1. 시스템 설정 → 키보드 → 입력 소스 "편집..." 버튼 → "+" 버튼 → 영어 → "속 입력기" → "추가" 버튼
1. 메뉴 막대에서 현재 입력기를 속 입력기로 변경
1. 시스템 설정 → 개인정보 보호 및 보안 → 입력 모니터링에서 "속 입력기" 권한 허용
1. 시스템 설정 → 개인정보 보호 및 보안 → 손쉬운 사용에서 "속 입력기" 권한 허용

## 삭제 방법

1. 시스템 설정 → 키보드 → 입력 소스 "편집..." 버튼 → "속 입력기" → "-" 버튼
1. 로그아웃 후 재로그인
1. 시스템 설정 → 개인정보 보호 및 보안 → 입력 모니터링 → "속 입력기" → "-" 버튼  
1. 시스템 설정 → 개인정보 보호 및 보안 → 손쉬운 사용 → "속 입력기" → "-" 버튼
1. `/Library/Input Methods/SokIM.app` 삭제 또는 `brew uninstall --cask sokim` 실행

## 디버그 메시지 보기

1. 속 입력기 → 디버그 모드 활성화
1. 터미널에서 `log stream --predicate 'process == "SokIM" AND composedMessage CONTAINS ".swift"' --debug --style compact`
