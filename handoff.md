# InputPlumber Zotac Gaming Zone 버그 수정 - 작업 인계 문서

## 상태 요약 (최신, 2026-08-27)

**PR #664: pastaq가 08-27 12:01 UTC에 답변함 — PR을 계속 진행하기로 함, 접을 필요 없음.**
아래 "★ 2026-08-27 오후: pastaq 답변 및 다음 액션" 섹션 참고. **다음 세션은 여기부터 시작할 것**
(실기기 테스트 3건 미착수).

**(구) PR #664: pastaq로부터 CHANGES_REQUESTED 2회(08-25, 08-26) — "손댈 필요 없음"이었던 08-26 요약은 틀렸음, 정정함.**
08-27에 실기기 재검증 + 코드 근거로 종합 반박 코멘트를 작성해 게시 완료. 자세한 내용은 아래
"★ 2026-08-27 세션" 참고.

**패들: 해결 → 구현 → Steam Game Mode 검증 → 커밋 완료.** 08-24 세션의 진단이 틀렸다는 것이 08-25 실측으로 확정됐고, 진짜 원인과 해결책을 찾아 코드로 구현한 뒤 실기기에서 전부 검증했다.

**다이얼: 프로토콜 완전 확정, 구현했으나 성능 문제로 철회.** 아래 "다음 세션 작업" 참고. 08-27에
벤더 드라이버 소스(OpenZotacZone/ZotacZone-Drivers)를 추가로 분석함 — "★ 2026-08-27 세션" 참고.

**HOME 버튼(짧게 Meta+D / 길게 Ctrl+Alt+KP.): 08-26에 매핑 완료, 실기기 검증 + 커밋 + push + PR 코멘트까지 완료.** 자세한 내용은 아래 "★ 2026-08-26 세션 핵심 발견" 참고. 08-25까지의 "미매핑 유지" 결정은 이걸로 뒤집혔다. **다만 이 매핑도 08-26 두 번째 리뷰에서 pastaq가 재반박함 — 08-27에 실기기로 재검증해서 반박 근거 확보, 아래 참고.**

**PR #668: 08-27 12:04 UTC에 pastaq가 APPROVED로 전환함 — 바로 이어서 "Reminder for me to squash
merge this PR" 코멘트 남김, 본인이 직접 병합할 예정으로 보임. CI는 `Run tests` SUCCESS, x86_64/
aarch64 build는 그 시점 IN_PROGRESS(그래서 mergeStateStatus가 일시적으로 UNSTABLE). 우리 쪽에서
더 할 일 없음 — 병합 여부만 확인하면 됨.**

**(구) PR #668: 08-26 요약("6건 전부 반영, 완료")도 틀렸었음 — 08-26 17:23에 pastaq가 후속 리뷰로
`hid_report.rs` line 10의 `packed_struct::prelude::*` 와일드카드 import를 추가로 지적("Seems
another one snuck in")했는데 반영이 안 돼 있었음. 08-27에 발견해서 즉시 named import로 고치고
커밋 `3389ed7` push 완료.** pastaq의 리뷰 상태 자체는 08-26 17:23 리뷰(COMMENTED)로 이미
CHANGES_REQUESTED에서 갱신되어 공식적으로는 블로킹 아님, CI 전부 SUCCESS, MERGEABLE. 아래 PR #668
항목과 "다음 세션 작업 1" 참고.

**⚠️ CLAUDE.md의 capability_map 아키텍처 설명이 08-27에 정정됨.** 이전까지 "번역은 소스와 무관하게
CompositeDevice 전체에 전역 적용된다"고 되어 있었는데, 이건 틀렸다 (실제로는
`source_devices[].capability_map_id`가 그 소스 하나만의 전용 translator를 만듦 — 전역 경로는 최상위
`capability_map_id`가 있을 때만 쓰이고 이 기기 config엔 없음). 이 세션 이전의 요약 다수(아래 08-25/
08-26 섹션 포함)가 이 틀린 설명을 인용하고 있으니, **다이얼/터치패드 회귀 원인을 다시 설명할 때는
"★ 2026-08-27 세션"의 정정된 버전을 우선할 것.**

- **PR: https://github.com/ShadowBlip/InputPlumber/pull/664** (커밋 7개, `whirlwind80:fix/zotac-zone-device-names` → `ShadowBlip:main`). 상태: OPEN / MERGEABLE. 08-27 기준 리뷰 2회 CHANGES_REQUESTED(pastaq), 08-27에 종합 반박 코멘트 게시 완료 (https://github.com/ShadowBlip/InputPlumber/pull/664#issuecomment-5438401478).
- **PR #668: https://github.com/ShadowBlip/InputPlumber/pull/668** — 패들 수정.
  브랜치 `fix/zotac-zone-paddles`, `upstream/main`(23f84b7) 바로 위에 리베이스됨.
  PR #664와 **독립적**이다: 패들 커밋은 `src/drivers/zotac_zone/`만 건드리고, 필요한 전제
  (interface 3 hidraw 소스, `KEY_HOME/KEY_END → LeftPaddle1/RightPaddle1` 매핑)는 upstream에 이미 있다.
  08-25에 리뷰어(`pastaq`)로부터 **CHANGES_REQUESTED** (인라인 코멘트 6건, 요지: `hid_report.rs`를
  다른 드라이버들처럼 `PackedStruct` 기반으로 다시 짜라는 것 + 주석 2곳 정리 + 테스트 파일 분리 +
  미사용 import 정리). 08-26에 커밋 `2535562`(리워크) + `f231971`(주석 한 줄로 재압축, 리뷰
  제안과 거의 동일하게)로 전부 반영, push 완료. 인라인 6건에 `Done in 2535562` 답글 + PR
  일반 댓글에 변경 요약 + 이후 `f231971` 추가 반영 후속 댓글까지 사용자가 직접 작성해서 등록 완료.
  **그런데 08-26 17:23에 pastaq가 후속 리뷰(COMMENTED)로 `hid_report.rs` line 10의
  `packed_struct::prelude::*` 와일드카드를 추가 지적함 — 이건 08-26 세션 요약에서 누락돼 있었고
  08-27에 발견해서 커밋 `3389ed7`로 named import(`PackedStruct`, `PackingError`,
  `PrimitiveEnum_u8`)로 교체, push 완료.** 참고로 리포 내 다른 13개 드라이버
  `hid_report.rs`는 전부 `use packed_struct::prelude::*;`를 그대로 쓰고 있어서 이 요청이 리포
  컨벤션과는 다소 안 맞지만, 간단한 수정이라 그냥 반영함(사용자 판단). 코드/검증 상세는 아래
  "다음 세션 작업 1" 참고.
