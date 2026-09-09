# InputPlumber Zotac Gaming Zone 버그 수정 - 작업 인계 문서

## 상태 요약 (최신, 2026-09-09)

**닫힌 PR 664/668 리뷰 쟁점을 코멘트 원문으로 재대조 완료 — 남은 진짜 이견은 HOME 버튼 두 개뿐이고,
그것도 "OGUI intercept mode를 전제하느냐"의 차이다.** 이 과정에서 **우리 쪽 서술 2건이 틀렸음이
드러나 정정**했다(DBus 타겟은 실제로 붙어 있다; `QuickAccess2`/`Keyboard`가 안 먹는 이유는
`InterceptMode = 0`이다). 상세는 §9-11.

**리부트 검증 완료 (TODO 1) — 리부트 후에도 컴포짓 1개, 소스 8개 전부 정상, 게임모드에서 사용자
확인 "문제 없음".** 노드 번호는 09-08 대비 전부 바뀌었지만(이름/`phys_path` 매칭이라 무관) 구성은
동일. 09-08의 중복 컴포짓/xpad 누락 문제는 재발하지 않았다. 상세는 "★ 2026-09-09 세션" §8 참고.

**다이얼 검증 완료 (TODO 2) — 다이얼은 볼륨(왼쪽)/화면 밝기(오른쪽)로 동작하고, `zone1`의
`Left/RightStickDial` 매핑은 런타임에서 전부 버려지는 no-op이다.** 원인은 타겟 라우팅이 capability
기준 필터링이고(`composite_device/targets.rs:303-312`) `xbox-elite`/`mouse`/`keyboard` 어느
타겟도 `Gamepad:Dial:*`을 선언하지 않기 때문. 하드웨어·벤더 드라이버·config는 전부 정상.
upstream 패키지 config도 같은 죽은 매핑을 갖고 있다. 상세는 "★ 2026-09-09 세션" §1-5 참고.

