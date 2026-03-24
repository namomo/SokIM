# 속 입력기 (SokIM) 아키텍처 분석

속 입력기(SokIM)는 macOS 환경에서 빠르고 매끄러운 한영 전환을 제공하기 위해 개발된 입력기(Input Method)입니다. Swift와 InputMethodKit을 기반으로 작성되었습니다.
이 문서는 SokIM 프로젝트의 소스 코드를 분석하여 전반적인 아키텍처와 핵심 컴포넌트의 역할을 정리한 것입니다.

## 1. 개요 (Overview)

SokIM은 OS의 `InputMethodKit` 생태계 내에서 동작하며, 사용자 입력을 가로채어(Intercept) 자체적인 상태 머신(오토마타)을 거친 후 애플리케이션(Client)으로 완성 또는 조합 중인 문자를 전달합니다.
특히 카라비너(Karabiner-Elements) 등과의 충돌 방지 및 안전성 확보를 위해 저수준의 USB HID 콜백(IOHIDManager)을 활용하여 입력 모니터링을 수행하는 점이 특징입니다.

## 2. 주요 아키텍처 흐름

입력 처리는 크게 **[모니터링] -> [이벤트 수신] -> [상태 평가 프로세스] -> [클라이언트 전달]** 의 4단계 흐름을 거칩니다.

1. **InputMonitor**: OS보다 먼저/혹은 병렬로 IOHIDManager를 통해 저수준 키보드 입력(`Input`)을 수집하고 큐(순서)를 정리합니다.
2. **AppDelegate.handle / Controller**: OS 패키지(InputMethodKit) 콜백으로 넘어온 `NSEvent`를 수신합니다.
3. **State & Engine**: `NSEvent`와 앞서 모니터링로 수집한 `Input` 큐를 비교 대조하며 타당성을 검증합니다. 이후 현재 선택된 `Engine`(영문 Qwerty 또는 한글 TwoSet)을 사용해 문자 조합을 갱신합니다.
4. **Strategy**: 사용자가 입력 중인 대상 앱의 특성에 맞춰 `DirectStrategy` 또는 `MarkedStrategy`를 선택하고, 완료된 텍스트를 대상 앱(Target Client)으로 전달합니다.

---

## 3. 핵심 컴포넌트 분석

### 3.1. 애플리케이션 진입점 및 제어 (AppDelegate & Controller)
- **`AppDelegate.swift`**: 
  - 앱의 생명주기를 관리하며, `InputMonitor`, `ClickMonitor`, `HotKeyMonitor` 등 각종 모니터링 객체들을 시작/중지시킵니다.
  - OS의 `InputMethodKit`으로부터 유효한 이벤트가 들어왔을 때, `handle(_:client:)` 메서드에서 실질적인 흐름 제어(Event Handling)를 총괄합니다.
  - ABC(기본 영문) 입력기 제한 기능 및 보안 입력(Secure Input) 시 영문 전환 등의 편의/보안 로직을 포함합니다.
- **`Controller.swift`**: `IMKInputController`를 상속받아 OS에서 넘어오는 이벤트를 수신한 뒤 이를 `AppDelegate`로 위임(Delegate)하는 브릿지 역할을 수행합니다.

### 3.2. 입력 모니터링 (InputMonitor)
- **`InputMonitor.swift`**: 
  - `IOHIDManager`를 이용해 애플리케이션 단이 아닌 **하드웨어 디바이스(USB HID) 레벨**에서 키보드 이벤트를 수집합니다.
  - Caps Lock을 이용한 한/영 전환 인식, 조합 키(Command, Option, Shift) 상태 추적 등의 까다로운 키 맵핑 선행 처리를 수행합니다.
  - 동일한 타임스탬프와 키보드 이벤트를 비교해 오작동을 필터링(flush)합니다.

### 3.3. 상태 관리 및 오토마타 (State & Engine)
- **`State.swift`**: 
  - 현재 입력기의 **상태(Status)**를 보관하는 핵심 모델입니다.
  - 상태값으로 활성화된 엔진(`TwoSet` vs `Qwerty`), Modifier 키 눌림 상태, 조합 완료된 문자(`composed`), 현재 조합 중인 문자(`composing`) 등을 가집니다.
  - 키 입력이 들어오면 `engine`에 묻고 이를 토대로 자신의 `composed` 및 `composing`을 갱신합니다.
  - 백스페이스 입력 처리 단위, 한/영 전환(`rotate()`) 액션을 직접 처리합니다.
- **`Engine.swift` (Protocol)**: 
  - 가상 키코드 기반의 `NSEvent`와 `USB HID Usage` 매핑 정보를 바탕으로 글자를 만드는 인터페이스입니다.
  - **`TwoSetEngine.swift`**: 두벌식 한글 오토마타입니다. 초성/중성/종성의 구조체(`Hangul`)를 기반으로 입력된 자모를 합치거나(combineChars), 백스페이스 입력 시 자모 단위로 분해(backspaceComposing)하는 복잡한 처리를 담당합니다.
  - **`QwertyEngine.swift`**: 영문(QWERTY) 입력 오토마타입니다. Option 단축키를 이용한 특수문자 조합 등을 처리합니다.

### 3.4. 클라이언트 전송 전략 (Strategy)
- **`Strategy.swift` / `DirectStrategy` / `MarkedStrategy`**: 
  - 입력기가 동작 중인 대상 애플리케이션(Client)의 종류에 따라 입력 전달 방식이 다릅니다. (예: Xcode, Pages, Chrome 등)
  - 대상 애플리케이션이 지원하는 텍스트 속성 정보를 분석하여, 문자열을 직접 주입할지(`DirectStrategy`) 아니면 마크된 조합 텍스트 속성을 사용할지(`MarkedStrategy`)를 동적으로 결정합니다.
  - 각 Strategy는 `next(...)`, `backspace(...)`, `commit(...)` 인터페이스를 대상 클라이언트에 맞게 구현합니다.

## 4. 특이사항 및 요약

- **저수준 접근**: SokIM은 macOS의 전통적인 InputMethodKit에만 의존하지 않고 리스크가 있지만 더 빠르고 정확한 응답성을 위해 `HID` 모니터링을 결합했습니다. 방어 로직으로 IOHIDManager 접근 권한이 막혀있을 때의 에러 핸들링과 Karabiner-Elements 가상 키보드 우회 매칭을 수행합니다.
- **정교한 상태 제어**: `Caps Lock`, `Command + Space`, `Shift + Space` 등 한국 사용자가 자주 사용하는 다양한 한영 전환 방식을 네이티브 레벨에서 모방하거나 처리하도록 세밀하게 구현되어 있습니다.
- **유연한 타겟 지원**: 대상 애플리케이션의 특성을 런타임에 파악해 Strategy를 선택함으로써, IDE 및 터미널 환경에서도 입력 조합이 매끄럽게 이루어지는 장점을 가집니다.