- **PR #664에 커밋 두 개 추가됨** (`35b86f7` 왼쪽 터치패드 스크롤 회귀 수정, `61df761` HOME 버튼 Screenshot/Guide 매핑). 총 5개 커밋.
- **이슈 #655에 진행 상황 코멘트 남김** (https://github.com/ShadowBlip/InputPlumber/issues/655).
  같은 기기 사용자가 올린 이슈. 이전 세션 코멘트의 "다이얼과 백버튼 모두 신호 없음"이 사실과
  달랐으므로 정정했다(패들만 무신호, 다이얼은 rid=3으로 신호가 있으나 evdev에 안 뜸).
  ⚠️ CONTRIBUTING.md는 **AI로 리뷰어에게 답변하는 것을 금지**한다. PR 리뷰 코멘트 답변은
  반드시 사용자가 직접 작성할 것.
- **⚠️ `/etc/inputplumber/devices.d/` 와 `capability_maps.d/` 에 철회된 다이얼 버전 config가 남아 있을 수 있음.**
  저장소 최신본으로 재배포할 것:
  `sudo cp ~/InputPlumber/rootfs/usr/share/inputplumber/devices/50-zotac-zone.yaml /etc/inputplumber/devices.d/`
  `sudo cp ~/InputPlumber/rootfs/usr/share/inputplumber/capability_maps/zone_type1.yaml /etc/inputplumber/capability_maps.d/`
  `sudo systemctl restart inputplumber`
- **패들 임시 조치 (PR 머지 전까지):** 패키지 바이너리에는 #668 코드가 없으므로 아무도 매핑을
  설정해주지 않는다. 전원이 완전히 끊기면(따뜻한 재부팅은 유지될 수도 있음 — 미검증) 패들이 죽는다.
  그때는 `python3 ~/zotac-zone-tools/zotac-zone-paddles` 실행.
  **root 불필요** — `/dev/hidraw2`가 `crw-rw-rw-`라 일반 사용자로 열린다.
  **Game Mode에서 쓰려면** `~/zotac-zone-tools/zotac-zone-paddles.sh`(래퍼)를 Steam에 비-Steam
  게임으로 등록. Steam은 실행 파일만 등록되므로 `.sh` 래퍼가 필요하다
  (데스크톱 모드에서 Steam → 게임 → 비-Steam 게임 추가 → 파일 형식 "모든 파일" → `.sh` 경로 선택).
  사용자는 자동 실행(udev/systemd) 대신 **수동 실행**을 선택했고, 기기에 영구 저장(`CMD_SAVE_CONFIG`)도
  하지 않기로 했다. #664 + #668이 모두 머지되어 이미지에 반영되면 이 조치는 전부 불필요해진다.
- **실기기 현재 상태: 정상.** `inputplumber.service` enabled + active, 패키지 바이너리 실행 중. 컨트롤러 정상 사용 가능.
  - ⚠️ **패들 remap은 휘발성** — 지금은 동작하지만 전원 껐다 켜면 사라짐 (`CMD_SAVE_CONFIG`를 일부러 안 보냄). 재부팅 후 다시 테스트하려면 `python3 ~/zotac-zone-tools/setmap.py` 재실행 필요.

## Bazzite 업데이트와 `/etc` override의 수명

일반 설명(왜 유지되는지, 머지 확인/정리 명령)은 **CLAUDE.md "This machine" 섹션으로 이동함** —
거기가 정본. 여기 남기는 건 이 세션에서만 확인된 사실뿐:
- `rpm -qf /etc/inputplumber` → "not owned by any package" (2026-08-25 확인).
- **정리 완료**: `.d` 없는 `/etc/inputplumber/devices/`(한 번도 로드된 적 없는 어제 세션의 잔재)는
  2026-08-25에 삭제함.