**upstream 이슈 후보 6건 검증·정리 완료(§12)** — 그중 하나("매핑된 capability가 `device N test`에
안 뜬다")는 upstream이 이미 `#690`으로 등록해둔 알려진 v2 맵 버그였다. 남은 건 실제 제출뿐.

**이 밖에 09-09 세션에서 한 것**: 벤더 게임패드 entry의 `phys_path: "*/input1"` 제거(TODO 5,
이름만으로 매칭 — 검증 완료), 패들 스크립트 `~/zotac-zone-tools/obsolete/`로 퇴역(TODO 3),
로컬 `main`을 `upstream/main`(0.79.2)에 동기화(TODO 6), config 주석을 PR 리뷰 기준으로 축약,
그리고 이 문서의 과거 기록 분리.

## 환경

- 기기: Zotac Gaming Zone (board_name: G0A1W / G1A1W, sys_vendor: ZOTAC), USB 0x1ee9:0x1590, fw 1.3.9
- OS: Bazzite (bootc/rpm-ostree, `ghcr.io/ublue-os/bazzite-deck:stable`).
  2026-09-08 시점 `44.20260907`, 커널 `7.2.3-ogc3.1.fc44`, 벤더 드라이버 `zotac_zone_hid` 포함
- 설치 InputPlumber: `0.79.0-4` / 체크아웃 `claude` 브랜치: `0.78.1` / 로컬 `main`: `0.79.2`
  (⚠️ 설치판 동작을 코드로 설명할 땐 `git show main:<path>` 로 읽을 것)
- 개발 환경: toolbox (`inputplumber-dev`) — 상세는 CLAUDE.md "Development environment on this machine"
- 저장소: origin = whirlwind80/InputPlumber (fork), upstream = ShadowBlip/InputPlumber
- 로컬 경로: `~/InputPlumber` (= `/var/home/whirlwind80/InputPlumber`)
- 배포된 override: `/etc/inputplumber/devices.d/50-zotac-zone.yaml`,
  `/etc/inputplumber/capability_maps.d/zone_type1.yaml` (+ 미참조 잔재 `zone_type1_dial.yaml`).
  리포의 `rootfs/usr/share/inputplumber/...` 사본과 항상 동기화할 것

## 과거 기록은 `handoff-archive.md`

2026-08-25 ~ 09-04 세션 기록(= `hid-generic` 시절 하드웨어 서술 + 닫힌 PR 664/668 리뷰 과정)은
전부 **`handoff-archive.md`**로 옮겼다. 아래 내용은 벤더 드라이버가 들어온 뒤(09-08~)의 현재 상태만
다룬다. 리뷰 쟁점이 **어떻게 결론났는지**는 2026-09-09 세션 §9-11 대조표에 정리돼 있으므로,
대부분의 경우 아카이브를 열 필요가 없다.

---

## ★ 2026-09-08 세션 — 벤더 드라이버 도착, 실기기 재검증 및 컨트롤러 복구

### 0. 발단

사용자 신고: "Bazzite 업데이트 후 컨트롤러가 정상 동작하지 않는다. guide, QAM, screenshot, paddle
버튼이 안 된다."

### 1. 1차 원인 — 서비스가 masked 상태로 방치돼 있었음

`inputplumber.service`가 09-07 21:35부터 23시간째 `masked` + `inactive`였다. 업데이트 재부팅 직후
정상 기동했다가 2초 만에 누군가 `systemctl mask` + `stop`을 실행한 흔적(로그상 SIGTERM). 이전 세션의
evtest 디버깅 절차(mask 후 복구)를 되돌리지 않은 것으로 보인다. `unmask` + `start`로 복구.
**교훈: CLAUDE.md에 이미 있는 "작업 끝나면 반드시 unmask" 경고를 실제로 지킬 것.**

### 2. ★★ 진짜 사건 — 벤더 드라이버가 들어와 있었다

복구 후 로그를 보다 `driver: Some("zotac_zone_hid")`를 발견. `lsmod`/`modinfo`/`/sys/bus/hid/devices`
전부 확인 결과 3개 HID 인터페이스 전부 벤더 드라이버 바인딩. 커널 sysfs에 `btn_m1/remap`,
`btn_m2/remap`, `btn_a/remap`, `dpad_*/remap` 등도 노출됨. 즉 CLAUDE.md가 "언젠가 오면 재검증
세션이 필요하다"고 예고한 바로 그 시점이었다. 동시에 InputPlumber 0.79.0 패키지가 **자체 네이티브
Zotac Zone config**(패키지판 `50-zotac-zone.yaml` + `zone_type1.yaml`)를 갖고 들어왔다.

### 3. 버그 3개를 순차적으로 찾아 고침

**(a) 컴포짓 디바이스가 2개로 쪼개짐** — Steam에 "Xbox Elite 2"가 2개 잡히고 버튼이 엉킴.
원인은 두 겹이었다:
- `SourceDevice.unique`의 기본값이 `true`라, 이미 매칭된 entry에 다른 물리 장치가 또 매칭되면
  기존 컴포짓에 합치지 않고 **새 컴포짓을 만든다**. → 전 entry에 `unique: false` 추가.
- 더 결정적으로, **`/etc` override와 `/usr/share` 패키지 config가 둘 다 로드된다**(파일명 dedup
  없음, `config/path.rs` + `manager.rs`의 `load_device_configs()` 확인). 우리 override엔 Dials
  entry가 주석 처리돼 있었는데 패키지판엔 살아 있어서, Dials 장치가 override에서 매칭 실패 →
  패키지 config로 폴백 → **패키지 config 기준의 두 번째 컴포짓 디바이스가 생성**됐다.
  → override에 Dials entry를 추가해 패키지판을 완전히 커버하는 superset으로 만들자 재시작을
  반복해도 항상 컴포짓 1개로 안정됨.

**(b) F16-F19 배치가 통째로 바뀜** — `evtest`로 실측(InputPlumber 정지 후 `event3` 캡처):

| 버튼 | 벤더 드라이버 하 | hid-generic 시절 |
|---|---|---|
| ZOTAC | `KEY_F16` | `KEY_F17` |
| MORE/QAM | `KEY_F17` | `KEY_F18` |
| HOME 짧게 | `KEY_F18` | `Meta+D` 코드 |
| HOME 길게 | `KEY_F19` | `Ctrl+Alt+KP.` 코드 |
| 패들 M2/M1 | `KEY_HOME`/`KEY_END` | 동일 |

즉 **HOME 버튼이 더 이상 조합키를 안 보내고 깔끔한 F18/F19를 보낸다.** capability_map의
chord entry 2개는 죽은 코드가 되어 제거하고, F17→QuickAccess / F18→Screenshot / F19→Guide로 교체.
(`QuickAccess2`/`Keyboard`는 여전히 xbox-elite에서 evdev 출력이 없으므로 upstream 패키지판 그대로는
안 씀.)

**★ 이로써 PR #664 리뷰 논쟁이 사후적으로 정리됐다**: pastaq의 매핑 컨벤션 주장은 *벤더 드라이버가
있는 하드웨어에서는 옳았고*, 이쪽 실측 반박은 *hid-generic 하드웨어에서 옳았다*. 서로 다른 커널을
보고 있었던 것. 어느 쪽도 틀리지 않았다.

**(c) ★ 가장 오래 걸린 함정 — 벤더 드라이버의 게임패드 노드는 아무것도 안 보낸다**

`ZOTAC Gaming Zone Gamepad`(`event10`/`js0`)는 이름도 딱 맞고 `BTN_SOUTH`~`BTN_THUMBR`,
`ABS_X/Y/Z/RX/RY/RZ`, `ABS_HAT0X/Y`, `BTN_TRIGGER_HAPPY1-6`까지 **완전한 게임패드 capability를
선언**하며 FF 이펙트 업로드도 성공한다. 그런데 **ABXY를 눌러도 이벤트가 하나도 안 나온다**
(`evtest`로 확정). 실제 게임패드 입력은 예전 그대로 커널 `xpad`의 `event6`/`js1`(USB 인터페이스 0,
`phys_path */input0`)로만 흐른다.

세션 초반에 내가 이 `*/input0` entry를 "벤더 드라이버가 게임패드를 제공하니 중복"이라고 판단해
제거한 것이 표준 버튼(ABXY/스틱/트리거/D패드/숄더)이 죽은 직접 원인이었다. 증상이
"Xbox Elite 2에선 특수 버튼만 되고, 옆에 raw로 뜬 ZOTAC 컨트롤러에선 기본 버튼이 된다"로 나타나서
설정 오류로 안 보이고 마치 타겟 출력이 깨진 것처럼 보였다. entry 복원 후 정상화.

### 4. 최종 상태 (사용자 확인: "이제 정상 동작하는 것 같습니다")

컴포짓 디바이스 1개, 소스 = `hidraw2` + `event6`(xpad, 진짜 게임패드) + `event7`(Dials) +
`event10`(벤더 게임패드, FF 전용) + `event3`(Keyboard) + `iio:device0`. 다이얼도 이번에 처음으로
제대로 물렸다(`zone1`의 `REL_HWHEEL/REL_WHEEL → LeftStickDial/RightStickDial`).

패들 hidraw 워크아라운드(`~/zotac-zone-tools/zotac-zone-paddles`)는 **이제 불필요** — 펌웨어가
알아서 `KEY_HOME`/`KEY_END`를 보내고, 커널 sysfs remap 경로도 열려 있다. Steam 비-Steam 게임
등록도 정리해도 된다.

### 5. 삽질하면서 알게 된 것 (다음 세션 필독)

- **`inputplumber device N test`의 Buttons 패널은 소스가 "선언한" capability만 보여준다.**
  `Screenshot`/`QuickAccess`/`LeftPaddle1`처럼 capability_map 번역으로만 생기는 건 동작해도 박스가
  안 뜬다. 반대로 `RightPaddle1/2` 박스는 `event10`이 `BTN_TRIGGER_HAPPY5/6`을 선언해서 뜨지만
  실제로 켜지는 건 키보드 경로다. **패널에 없다/반응 없다를 "매핑이 깨졌다"로 읽지 말 것.**
- `filtered_events:`는 모든 capability_map YAML에 있지만 **`CapabilityMapConfigV2`에 없는 필드다.**
  serde가 그냥 무시한다. 필터링은 source_devices entry의 `events: {include/exclude}`로 해야 하고,
  이건 raw evdev 코드가 아니라 **번역된 Capability 문자열**(`"Gamepad:Button:RightPaddle1"`)로
  매칭한다.
- InputPlumber 코어 버그 후보: `target/mod.rs` ~L551에서 타겟의 `write_event`/`emit()`이 한 번
  실패하면 **그 타겟의 `run()` 루프가 영구히 죽는다**(로그는 `debug` 한 줄, 재생성 없음).
- `flatpak-spawn --host sudo`의 `ksshaskpass: Unable to parse phrase` 경고는 **무해하다**(명령은
  실행됨). 단 `sudo bash -c '...'`로 감싸면 진짜로 깨진다.
- 이번 세션에서 백그라운드 캡처(journalctl -f, timeout evtest)로 사용자 버튼 입력과 타이밍을
  맞추려던 시도는 계속 실패했다. **사용자 본인 터미널에서 포그라운드로 돌리게 하고 결과를 붙여받는
  방식이 유일하게 잘 됐다.**

### 5.5. pastaq 주장과의 최종 대조 (세션 말미에 확인)

| pastaq 주장 | 현재 config | 부합 |
|---|---|---|
| F16 → `Guide` | 동일 | ✅ |
| MORE(F17) → `QuickAccess` | 동일 (이번에 `Guide`에서 고침) | ✅ |
| HOME 짧게 → `QuickAccess2` | `Screenshot` | ❌ 의도적 |
| HOME 길게 → `Keyboard` | `Guide` | ❌ 의도적 |
| `unique: false` 추가하라 | 전 entry에 추가 | ✅ |
| 터치패드 entry 제거 | 유지(제거 상태) | ✅ |
| 벤더 드라이버가 정답 | 그 전제로 재작성 | ✅ |
| 드라이버 있으면 게임패드가 `*/input3` | 여기선 `*/input1`+`*/input0` | ⚠️ 재현 안 됨 |

**의도적으로 다른 2개의 근거는 0.79.0에서도 그대로 유효함을 확인함**(`git show v0.79.0:src/input/
event/evdev.rs` → `GamepadButton::Keyboard => vec![]`, `QuickAccess2 => vec![]`). `QuickAccess`와
`Screenshot`만 `xpad.rs::write_event`가 특수 처리한다. pastaq의 컨벤션은 **DBus/`unified_gamepad`
타겟이 런타임에 붙는 스택**(OpenGamepadUI가 타겟을 교체하는 경우)을 전제로 한 것으로 보이는데, 이
기기 config의 `target_devices`엔 DBus 타겟이 없다. **주목할 점: upstream이 이번에 패키지로 낸
`50-zotac-zone.yaml`도 DBus 타겟 없이 `QuickAccess2`/`Keyboard`를 쓰고 있어서, 그대로 쓰면 HOME
버튼 두 개가 아무 동작도 안 한다.**

또 하나의 차이: **upstream 패키지판에는 `*/input0`(xpad) entry가 없다.** 오늘 실측으로 그게 없으면
표준 버튼이 전부 죽는 걸 확인했으므로 이 부분은 우리 쪽이 이 하드웨어에 대해 더 정확하다.

### 6. 다음 세션 TODO

1. 재부팅 후에도 컴포짓 디바이스 1개 + 전 버튼 정상인지 한 번 더 확인(이번엔 재시작만 반복 검증함).
2. 다이얼 실제 동작 확인 — config는 물렸지만 실기기에서 좌/우 다이얼을 돌려본 적은 아직 없음.
3. `~/zotac-zone-tools/zotac-zone-paddles` 및 Steam 비-Steam 게임 등록 정리(이제 불필요).
4. upstream 이슈 후보 4건 정리해서 올릴지 결정: 타겟 `run()` 루프 영구 사망, `unique` 기본값 함정,
   `filtered_events` 무시, 동명 config 중복 로드. (PR은 안 내더라도 이슈는 가치 있음)
5. **벤더 게임패드 entry의 `phys_path: "*/input1"` 제약을 뺄지 검토** — upstream 패키지판은
   `phys_path` 없이 이름만으로 매칭한다. 원래 이 제약을 넣은 이유는 중복 컴포짓 디바이스 방지였는데
   이제 `unique: false`가 들어가서 그 역할이 대부분 사라졌고, 빼면 pastaq가 자기 기기에서 보고한
   `*/input3` 토폴로지(여기선 재현 안 됨)에도 자동으로 대응된다. **다만 지금 정상 동작 중이므로
   건드리면 재검증 필요** — 우선순위 낮음, 다른 기기/커널로 옮길 계획이 생기면 그때 하는 게 맞다.
6. **체크아웃(0.78.1)과 설치 패키지(0.79.0-4)의 버전이 어긋나 있다.** 09-08 세션에서 "QuickAccess2/
   Keyboard는 출력이 없다" 같은 코드 근거를 0.78.1 기준으로 읽고 있었다는 걸 뒤늦게 발견해서
   `v0.79.0` 태그로 재확인했고 결론은 같았지만, 앞으로도 이러면 위험하다. `upstream/main`(현재
   `0ca9869`, 0.79.1)을 받아서 로컬 `main`을 동기화해둘 것. **단 `claude` 브랜치에 머지하지 말 것**
   (문서 파일이 upstream으로 새는 걸 막는 이 리포의 규칙).
7. 미착수로 계속 남아있는 것: PR 코드 라인별 설명 듣기.

---

## ★ 2026-09-09 세션 — 다이얼 실동작 검증 (TODO 2), 결론: capability 매핑은 no-op

### 0. 결과 요약

**다이얼은 "볼륨(왼쪽)/화면 밝기(오른쪽)"로만 동작한다. `zone1`의 `Left/RightStickDial` 매핑은
문법상 맞지만 런타임에서 완전히 버려진다 — 하드웨어·드라이버·설정이 아니라 타겟 쪽이 막힌 것.**
코드 변경 없음. 리부트 검증(TODO 1)은 이 세션에서 미착수.

### 1. 두 경로 중 하나만 살아 있다

| 경로 | 결과 |
|---|---|
| rid=3 → 벤더 드라이버 → `Dials`(event7)의 `REL_HWHEEL`/`REL_WHEEL` → `zone1` → `Left/RightStickDial` | ❌ 버려짐 |
| rid=3 → HID 코어 → `Keyboard`(event3)의 `KEY_VOLUMEUP`/`KEY_BRIGHTNESSUP` → 가상 키보드(event16) 통과 | ✅ 볼륨/밝기 |

`sudo libinput debug-events`로 실측: 왼쪽 다이얼 3칸 → `event16 KEY_VOLUMEUP` 3회, 오른쪽 3칸 →
`event16 KEY_BRIGHTNESSUP` 3회. **`InputPlumber Mouse`(event22)에서는 스크롤이 단 하나도 안 나옴.**

### 2. ★ 버려지는 지점 (코드 + 런타임 양쪽 확정)

- `src/input/composite_device/targets.rs:303-312` — `TargetDeviceSet::write_event()`는
  `target_devices_by_capability`에서 해당 capability를 조회하고, **없으면 `trace` 한 줄 남기고
  `return`한다.** 타겟 라우팅은 capability 기준 필터링이다.
- 런타임 `TargetCapabilities`(227개)에 **`Gamepad:Dial:*`이 하나도 없다.**
  - `xbox-elite`: `create_virtual_device()`가 `with_keys` + abs 축만 부르고 **relative 축을 아예
    선언하지 않는다**(`src/input/target/xpad.rs:148-156`). `get_capabilities()` 목록(L181-224)에도
    `Gamepad::Dial` 없음.
  - `mouse`/`keyboard` 타겟: `Mouse::*` / `Keyboard::*`만 선언.
- 그래서 다이얼 이벤트는 번역 직후 소멸한다. 게다가 `capability_map_id: zone1`이 붙어 있어서 raw
  `Mouse:Wheel`로도 안 나간다(= 가상 마우스 스크롤도 없음). 실측과 정확히 일치.

**주의: `mouse` 타겟의 `translate_event()`는 제네릭이라(`EvdevEvent::from_native_event`)
`Gamepad::Dial` → `REL_HWHEEL`/`REL_WHEEL`로 번역할 능력이 있고 가상 마우스도 그 축을 선언한다
(`mouse.rs:107-111`). 즉 "번역기는 가능한데 라우터가 이벤트를 안 넘겨준다"는 구조 —
`translate_event`만 읽고 "될 것"이라 판단하면 틀린다. 반드시 `targets.rs`의 라우팅을 볼 것.**

### 3. 하드웨어/드라이버는 완전 정상 (확인 근거)

- HID debugfs(`/sys/kernel/debug/hid/0003:1EE9:1590.0001/events`, grab 무관·비침습)로 rid=3 실측:
  왼쪽 CW 3칸 = `03 00 00 08` ×3, 오른쪽 = `03 00 00 01`/`02`/`01`. CLAUDE.md의 다이얼 비트맵
  (`0x01` R-CW / `0x02` R-CCW / `0x08` L-CW / `0x10` L-CCW)과 정확히 일치.
- 벤더 드라이버 소스(`zotac-zone-hid-core.c:91-101`)가 rid=3을 `wheel_input`에 `REL_WHEEL`/
  `REL_HWHEEL`로 정상 출력하고, `Dials` 노드는 `EV=5`, `REL=140`(= `REL_HWHEEL`+`REL_WHEEL`)만
  선언한다. 09-08의 게임패드 노드 같은 "선언만 하고 안 보내는 껍데기"가 **아니다.**
- `/etc`의 `zone_type1.yaml` L70-89 다이얼 규칙도 정상.

### 4. upstream도 같은 문제를 갖고 있다 (이슈 후보 추가)

패키지판 `/usr/share/inputplumber/devices/50-zotac-zone.yaml`의 `target_devices`도
`xbox-elite`/`mouse`/`keyboard`이고, 패키지판 `zone_type1.yaml`에도 동일한
`REL_HWHEEL→LeftStickDial` / `REL_WHEEL→RightStickDial` 규칙이 있다. **즉 상류 설정에서도 이
다이얼 매핑은 죽은 코드다.** TODO 4의 upstream 이슈 후보 목록에 5번째 항목으로 추가할 것:
"capability_map이 어떤 타겟도 선언하지 않은 capability로 번역하면 `trace` 로그 한 줄만 남기고
조용히 버려진다 — 설정 로드 시점에 경고할 수 있는 종류의 오류다."

### 5. 판단 — 현 상태 유지 권고

- 볼륨/밝기는 핸드헬드에서 유용한 기본 동작이고, 지금 실제로 잘 된다.
- 다이얼을 게임패드로 보내려면 `xbox-elite` 타겟에 relative 축을 추가하는 upstream 변경이 필요한데,
  실제 Xbox 컨트롤러에 없는 축이라 받아들여지기 어렵다.
- **`Dials` 소스 entry 자체는 반드시 유지할 것** — 빼면 패키지 config의 superset이 깨져서 중복
  컴포짓 디바이스가 다시 생긴다(09-08 세션 참고).

### 6. 헛수고한 것 (다음에 반복하지 말 것)

**`sudo busctl --system monitor org.shadowblip.InputPlumber`로 번역된 이벤트를 보려는 시도는
안 된다.** 캡처 2839줄에 shadowblip 트래픽이 **0건**이었다(대조군으로 누른 ZOTAC 버튼조차 안 잡힘).
컴포짓 디바이스에 `dbus0` 타겟이 `DbusDevices`로 붙어 있긴 하지만 실제로 시그널을 내보내지 않는다
(`InterceptMode`가 0이라 일반 이벤트는 DBus로 안 감 — `composite_device/mod.rs:1080-1105`).
번역 결과를 보고 싶으면 `LOG_LEVEL=trace`로 데몬을 띄우거나, 가상 타겟 노드를 직접 `evtest`할 것.

### 7. 이 세션의 baseline (리부트 전, 정상 상태)

컴포짓 1개(`CompositeDevice0` "Zotac Zone"), 소스 8: `hidraw2`, `event7`(Dials), `event10`(벤더
게임패드), `event3`(Keyboard), `iio:device0`, LED×2, `event6`(xpad).
가상 장치: `Microsoft X-Box One Elite 2 pad` = event21/js2, `InputPlumber Mouse` = event22,
`InputPlumber Keyboard` = event16. (노드 번호는 리부트마다 바뀜)

### 8. 리부트 검증 (TODO 1) — 완료, 정상

리부트 직후 실측:

| 항목 | 결과 |
|---|---|
| 컴포짓 디바이스 | 1개 (`CompositeDevice0`) |
| 소스 8 | `hidraw2`, `event13`(xpad, `*/input0`), `event8`(Keyboard), `event9`(Dials), `event11`(벤더 게임패드), `iio:device0`, LED×2 |
| 터치패드(`event10`) | 의도대로 미grab |
| 가상 컨트롤러 | `Microsoft X-Box One Elite 2 pad`(event21/js2) 1개 |
| 타겟 | gamepad0 + keyboard0 + mouse0 |

사용자가 게임모드에서 확인 후 "문제 없어 보입니다". **09-08 세션에서 겪은 중복 컴포짓 디바이스,
xpad 소스 누락, raw 컨트롤러 노출은 재부팅 후에도 재발하지 않았다.**

노드 번호는 09-08 대비 전부 이동했다(`event6→13`, `event3→8`, `event7→9`, `event10→11`,
가상 마우스 `22→20`, 가상 키보드 `16→19`). 이름/`phys_path` 기반 매칭이라 영향 없음이 실증됐다.
**핸드오프 문서에 노드 번호를 적을 때는 항상 "이 시점 값"임을 전제할 것.**

`/etc` override 3개(`devices.d/50-zotac-zone.yaml`, `capability_maps.d/zone_type1.yaml`,
`capability_maps.d/zone_type1_dial.yaml`) 전부 리부트 후에도 존재. 단
**`zone_type1_dial.yaml`(id `zone1_dial`)은 어느 `source_devices` entry도 참조하지 않는 잔재**다
— 다이얼 규칙은 `zone_type1.yaml` 안으로 들어갔고, §2에 따라 그 매핑 자체가 no-op이다. 무해하지만
정리 후보.

게임모드에서 `Microsoft X-Box 360 pad 0`(event25/js3)이 함께 보이는데 이건 **Steam Input이 만드는
자체 에뮬레이션 패드**이지 중복 InputPlumber 컨트롤러가 아니다. 09-08의 "Xbox Elite 2가 2개" 증상과
혼동하지 말 것.

### 9. ★ 닫힌 PR 664/668 리뷰 쟁점 최종 대조 (GitHub 코멘트 원문 재조회)

`gh api repos/ShadowBlip/InputPlumber/issues/{664,668}/comments`로 코멘트 전문(664: 7건,
668: 4건)을 다시 받아 현재 런타임과 대조했다. 코드 변경 없음(문서만).

| pastaq의 주장 (날짜) | 현재 상태 | 판정 |
|---|---|---|
| ZOTAC → `Guide` | F16 → `Guide` | ✅ 채택 |
| MORE → `QuickAccess` (08-26, 08-28 재주장) | F17 → `QuickAccess` (09-08에 `Guide`에서 고침) | ✅ 채택 |
| HOME 짧게 → `QuickAccess2` | F18 → `Screenshot` | ❌ 의도적 |
| HOME 길게 → `Keyboard` | F19 → `Guide` | ❌ 의도적 |
| "드라이버 유/무 매핑이 1:1로 같아야" (08-28) | 벤더 드라이버가 버튼당 F-키 하나씩 보내 조합키 소멸 → chord entry 2개 삭제 | ✅ 자연 해소 |
| 터치패드 entry 제거 (08-27) | 제거 유지 | ✅ 채택 |
| `unique: false` | 전 entry 적용 | ✅ 채택 |
| "다이얼은 드라이버 정리할 때 다시 보겠다" (08-27) | 드라이버 도착·전용 노드 생김, **매핑은 여전히 무동작**(§2) | ⚠️ 미해결 |
| "capability map 항목은 전부 composite capabilities에 들어간다" (08-28) | `Capabilities`에 `Guide`만 존재, `Screenshot`/`QuickAccess`/`Paddle` 없음 | ❌ 우리 관측이 맞음 |
| "드라이버 없으면 게임패드가 enumerate 안 된다" (09-02, 664 종료 사유) | `hid-generic` 시절에도 xpad 노드 존재했고 지금도 진짜 입력은 xpad | ⚠️ 재현 안 됨 |
| 드라이버 있으면 게임패드가 `*/input3` | 여기선 `*/input0`(xpad) + `*/input1`(벤더) | ⚠️ 재현 안 됨 |
| PR 668 대신 드라이버를 OGC에 싣겠다 (09-03) | 드라이버가 패들 매핑 직접 적용, `KEY_HOME`/`KEY_END` 네이티브 도착 | ✅ **그의 판단이 옳았음** |

**`Capabilities` 항목의 해소**: pastaq가 인용한 `composite_device/mod.rs` L233-256은 **device-wide
`capability_map_id`** 경로(`load_capability_map` → `translatable_capabilities`)다. 이 config는 그
최상위 키를 쓰지 않고 **per-source `capability_map_id`**만 쓰므로 그 코드가 아예 안 탄다. 양쪽 다
맞는 말이었고 서로 다른 경로를 보고 있었다.

### 10. ⚠️ 이번 대조에서 드러난 우리 쪽 오류 2건 (09-08 §5.5 서술 정정)

**(a) "이 기기 config엔 DBus 타겟이 없다"는 틀렸다.** 런타임 확인:
`DbusDevices = ["/org/shadowblip/InputPlumber/devices/target/dbus0"]` — **붙어 있다.**

**(b) 따라서 `QuickAccess2`/`Keyboard`가 "죽은 코드"라는 것도 조건부다.**
`src/input/event/dbus.rs:197-198`에 `QuickAccess2 → Action::Quick2("ui_quick2")`,
`Keyboard → Action::Keyboard` 매핑이 **실재한다.** pastaq의 "It is not a no-op"은 원리상 옳았다.

무동작인 **진짜 이유는 `InterceptMode`**다. `CompositeDevice::write_event`
(`composite_device/mod.rs:1080-1105`)는 intercept mode가 `Always`/`GamepadOnly`일 때만 일반
게임패드 이벤트를 DBus 타겟으로 보낸다. 이 기기는 `InterceptMode = 0`이라 evdev 코드도 DBus
시그널도 안 나온다. **실용적 결론은 그대로지만, 근거를 "DBus 타겟이 없어서"가 아니라
"intercept mode가 꺼져 있어서"로 말할 것.** CLAUDE.md 두 군데를 이에 맞춰 정정했다.

**미해결**: 08-27 실기기 테스트는 OGUI를 실제로 띄운 상태(`opengamepadui --overlay-mode`,
`gamescope-session-ogui-steam`)에서도 무반응이었다. 그때 왜 intercept mode가 안 켜졌는지는
여전히 설명되지 않았다. HOME 매핑을 다시 논의하게 되면 여기부터 파야 한다.

### 11. upstream 패키지 config와 지금 남은 차이 3가지

PR은 닫혔지만 upstream은 0.79.0에서 자체 Zotac config를 냈고, 우리 `/etc` override와 이렇게 다르다:

1. **`*/input0` xpad entry가 upstream엔 없다** — 없으면 ABXY·스틱·트리거가 전부 죽는다(09-08 실측).
   이 하드웨어에 대해선 우리 쪽이 정확하다.
2. **`QuickAccess2`/`Keyboard`를 쓴다** — §10에 따라 OGUI intercept 없이는 HOME 두 개가 무동작.
3. **다이얼 매핑이 양쪽 다 죽어 있다** — upstream도 `target_devices`가 `xbox-elite`/`mouse`/
   `keyboard`뿐이라 `Gamepad:Dial:*`을 받을 타겟이 없다.

**종합**: 닫힌 두 PR의 쟁점은 대부분 사후 해소됐다. 매핑 컨벤션 논쟁은 서로 다른 커널을 보고 있던
것이고(벤더 드라이버 하드웨어에서는 그가 옳다), 패들은 그의 판단대로 드라이버가 해결했고,
터치패드·`unique: false`는 그대로 채택했다. **남은 진짜 이견은 HOME 버튼 두 개뿐이며, 그것도
"OGUI intercept를 전제하느냐"의 차이라 옳고 그름이 아니라 전제의 차이다.**

### 12. upstream 이슈 후보 정리 (TODO 4) — 6건 검증 완료, 1건은 이미 등록돼 있었음

전부 **로컬 `main`(0.79.2)** 기준으로 재확인했다(작업트리 0.78.1로 읽지 말 것). 중복 제보를 피하려고
`gh search issues`로 upstream 기존 이슈도 확인했다.

**★ 먼저: `#690`이 우리 관측 하나를 이미 커버한다.**
`#690` (2026-09-02, OPEN) "capability_map_v2 fails to map capabilities properly" —
*"The v2 Capability Map fails to remove the source events and populate the mapped capabilities for a
source device. This results in the exclude/include list on a composite device to block all mapped
events and the device test to show original source capabilities."*
08-28부터 우리가 관측한 "`device N test`에 `Screenshot`/`QuickAccess` 박스가 안 뜬다"와
09-09에 확인한 "`Capabilities`에 `Guide`만 있다"가 바로 이것이다. **pastaq의 "다 나와야 한다"도
맞았고 우리 관측도 맞았다** — v2 맵의 알려진 버그이고, 그가 PR 664를 닫은 그날 직접 등록했다.
**새 이슈로 내지 말고 `#690`에 확인 데이터를 붙일 것.**

| # | 후보 | 0.79.2 검증 근거 | 판단 |
|---|---|---|---|
| 1 | 타겟 `run()` 루프 영구 사망 | `target/mod.rs:551-559` — `receive_commands` 에러가 `log::debug!` 한 줄 뒤 `break`, `stop()` 후 `Ok(())`. 위로 전파도 재생성도 없음 | 제출 |
| 2 | `unique` 기본값 `true` | `manager.rs:1036` `unwrap_or(true)` | 단독 제출 안 함 — 의도된 설계. 4번 본문에 포함 |
| 3 | `filtered_events` 무시 | `CapabilityMapConfigV1`/`V2` 어디에도 필드 없음, `capability_map_v2.json` 스키마에도 없음. 그런데 셰어된 capability map YAML **36개**가 이 키를 갖고 있음 | 제출 (PR로도 가능) |
| 4 | 동명 config 중복 로드 | `manager.rs:1673-1697` — 파일명·`name:` 어느 쪽으로도 dedup 없이 전부 `push` | 제출 |
| 5 | `REL_WHEEL`/`REL_HWHEEL` 축 붕괴 | 입력 `evdev.rs:83`이 `InputValue::Float`로 축 소실 → `:482`에서 `Mouse::Wheel`로 합류 → `:723-726`이 두 코드 모두에 출력 → `:984` Float 분기는 code 무관 | 제출 |
| 6 | 타겟이 선언 안 한 capability는 조용히 드롭 | `composite_device/targets.rs:303-312` | 제출 (본문에서 `#690`과 구분할 것) |

**전례**: `#531`(closed) "Deck target fails to run"의 로그에 `Error processing received command:
Target device stopped`가 DEBUG로 찍힌 뒤 타겟이 멈추는 형태가 그대로 보인다 — 후보 1의 실사례다.

**Zotac config 차이 3건(§11)은 새 이슈가 아니라 `#655`로.**
`#655` "Zotac gaming zone QAM and Guide /Steam Button don't work"가 **OPEN**이다. 패키지판 config에
`*/input0` xpad entry가 없어 표준 버튼이 죽는 문제는 그 이슈에 실사용자 데이터로 붙이는 게 맞다.

**미착수**: 실제 이슈 본문(영문) 작성 및 제출. 위 6건 중 5건 + `#690` 코멘트 + `#655` 코멘트.

### 13. 다음 세션 TODO (09-08 목록에서 갱신)

1. ~~리부트 후 컴포짓 1개 + 전 버튼 정상인지 확인~~ → **완료(이 세션). 정상.**
2. ~~다이얼 실동작 확인~~ → **완료(이 세션).** 볼륨/밝기로 동작, capability 매핑은 no-op.
3. ~~`~/zotac-zone-tools/zotac-zone-paddles` 정리~~ → **완료(이 세션).**
   `~/zotac-zone-tools/obsolete/`로 이동(삭제 아님, 벤더 프로토콜 CRC/프레임 구현이 참고 가치가
   있어서) + 이유를 적은 README 동봉. CLAUDE.md의 두 군데 경로 서술도 갱신함.
   ⚠️ **Steam 비-Steam 게임 등록은 아직 남아 있음** — `shortcuts.vdf`에 `zotac-zone-paddles.sh`
   항목 존재 확인. 바이너리 VDF이고 Steam 실행 중엔 덮어써지므로 직접 편집하지 않았다. 사용자가
   Steam 라이브러리에서 우클릭 → 관리 → 비-Steam 게임 제거로 지워야 함. (스크립트를 옮겼으므로
   지금 실행하면 실패한다 — 어차피 hidraw2가 root 전용 + InputPlumber 점유라 동작 불가)
4. ~~upstream 이슈 후보 정리~~ → **정리 완료(§12).** 6건 전부 0.79.2로 검증, 중복 검색까지 마침.
   **남은 것은 실제 제출**: 신규 이슈 5건(후보 1·3·4·5·6) + `#690`에 확인 코멘트 + `#655`에
   패키지 config의 `*/input0` 누락 보고. 영문 본문은 아직 안 씀.
5. ~~벤더 게임패드 entry의 `phys_path: "*/input1"` 제약 제거 검토~~ → **완료(이 세션). 제거함.**
   근거: `has_matching_evdev`의 name 매칭은 `glob_match` 전체 문자열 매칭이라
   (`src/config/mod.rs:855-861`) 이 entry의 두 이름 대안이 xpad 노드(`ZOTAC Gaming Zone`)와는
   절대 겹치지 않는다. 원래 제약 목적(중복 컴포짓 방지)은 `unique: false`가 대신한다.
   검증: 재시작 후 컴포짓 1개 / 소스 8개 / 노드 구성 / 경고로그 없음 전부 변경 전과 동일,
   사용자 실기기 버튼 확인 "동작하는 것 같습니다". 얻은 것은 이식성(pastaq가 보고한 `*/input3`
   토폴로지 대응). 배포본 백업: `/etc/inputplumber/50-zotac-zone.yaml.bak-20260909`
   (⚠️ `devices.d/` **안에** 백업을 두지 말 것 — 확장자가 `.yaml`이면 중복 config로 로드된다.
   지금 백업은 디렉터리 밖에 있고 확장자도 `.bak-*`라 안전).
   **xpad entry의 `*/input0`은 그대로 뒀다** — 같은 논리로 뺄 수 있지만 미검증이고, 잘못되면
   표준 버튼이 통째로 죽는 쪽이라 사용자 결정 대기.
6. ~~`upstream/main`을 받아 로컬 `main` 동기화~~ → **완료(이 세션).**
   `git fetch upstream main:main`으로 fast-forward: `23f84b7`(0.78.1) → `8e3c86b`(0.79.2).
   `claude` 브랜치는 손대지 않음. **여전히 체크아웃(claude)은 0.78.1 코드**이므로, 설치판 동작을
   설명할 땐 작업트리가 아니라 `git show main:<path>` 또는 `git show v0.79.0:<path>`로 읽을 것
   (CLAUDE.md 개발환경 섹션에 경고 추가함). **`main`을 `claude`에 머지하지 말 것.**
7. 미착수로 계속 남아있는 것: PR 코드 라인별 설명 듣기.

---

## 참고: 디버깅 기법

일반적인 디버깅 기법(hidraw 캡처, HID 리포트 디스크립터 파싱, capability_map chord 로깅 함정,
`pkill -f` 자기 셸 오살, 실기기 dev 바이너리 테스트 절차 등)은 전부 **CLAUDE.md "Debugging input
pipeline issues on real hardware" 섹션으로 이동함** — 거기가 정본.