- 2026-08-25 기준 설치 패키지는 `inputplumber-0.78.0-5.fc44`, `grep -c "phys_path" ...` 결과 `0`
  (아직 #664 미반영 상태).

## 목표

Zotac Gaming Zone (모델명 G0A1W)에서 Bazzite 44.x (InputPlumber 기반) 전환 후 발생한 입력 문제 수정, upstream(ShadowBlip/InputPlumber)에 제출.

## 환경

- 기기: Zotac Gaming Zone (board_name: G0A1W / G1A1W, sys_vendor: ZOTAC), USB 0x1ee9:0x1590, fw 1.3.9
- OS: Bazzite 44.20260820 (bootc/rpm-ostree, ghcr.io/ublue-os/bazzite-deck:stable)
- 개발 환경: toolbox (`inputplumber-dev`), Rust 1.98.0 — 상세(flatpak-spawn, sudo 비대화식 등)는
  **CLAUDE.md "Development environment on this machine" 섹션 참고**
- 저장소: origin = whirlwind80/InputPlumber (fork), upstream = ShadowBlip/InputPlumber
- 로컬 경로: `~/InputPlumber` (= `/var/home/whirlwind80/InputPlumber`)
- git 브랜치 `fix/zotac-zone-device-names`, origin push 완료, 커밋 7개:
  1. `c00deba` fix(Zotac Zone): correct evdev device names for gamepad and keyboard
  2. `399510c` fix(Zotac Zone): swap Steam/QAM button capability bindings
  3. `193328a` fix(Zotac Zone): deduplicate composite device and fix View button
  4. `35b86f7` fix(Zotac Zone): stop the dial mappings from swallowing touchpad scrolling
  5. `8d89ead` fix(Zotac Zone): address PR #664 review feedback on capability map and evdev names
     (08-26 첫 리뷰 대응: 다이얼 매핑을 `zone_type1_dial.yaml`/`zone1_dial`로 분리 + 터치패드
     source에서 `capability_map_id: zone1` 제거, evdev name을 덮어쓰기 대신 glob으로, 주석 정리.
     이전 요약에서 누락돼 있었음 — 08-27에 발견.)
  6. `61df761` fix(Zotac Zone): map real HOME button chords to Screenshot/Guide
  7. `9629ea6` fix(Zotac Zone): trim verbose capability_map comments per PR #664 review
     (08-27, 두 번째 리뷰의 주석 정리 요청 반영 — 118-16-line 파일 정리)

---

## ★ 오늘의 핵심 발견 (2026-08-25)

### 1. 어제 진단이 틀린 부분 — rid=3은 패들이 아니라 **다이얼**이다

어제 "Report ID 3의 byte[3] 비트 0x08 = 왼쪽 패들, 0x01 = 오른쪽 패들"로 결론냈으나 **오진**이었다.
재부팅 후 통제된 조건에서 여러 번 캡처한 결과와, 벤더 커널 드라이버 소스가 **비트 단위로 일치**:

| 비트 | HID usage (디스크립터) | 실제 정체 |
|---|---|---|
| 0x01 | 0xb5 Scan Next Track | 오른쪽 다이얼 시계방향 |
| 0x02 | 0xb7 Stop | 오른쪽 다이얼 반시계방향 |
| 0x08 | 0xe9 Volume Increment | 왼쪽 다이얼 시계방향 |
| 0x10 | 0xea Volume Decrement | 왼쪽 다이얼 반시계방향 |

벤더 드라이버(`zotac-zone-hid-core.c`)의 정의와 완전 일치:
```c
#define ZOTAC_RIGHT_DIAL_CW_BIT 0
#define ZOTAC_RIGHT_DIAL_CCW_BIT 1
#define ZOTAC_LEFT_DIAL_CW_BIT 3
#define ZOTAC_LEFT_DIAL_CCW_BIT 4
```

어제 "패들을 눌렀다"고 기록된 캡처는 실제로는 다이얼을 함께 돌린 것이었음. 방향 통제 실험(시계방향 10초 → 정지 5초 → 반시계방향 10초)에서 `0x08`(127회) / `0x10`(122회)이 구간별로 완벽히 갈렸다.

**어제 "read()가 계속 WouldBlock만 반환"한 것은 코드 버그가 아니었다.** 패들이 애초에 아무 리포트도 안 보내고 있었을 뿐. hidapi도, USB 상태 꼬임도, InputPlumber도 무관.

### 2. 패들이 무신호였던 진짜 이유 — 기기 펌웨어의 버튼 매핑이 비어 있음

interface 3(설정 인터페이스, `/dev/hidraw2`)에 `CMD_GET_BUTTON_MAPPING`(0xA2)을 보내 확인:

```
button M1 (0x01):  gamepad=00 00 00 00  mod=00  kbd=00 00 00 00 00 00  mouse=00
button M2 (0x02):  gamepad=00 00 00 00  mod=00  kbd=00 00 00 00 00 00  mouse=00
button A  (0x0F):  gamepad=00 10 00 00   ← 일반 버튼은 매핑이 들어 있음
```

**M1/M2는 매핑이 전부 0** = 펌웨어가 "보낼 게 없음"으로 판단 → USB로 아무것도 안 나감.
소프트웨어로는 해결 불가. 기기에 매핑을 써넣어야만 함.

이것이 upstream 코드가 원래 하려던 일이었다:
```rust
set_attribute(&mut device, "btn_m2/remap/keyboard", "home");
set_attribute(&mut device, "btn_m1/remap/keyboard", "end");
```
벤더 커널 드라이버 `zotac_zone_hid`가 sysfs로 이걸 해주는데, **이 커널에는 그 모듈이 아예 없음**
(`modinfo zotac_zone_hid` → not found, 3개 인터페이스 전부 `hid-generic` 바인딩).

### 3. 해결 — 설정 프로토콜을 직접 구현해서 M1/M2 remap (실기기 검증 완료 ✅)

`/dev/hidraw2`에 `CMD_SET_BUTTON_MAPPING`(0xA1) 2개를 보내 M1→End, M2→Home 설정 →
**패들이 즉시 KEY_HOME/KEY_END를 보내기 시작함.**

```
[19.11] 02 00 00 4a  ← 왼쪽 패들 = Home (M2)
[21.01] 02 00 00 4d  ← 오른쪽 패들 = End  (M1)
```

그리고 `zone_type1.yaml` capability_map이 **이미** `KEY_HOME → LeftPaddle1`, `KEY_END → RightPaddle1`로
되어 있어서 **좌우까지 그대로 맞음. capability_map 수정 불필요.**

---

## 벤더 설정 프로토콜 (interface 3 = `/dev/hidraw2`)

전부 검증 완료. 프레임 형식/CRC/명령 코드/페이로드 상세는 **CLAUDE.md "Vendor config protocol"
섹션으로 이동함** — 거기가 정본.

### 검증에 쓴 스크립트

`~/zotac-zone-tools/` 에 보관됨 (저장소 밖, 재부팅에도 유지). 벤더 드라이버 C 소스 사본은 `~/zotac-zone-tools/vendor-driver-reference/`:
- `capture.py <node> <sec>` — hidraw 단일 노드 캡처
- `multicap.py <node...> <sec>` — 다중 노드 동시 캡처
- `rdesc.py` / `rid3.py` — HID 리포트 디스크립터 덤프/파싱
- `crc.py` — CRC 구현 검증 (기기 접근 없음)
- `getcmd.py` — 읽기 전용 GET 명령 (device info, button mapping)
- `setmap.py` — M1→End, M2→Home remap 적용. `/dev/hidraw2` 하드코딩이라 노드 번호가 바뀌면 실패함
- `clearmap.py` — M1/M2 매핑을 비움 (초기 상태 재현용, 코드 검증에 사용)
- **`zotac-zone-paddles`** — 위 setmap의 개선판. sysfs에서 vendor/product + interface 3으로
  **노드를 자동 탐색**하므로 번호가 바뀌어도 동작. 실행 권한 있음(shebang 포함).
  **전원을 완전히 껐다 켠 뒤 패들이 안 되면 이걸 실행하면 됨.**
- **`zotac-zone-paddles.sh`** — 위 스크립트를 절대 경로로 호출하는 한 줄 래퍼.
  Steam 비-Steam 게임 등록에 쓴다 (Steam이 실행 파일만 받으므로).

---

## 하드웨어 실측 정보

hidraw/evdev 노드 레이아웃, 버튼→신호 매핑 등 상세는 **CLAUDE.md "Hardware reference" 섹션으로
이동함** — 거기가 정본. `/dev/hidraw*`는 `crw-rw-rw-`라 root 없이 읽기/쓰기 가능하다는 것만
여기 재확인용으로 남김 (InputPlumber가 hide하기 전까지).

---

## 확정된 근본 원인 (PR #664, 전부 수정 + 검증 완료)

### 원인 1: evdev name 필드 불일치 (`c00deba`)
`50-zotac-zone.yaml`의 gamepad/keyboard evdev name이 실제 커널 보고 이름과 달랐음.

### 원인 2: `/etc` override 경로 트랩 (로컬 설정 문제, git과 무관)
override는 `/etc/inputplumber/devices.d/`, `/etc/inputplumber/capability_maps.d/` (**`.d` 필수**).
`/usr/share/inputplumber/devices`(`.d` 없음)와 비대칭이라 헷갈림. **upstream 이슈 등록 검토 가치 있음 (아직 안 함).**

### 원인 3: capability_map의 F17/F18 바인딩이 뒤바뀜 (`399510c`)
F17→`QuickAccess`(= Guide+South 콤보로 합성되어 QAM처럼 동작), F18→`QuickAccess2`(evdev 코드 빈 배열 = 미구현).
F17→`Guide`, F18→`QuickAccess`로 스왑.

### 원인 4: View 버튼 + 중복 가상 컨트롤러 (`193328a`)
gamepad evdev name이 event2/5/6에 전부 매칭되어 CompositeDevice 3개 생성 → `phys_path: "*/input1"` 추가.
실제 게임패드 기능은 `event14`(커널 xpad)에서 나오는데 config에 없어서 Steam이 raw로 직접 읽고 있었음 → 소스로 추가해 grab.

**실기기 검증 완료:** Steam 버튼 ✅ / QAM ✅ / 스틱·ABXY·숄더·트리거·D패드 ✅ (가상 컨트롤러 1개로 통합) / View ✅

---

## 다음 세션 작업

### -2. PR #668 — pastaq APPROVED (08-27 12:04 UTC), 병합 대기만 남음

"Reminder for me to squash merge this PR" 코멘트로 봐서 pastaq 본인이 직접 머지할 예정. 다음
세션 시작 시 `gh pr view 668 --repo ShadowBlip/InputPlumber --json state,mergedAt`로 머지 여부만
확인하면 됨 — 병합됐으면 이 리포의 로컬 override(`/etc/inputplumber/...`)는 그대로 유지, upstream
버전이 배포에 반영되기 전까지는 CLAUDE.md 안내대로 계속 override로 운용.

### -1. ★★ 최우선 — pastaq 08-27 답변에 대한 실기기 검증 3건 (2026-08-27 오후 세션에서 코드 분석만 완료, 실기기 테스트는 미착수 — "내일 할게요"로 보류됨)

pastaq가 08-27 12:01 UTC 코멘트(https://github.com/ShadowBlip/InputPlumber/pull/664#issuecomment-5438753815)에서
**PR을 계속 진행하기로 확정**하고 아래 3가지를 요청함.
다이얼 건은 pastaq 본인 착각이었다고 인정했으므로 추가 조치 불필요.

**1) HOME/QAM 매핑을 `inputplumber device 0 test`로 직접 검증**

pastaq는 OpenGamepadUI 같은 "부실한 userspace 스택"을 거치지 말고 InputPlumber 자체 테스트
TUI로 확인하라고 요청함 (`src/cli/device.rs`의 `DeviceCommand::Test` → `DeviceTestMenu`, DBus로
실행 중인 데몬에 붙어서 라이브 capability 이벤트를 보여줌 — 데몬을 stop/mask할 필요 없음, sudo도
불필요할 가능성 높음). 실행:
```
flatpak-spawn --host inputplumber devices list      # Zotac Zone의 CompositeDevice 번호 확인
flatpak-spawn --host inputplumber device <N> test    # TUI에서 HOME 짧게/길게, MORE(F17) 눌러보기
```

**2) 5초 파워메뉴의 원인 규명 — `libinput debug-events` + `evtest` (mask 필요, sudo)**

pastaq 추측: "LEFTMETA+D가 powerbuttond에 감지된 것"(HOME 길게 chord가 아니라 시스템 레벨 동작).
CLAUDE.md "Debugging input pipeline issues on real hardware" 섹션의 grab-release 절차 그대로:
```
sudo systemctl mask inputplumber && sudo systemctl stop inputplumber
# 터미널 두 개에서 동시에:
sudo libinput debug-events
sudo evtest
# HOME 5초 길게 눌러서 관찰
sudo systemctl unmask inputplumber && sudo systemctl start inputplumber
```

**3) 터치패드 `source_devices` entry(그룹 `mouse`, evdev name glob `{ZOTAC Gaming Zone
Mouse,ZOTAC Gaming Zone Dials,Zotac Technology Limited ZOTAC GAMING ZONE Mouse}`) 제거 검토**

pastaq 질문: "다른 버튼이 그 evdev로 안 지나간다면 grab 자체가 불필요하지 않은가."

★ 코드/히스토리 분석 완료 (2026-08-27 오후, 실기기 테스트는 아직):
- 이 entry엔 `capability_map_id`가 없음 — 순수 passthrough (번역 없이 그대로 InputPlumber의
  가상 `mouse` target으로 전달).
- 이 entry가 존재하는 근본 이유는 **pastaq 본인이 작성한 PR #410**("Fix(Hardware Support):
  Passthrough Zotac Zone Touchpads", 커밋 `0da2482`)과 직결됨 — 그 커밋 메시지: *"rel events
  produced have no target device to emit from"*. 즉 grab을 안 하면 REL 이벤트를 받아줄 target이
  없어서 터치패드가 SteamOS에서 동작 안 했다는 게 원래 문제였고, 이 config의
  `target_devices: [xbox-elite, mouse, keyboard]`에 `mouse`가 포함된 것도 이 문제 해결용으로 보임.
- 다만 PR #410 당시엔 "ZOTAC Gaming Zone Mouse"(포인터)와 "ZOTAC Gaming Zone Dials"(스크롤,
  현재는 "왼쪽 터치패드"로 정정된 그 이름)가 **서로 다른 evdev 노드**였던 것으로 보이는데, 이
  기기(hid-generic 전용, 벤더 드라이버 없음)에서는 왼쪽 스크롤 터치패드와 오른쪽 포인터
  터치패드가 **하나의 evdev 노드(event4)로 합쳐져** 나오는 것으로 보임(CLAUDE.md Hardware
  reference 참고: `event4` = "...ZONE Mouse" (`REL_X`/`REL_Y`/`REL_WHEEL`/`REL_HWHEEL`) 전부 한
  노드).
- **결론: 코드만으로는 판단 불가.** 두 가지 옵션이 있음: (a) 현행대로 grab 유지 → InputPlumber의
  가상 mouse target을 거쳐 나감, (b) entry 제거 → PR #410 원래 방식대로 grab 안 하고 네이티브로
  노출. 이 기기에선 두 터치패드가 한 노드로 합쳐져 있어서 (b)로도 포인터+스크롤이 그대로 동작할
  가능성이 있지만, 실기기 A/B 테스트(entry 주석 처리 → 데스크톱/Gaming Mode 둘 다에서 포인터
  이동 + 왼쪽 터치패드 스크롤 확인, Steam Input이 이 장치를 이상하게 인식하지 않는지도 확인) 전에는
  결론 낼 수 없음.

### 0. ★ 최우선 — PR 코드 라인별 설명 듣기 (사용자 요청)

CONTRIBUTING.md의 *"you must be prepared to explain every line that was generated"* 를 충족하기 위해,
제출한 PR의 코드를 **한 줄씩 설명받기로 함.** 아직 진행하지 않았음. 다음 세션에서 이것부터 할 것.

설명이 필요한 대상 (08-26 리워크로 `hid_report.rs`의 구체적인 함수/구조체 이름이 바뀌었음 —
아래는 최신 기준):
- `src/drivers/zotac_zone/hid_report.rs` (PR #668, `2535562`로 `PackedStruct` 기반 전면 재작성)
  - `calc_crc()` — 여전히 일반 함수(테이블 없는 바이트 단위 체크섬, `PackedStruct`는 체크섬을
    계산해주지 않으므로). `h1`~`h4` 각 단계가 무엇인지, 왜 검사 범위가 절대 오프셋 `5..0x3F`인지
    (시퀀스 번호가 왜 제외되는지), C의 u32 중간 연산을 Rust로 옮기면서 자릿수 잘림이 왜 결과에
    영향을 주지 않는지. `pub(crate)`로 열어둔 이유(캡처된 실제 프레임의 command 바이트가
    `Command` enum의 유효한 값이 아니라서 타입 세이프 API로는 테스트를 못 만듦)도 설명 대상.
  - `ButtonMappingRequest`/`ButtonMappingResponse` (`PackedStruct` derive) — 각 필드의 바이트
    오프셋 의미, `Command`/`ButtonId` enum 도입 이유, `ButtonId::sequence()`가 왜 필요한지.
  - `ButtonMappingRequest::to_bytes()` — `pack()`으로 만든 버퍼에 CRC를 나중에 덮어쓰는 이유
    (체크섬은 나머지 바이트가 다 채워진 뒤에야 계산 가능하므로).
- `src/drivers/zotac_zone/driver.rs` (PR #668)
  - `configure_via_sysfs()`가 bool을 반환하는 폴백 구조, 실패를 `Err`가 아니라 `warn!`으로
    처리하는 이유, `send_mapping()`의 ack 대기 루프가 왜 명령 코드로 필터링해야 하는지
    (기기가 2초마다 텔레메트리를 자동 송신하므로).
- `rootfs/usr/share/inputplumber/capability_maps/zone_type1.yaml` / `zone_type1_dial.yaml`
  (PR #664 커밋 `35b86f7`, `8d89ead`)
  - ⚠️ 08-27에 정정됨: "CompositeDevice 전체에 전역 적용"이 아니라
    `source_devices[].capability_map_id`가 소스별 전용 translator를 만든다는 게 실제 구조
    (`input/source/evdev.rs`, `gamepad.rs`). 원래 회귀는 터치패드 source 자신이 다이얼 규칙이
    든 맵을 갖고 있어서 자기 이벤트를 잘못 번역한 것. 설명 대상은 "★ 2026-08-27 세션" 섹션의
    정정된 버전 기준으로 할 것 — CLAUDE.md Architecture 섹션도 이미 이 내용으로 갱신됨.

참고: 리뷰어 응답은 **반드시 사용자가 직접 작성해야 함** (CONTRIBUTING.md에서 AI로 리뷰어에게
답변하는 것을 명시적으로 금지). 설명을 듣는 것과 답변을 대신 쓰게 하는 것은 다름.

### 1. 패들 — 구현 + 리뷰 대응 + 재검증 전부 완료 ✅ (08-26)

원래 구현(`5e17fbb`)에 리뷰어 `pastaq`가 CHANGES_REQUESTED를 남겼고, 08-26에 전부 반영했다.

**리뷰로 바뀐 것 (`2535562`, `f231971`):**
- `hid_report.rs`를 `PackedStruct` 기반으로 전면 재작성 — `Command`/`ButtonId` enum 도입
  (`ButtonId::sequence()`로 버튼→시퀀스 번호 변환), 기존 오프셋 상수 + 배열 조작 함수
  (`build_frame`/`finish_frame`/`set_keyboard_mapping`/`mapping_status`)는
  `ButtonMappingRequest`/`ButtonMappingResponse` 구조체로 대체. 이 저장소의 다른
  `drivers/*/hid_report.rs`들과 동일한 컨벤션.
- 유닛 테스트를 `hid_report_test.rs`로 분리 (마찬가지로 기존 컨벤션), `use super::*;` 대신
  명시적 import로.
- `driver.rs`의 주석 2곳을 리뷰어 제안대로 정리.
- `calc_crc`는 여전히 일반 함수(체크섬은 `PackedStruct`가 대신 해주지 않음)로 남기되
  `pub(crate)`로 열어서, 캡처된 실제 프레임(command 바이트가 `Command` enum엔 없는 값)에 대한
  CRC 검증 테스트가 타입 세이프 API를 안 거치고도 계속 동작하게 함.

원래 구현 요지는 그대로: interface 3에서 `Driver::new()` 시 매크로 버튼 설정 — 벤더 커널
드라이버가 있으면 기존 sysfs 경로, 없으면 hidraw로 `CMD_SET_BUTTON_MAPPING` 전송(M2→Home,
M1→End), M1/M2 독립 설정, `CMD_SAVE_CONFIG` 안 보냄(휘발성), yaml/capability_map 수정 불필요.

**검증 결과 (08-26, 리워크 후 재검증):**
- `cargo clippy --all -- -D warnings` 클린, 유닛 테스트 13/13 통과.
- 실기기(하드웨어 레벨): 매핑을 비운 상태에서 dev 바이너리 기동 → 로그에
  `config interface is bound to 'hid-generic' … configuring it over hidraw instead` /
  `Mapped Zotac Zone button 0x02 to key 0x4a` / `… 0x01 to key 0x4d` 확인 (status=0 수락).
  `sudo python3 ~/zotac-zone-tools/getcmd.py`로 InputPlumber와 무관한 별도 경로로 기기
  매핑 테이블을 재확인해서 바이트까지 일치함을 교차 검증함.
- **Steam Game Mode 실기기 검증 완료 ✅** — 패들이 게임패드 패들 버튼으로 정상 동작, 기존
  버튼들(Steam/QAM/View/스틱/ABXY/숄더/트리거/D패드)도 회귀 없음.
  ⚠️ 이 게이밍 모드 검증은 단순히 `sudo ... &`로 백그라운드 실행한 dev 바이너리로는 **안 됨** —
  게이밍 모드 전환이 데스크톱 로그인 세션을 끊어서 거기 딸린 프로세스가 같이 죽는다(한 번
  실제로 겪음: 컨트롤러가 완전히 안 잡히는 상태가 됨). 세션과 무관하게 살아있는 정식 systemd
  서비스로 dev 코드를 돌리려면, CLAUDE.md "Debugging input pipeline issues on real hardware"의
  "usroverlay + `/usr/bin/inputplumber` 교체" 절차를 쓸 것 (`ExecStart` 드롭인은 SELinux가
  막으므로 이 방법이 필요함).
  ⚠️ 또한 dev 바이너리를 `~/InputPlumber`에서 직접 실행하면, 그 브랜치 자체의
  `./rootfs/usr/share/inputplumber/devices`가 상대경로로 추가 설정 레이어로 잡혀서 `/etc`
  override와 충돌해 CompositeDevice가 중복 생성될 수 있다 (`fix/zotac-zone-paddles`처럼
  #664 이전 커밋에 기반한 브랜치에서 실제로 겪음). `/usr/bin`에 설치해서 실행하면 이 문제가 없음.

**PR 코멘트 대응 완료:** 인라인 6건에 `Done in 2535562`(주석 하나는 이후 `f231971`로 갱신 답글
추가), PR 일반 댓글에 변경 요약 등록 — 전부 사용자가 직접 작성 (CONTRIBUTING.md 규정).

커밋 시 주의: CONTRIBUTING.md에 따라 AI 생성분을 커밋 본문에 명시해야 하고,
벤더 드라이버 출처(GPL-2.0-or-later, Luke D. Jones / OpenZotacZone)도 표기할 것 — 이미 반영됨.

### 2. 다이얼 — 구현했다가 **철회함** (프로토콜은 완전히 확정)

구현해서 실기기에서 동작까지 확인했으나, **시스템 전반이 버벅여서 되돌렸다.** 커밋하지 않았고
working tree에도 남아 있지 않다. 다시 하려면 아래 정보로 처음부터 재작성하면 된다.

확정된 프로토콜 (재현 불필요):
- `/dev/hidraw0` (USB interface 1), Report ID 3, 4바이트. `byte[3]`의 비트:
  `0x01` 우 CW / `0x02` 우 CCW / `0x08` 좌 CW / `0x10` 좌 CCW.
- 각 디텐트는 **press + release 펄스**로 오므로, 이전 상태와 비교해 새로 눌린 비트만 세면 된다.
- 벤더 드라이버는 좌 다이얼 → `REL_HWHEEL`, 우 다이얼 → `REL_WHEEL`로 내보낸다.

시도했던 구현과 성능 문제:
- `Driver`에 hidapi 논블로킹 핸들을 열고, `poll_dials()`가 폴링마다 대기 중인 리포트를 전부 드레인,
  `Capability::Mouse(Mouse::Wheel)` + `Vector2{x: 좌, y: 우}`로 방출. `poll_rate` 4ms.
- **버벅임의 유력한 원인**: interface 1에는 다이얼(rid=3)뿐 아니라 **터치패드/마우스(rid=4)** 와
  키보드(rid=2) 리포트가 같이 흐른다. 터치패드를 만지면 초당 수백 개가 쏟아지는데 그걸 4ms마다
  전부 읽어서 버리는 비용이 계속 발생한다. InputPlumber가 hidraw0을 hide하는 부작용도 있다.
- 다시 한다면: 폴링 대신 이벤트 기반(epoll) 읽기, 또는 poll_rate 완화, 혹은 rid=3 외 리포트를
  읽지 않도록 하는 방법을 먼저 검토할 것.

**부수 성과 — 왼쪽 터치패드 회귀를 발견하고 고침 (커밋 `35b86f7`, PR #664에 추가됨) ✅**

다이얼을 조사하다 발견했다. `zone_type1.yaml`의 `REL_HWHEEL → LeftStickDial`,
`REL_WHEEL → RightStickDial` 매핑은 다이얼이 evdev에 안 나오므로 다이얼용으로는 한 번도 발동한 적이
없었다. 그런데 **capability_map 번역은 소스별이 아니라 CompositeDevice 전체에 capability 기준으로
적용된다**(`composite_device/mod.rs`의 `process_event()`가 `translatable_capabilities.contains(&cap)`만
확인). 그래서 **왼쪽 터치패드**가 내는 wheel 이벤트가 dial capability로 바뀌었고, 이 기기의 타겟
(`xbox-elite`/`mouse`/`keyboard`) 중 `Gamepad::Dial`을 소비하는 게 없어서 전부 버려졌다.

**이 기기는 트랙패드가 2개다: 왼쪽 = 스크롤 전용 면, 오른쪽 = 포인터.** 즉 왼쪽 패드가 통째로
죽어 있었다.

이 회귀는 `c00deba`(mouse 소스의 evdev name 수정) 때문에 드러났다. 그 전에는 이름이
`ZOTAC Gaming Zone Dials`로 실제와 달라 소스가 매칭되지 않았고, 터치패드가 InputPlumber를
거치지 않아 정상 동작했다.

**실기기 검증 완료**: 매핑 2개 제거 후 왼쪽 트랙패드 스크롤 복구, 오른쪽 포인터 정상,
Steam 버튼·QAM 버튼 모두 정상 (회귀 없음).

### 3. 기타 미결

- **`devices` vs `devices.d` 네이밍 트랩** — upstream 이슈 미등록.
- **`/etc` override의 bootc 업데이트 지속성** — 미확인. PR 머지되면 무의미해짐.

---

## ★ 2026-08-26 세션 핵심 발견 — HOME 버튼 Screenshot/Guide 매핑

### 배경

`zone_type1.yaml`에 HOME 버튼의 real chord(`Meta+D` 짧게 / `Ctrl+Alt+KP.` 길게)를
`GamepadButton::QuickAccess2`/`Keyboard`로 매핑하는 커밋이 (다른 세션에서) 이미 작업트리에
올라와 있었음. 실기기(게이밍 모드)에서 테스트해보니 짧게=화면 캡쳐, 길게=Steam 메뉴로
동작했는데, **간헐적**이었고 코드상 설명이 안 됐음 — 그래서 원인 조사부터 시작.

### 핵심 발견: QuickAccess2/Keyboard는 이 기기에서 죽은 코드였다

- `src/input/event/evdev.rs`의 `event_codes_from_capability`: `QuickAccess2`/`Keyboard` 둘 다
  xbox-elite 타겟에서 **빈 evdev 코드 벡터**를 반환 (아무 신호도 안 나감).
- `src/input/target/xpad.rs::write_event`는 `QuickAccess`(→Guide+South 콤보)와
  `Screenshot`(→`KEY_RECORD`)만 특수 처리. `QuickAccess2`/`Keyboard`는 처리 없음.
- 애초에 `50-zotac-zone.yaml`의 `target_devices`가 `xbox-elite`/`mouse`/`keyboard`뿐이라
  **DBus 기반 `unified_gamepad` 타겟 자체가 이 기기에 연결돼 있지 않음** — 예전 커밋 주석의
  "OpenGamepadUI가 DBus로 읽는다"는 전제 자체가 이 기기에는 적용 안 됨.

### 왜 "화면 캡쳐 / Steam 메뉴"가 (간헐적으로) 보였나

`sudo journalctl -u inputplumber -f`를 `LOG_LEVEL=trace`로 걸어도 HOME 버튼 관련 로그가
**전혀** 안 찍힘 — 처음엔 이벤트가 아예 안 들어오는 줄 알았으나, 코드를 보니
`src/input/source/evdev/keyboard.rs::poll()`에서 chord 매핑에 속한 이벤트는
`translator.translate()`로 바로 들어가서 `KeyboardEventDevice::translate()`의
`log::trace!("Received event: ...")` 라인을 아예 안 거침 — "로그 없음"이 "이벤트 없음"을
의미하지 않았음.

실제 원인은 hidraw/HID debugfs(`/sys/kernel/debug/hid/<id>/events`)와, InputPlumber의 grab을
잠깐 풀고(`systemctl mask` + `stop`) `evtest`로 직접 확인해서 찾음:
- 하드웨어/커널은 완벽하게 정상 — 매번 정확히 `KEY_LEFTMETA`+`KEY_D` / `KEY_LEFTCTRL`+`KEY_LEFTALT`+`KEY_KPDOT`를 보냄.
- InputPlumber가 grab을 안 하고 있을 때 HOME을 짧게 누르면 **KDE의 기본 단축키("바탕화면 보기",
  Meta+D)가 그대로 실행됨** — 즉 raw 키가 데스크톱/게임 세션으로 새어나가면 그 세션의 자체
  단축키가 반응한다는 게 실증됨.
- 결론: 예전 세션이 봤던 "화면 캡쳐/Steam 메뉴"는 InputPlumber의 capability_map이 한 일이
  아니라, grab이 순간적으로 풀렸을 때(재시작 경합 등) raw chord가 새어나가 게이밍 세션 자체의
  기본 단축키가 반응한 부작용이었을 가능성이 높음.

### 조치

`QuickAccess2`/`Keyboard` 대신 **`Screenshot`(짧게)/`Guide`(길게)**로 target 변경 —
xbox-elite에 실제로 evdev 출력이 연결된 두 버튼이라, 예전에 우연히 새어나가던 동작을 의도된
정식 매핑으로 만듦.

**트레이드오프 (사용자 확인 후 진행):** chord가 소비되면 raw 키가 더 이상 데스크톱 세션으로
안 새어나가므로, **데스크톱 모드에서 KDE의 Meta+D "바탕화면 보기" 단축키는 더 이상 동작하지
않음.** capability_map 번역은 세션 종류를 구분하지 않으므로 게이밍/데스크톱 모드 둘 다 적용됨.

**결과:** 커밋 `61df761`, push 완료, PR #664에 설명 코멘트 등록 완료(사용자가 직접 작성 — AI
번역 사용 명시). 실기기 게이밍 모드에서 짧게/길게 반복 테스트로 일관 동작 확인.

**정리 필요했던 잔재:** 디버깅용으로 추가한 `/etc/systemd/system/inputplumber.service.d/99-trace-debug.conf`
(`LOG_LEVEL=trace`)를 작업 종료 후 삭제 안 하고 넘어갈 뻔함 — 최종 점검에서 발견해서 제거함.
**다음에 이런 drop-in을 추가하면 세션 끝나기 전에 지웠는지 반드시 다시 확인할 것.**

---

## ★ 2026-08-27 세션 — PR #664 두 번째 리뷰 대응, capability_map 아키텍처 정정

### 배경

08-26 요약에 "PR #664 안정적, 손댈 필요 없음"이라고 썼던 게 틀렸음이 이 세션에서 드러남 —
pastaq가 08-26 17:41~18:10에 `61df761`(HOME 버튼 매핑)에 대해 두 번째 CHANGES_REQUESTED를 남겼는데,
그걸 반영 안 한 채로 08-26 세션이 종료됐던 것. 인라인 코멘트 15건 이상 확인함.

### 1. HOME/QAM 매핑 재검증 — 실기기(데스크톱 + Gaming Mode) 둘 다 반증

pastaq 주장 핵심: "OpenGamepadUI의 intercept mode/Profile이 QuickAccess2를 가로채서 동작시킨다,
no-op이 아니다." 이걸 실기기로 직접 검증:

- 이 기기의 기본 Gaming Mode 세션이 이미 `gamescope-session-ogui-steam`("Steam Big Picture Plus",
  OpenGamepadUI QAM 포함, `/etc/sddm.conf.d/*.conf`에 `Session=gamescope-session-ogui-steam.desktop`
  로 설정돼 있음)이라는 걸 발견 — 별도 설치 불필요.
- 데스크톱 모드에서도 OpenGamepadUI를 띄울 수 있음: `ogui-overlay-mode.service`
  (`/usr/bin/opengamepadui --overlay-mode`, `WantedBy=default.target`, 기본 비활성).
- pastaq 요구 매핑(F17→QuickAccess, HOME짧게→QuickAccess2, HOME길게→Keyboard)으로 임시 교체해서
  (`/etc/inputplumber/capability_maps.d/zone_type1.yaml`, 매번 백업 후 복원) 데스크톱 + Gaming Mode
  둘 다 테스트:
  - **HOME 짧게(QuickAccess2)**: 두 세션 다 무반응. OpenGamepadUI 로그에도 버튼 입력 시점에 아무
    반응 없음.
  - **MORE/F17(QuickAccess)**: 무반응. 반면 **ZOTAC 버튼(안 바뀐 Guide)은 단독으로 Steam Gaming
    Mode 자체의 Quick Access Menu를 열었음** (OpenGamepadUI가 아니라 Steam 자체 동작으로 확인됨 —
    사용자가 직접 정정). Guide+South 합성(QuickAccess)이 오히려 방해했을 가능성.
  - **HOME 길게(Keyboard)**: 약 5초 눌렀을 때 시스템 전원 메뉴가 떴는데, chord의 실제 롱프레스
    임계값보다 훨씬 길어서 이 매핑과 무관한 시스템 레벨 동작으로 판단.
- **결론**: pastaq의 "OpenGamepadUI가 QuickAccess2를 가로챈다"는 주장은 기본 설정으로는
  재현되지 않음. PR 코멘트에 "OpenGamepadUI 자체에 추가로 켜야 하는 옵션이 있는지" 질문을 넣어둠.

### 2. 다이얼 capability_map — 벤더 드라이버 설치 대신 소스 코드 분석으로 대체 검증

사용자가 벤더 드라이버(OpenZotacZone/ZotacZone-Drivers) 실제 설치를 요청해서 백업까지 했으나,
소스 코드 분석만으로 충분한 근거가 나와서 실제 빌드/로드는 하지 않음. 백업 디렉터리
(`~/zotac-vendor-driver-test-backup-20260827-200406/`)는 세션 종료 전 삭제 완료.

**리포 구조**: `install_openzone_drivers.sh`가 HID 드라이버(`driver/hid/`)뿐 아니라 플랫폼
드라이버(RGB/팬, `driver/platform/`)와 다이얼용 별도 유저스페이스 파이썬 데몬
(`driver/dials/zotac_dial_daemon.py`, raw hidraw 직접 읽기 + uinput 합성, volume/brightness 액션)까지
한 번에 설치하는 풀 설치 스크립트 — 이번엔 이것 대신 `driver/hid`의 소스만 개별적으로 읽음.

**`zotac-zone-hid-core.c` 분석 결과**:
- 다이얼(report ID 3)은 실제로 별도 evdev 장치(`wheel_input`)로 분리됨 — 오른쪽 CW/CCW →
  `REL_WHEEL`, 왼쪽 CW/CCW → `REL_HWHEEL` (비트 정의가 우리 실측과 완전 일치).
- **그런데 터치패드용 `mouse_input` 장치도 자체 리포트의 `data[4]`에서 독자적으로 `REL_WHEEL`을
  냄** (다이얼과 무관한 터치패드 자체 스크롤). 즉 벤더 드라이버가 있어도 두 개의 서로 다른
  evdev 장치가 같은 capability(`Mouse::Wheel`)를 각각 낼 수 있음.
- 다이얼 전용 daemon(`zotac_dial_daemon.py`)은 커널 evdev를 안 쓰고 hidraw를 직접 읽음 — 향후
  다이얼 재구현 시(철회된 성능 이슈, "다음 세션 작업 2" 참고) 이벤트 기반 raw hidraw 읽기 참고 자료로
  유용할 수 있음.

**pastaq(#410)와의 배경**: 리뷰어가 이 기기 터치패드 관련 PR(#410, "Fix(Hardware Support):
Passthrough Zotac Zone Touchpads", 2025-08-02 머지)을 직접 작성한 이력이 있음 — 실질적 배경
지식 있는 리뷰어로 인지할 것. 다만 그 PR의 diff를 보면 당시 `ZOTAC Gaming Zone Dials`라는
커널 보고 이름을 사실상 "터치패드"로 이해하고 있었던 것으로 보이며, 우리 쪽 원래 `zone_type1.yaml`은
그 이름만 보고 "물리적 다이얼"로 잘못 가정하고 작성됐던 것으로 보임 — 이 네이밍 혼선이 논쟁의
뿌리일 가능성.

### 3. ★★ capability_map 아키텍처 오해 발견 및 정정 (중요, CLAUDE.md에 반영함)

`8d89ead` 커밋(위 "누락됐던 커밋" 참고)을 뒤늦게 발견해서 읽어보니, 이 세션 초반 내내(그리고
CLAUDE.md 자체도) "capability_map 번역은 소스와 무관하게 CompositeDevice 전체에 전역 적용된다"고
설명하고 있었는데, **이건 이 기기 설정에는 틀린 설명**이었음. 코드로 직접 재확인:

- `composite_device/mod.rs`의 `translatable_capabilities`/전역 `capability_map`(`load_capability_map`)은
  **CompositeDeviceConfig 최상위 `capability_map_id`**가 있을 때만 채워짐 — `50-zotac-zone.yaml`엔 그게 없음.
- 실제로 쓰이는 건 `source_devices[].capability_map_id` (`input/source/evdev.rs:121-126`) —
  소스 하나만의 전용 `EventTranslator`를 만듦 (`input/source/evdev/gamepad.rs:67`). 다른 소스와 무관.
- **원래 터치패드 회귀의 진짜 원인**: 터치패드(mouse) source 항목 자체가 다이얼 규칙이 들어있는
  `capability_map_id: zone1`을 갖고 있어서, **터치패드 자신의 translator가 자기 자신의 진짜 스크롤
  이벤트를 다이얼로 잘못 번역**한 것 — "전역 누출"이 아니라 "자기 자신에게 잘못된 맵이 붙어있었다"는
  훨씬 단순한 문제였음. `8d89ead`가 이미 정확한 원인으로 고쳐놓은 상태였음(별도 맵으로 분리 + 터치패드
  항목에서 capability_map_id 제거).
- pastaq의 v2 제안(line 64)이 여전히 위험한 이유도 재정리됨: "전역 누출"이 아니라 — 그 group의
  evdev name glob이 벤더 드라이버 없는 하드웨어에서 결국 터치패드 하나만 매칭하므로, **그 group
  자신의 translator**가 다시 자기 진짜 휠 이벤트를 다이얼로 잘못 번역할 것이라는, 훨씬 더 정확하고
  방어 가능한 논거.
- **CLAUDE.md의 Architecture 섹션(4번, capability_map)과 Hardware reference의 Dials 항목을 이
  세션에서 직접 정정함.** 두 곳 다 "소스별 vs 전역" 두 가지 경로를 구분해서 설명하도록 다시 씀.
  CLAUDE.md는 git에 트래킹되지 않는 로컬 전용 파일(`.git/info/exclude`)이라 커밋 불필요, 이미
  디스크에 반영됨.
- **주의**: 이 handoff.md의 08-25/08-26 섹션(위쪽)에는 정정 전의 틀린 설명이 여전히 남아있음 —
  다이얼/터치패드 관련해서 다시 설명할 일이 있으면 이 섹션(08-27)을 우선할 것.

### 4. PR 답변 게시 완료

한국어로 초안 작성 → 검토/수정 여러 차례 (동기 설명에 이슈 #655 링크 추가, "로컬 override로
계속 쓰겠다" 문장 삭제, "저희"→"제가"로 통일, "OpenGamepadUI's QAM"→"Steam Gaming Mode's own
Quick Access Menu"로 사실관계 정정 등) → 영어 번역 → **사용자가 직접 게시함** (2026-08-27T11:36:51Z,
https://github.com/ShadowBlip/InputPlumber/pull/664#issuecomment-5438401478). 내용: 동기 설명 +
HOME/QAM 실측 결과 3가지 + 다이얼 capability_map 정정된 근거 + 이견 없는 피드백(주석 정리, glob
매칭) 반영 완료 + 계속 막히면 이 부분을 빼거나 PR을 접겠다는 마무리.

**게시 직후 커밋 `9629ea6`**로 두 번째 리뷰의 주석 정리 요청(line 25/37/58 주석 블록 제거, line 85
`", real"` 접미사 제거)을 반영해서 push함 — `make test` 대신 `cargo test config::`(autostart-rules
포함) + `cargo clippy --all -- -D warnings`로 검증(YAML 주석/이름만 바뀐 변경이라 이 정도로 충분
판단).

**다음 세션에서 가장 먼저 할 일: pastaq가 이 08-27 코멘트에 응답했는지 확인.**

---

## 참고: 디버깅 기법

일반적인 디버깅 기법(hidraw 캡처, HID 리포트 디스크립터 파싱, capability_map chord 로깅 함정,
`pkill -f` 자기 셸 오살, 실기기 dev 바이너리 테스트 절차 등)은 전부 **CLAUDE.md "Debugging input
pipeline issues on real hardware" 섹션으로 이동함** — 거기가 정본.
