# InputPlumber Zotac Gaming Zone 버그 수정 - 작업 인계 문서

## 상태 요약 (최신, 2026-09-09)

**리부트 검증 완료 (TODO 1) — 리부트 후에도 컴포짓 1개, 소스 8개 전부 정상, 게임모드에서 사용자
확인 "문제 없음".** 노드 번호는 09-08 대비 전부 바뀌었지만(이름/`phys_path` 매칭이라 무관) 구성은
동일. 09-08의 중복 컴포짓/xpad 누락 문제는 재발하지 않았다. 상세는 "★ 2026-09-09 세션" §9 참고.


**다이얼 검증 완료 (TODO 2) — 다이얼은 볼륨(왼쪽)/화면 밝기(오른쪽)로 동작하고, `zone1`의
`Left/RightStickDial` 매핑은 런타임에서 전부 버려지는 no-op이다.** 원인은 타겟 라우팅이 capability
기준 필터링이고(`composite_device/targets.rs:303-312`) `xbox-elite`/`mouse`/`keyboard` 어느
타겟도 `Gamepad:Dial:*`을 선언하지 않기 때문. 하드웨어·벤더 드라이버·config는 전부 정상.
upstream 패키지 config도 같은 죽은 매핑을 갖고 있다. 상세는 "★ 2026-09-09 세션" 참고.
**리부트 검증(TODO 1)은 아직 미착수 — 다음 세션 최우선.**

## 상태 요약 (2026-09-08)

**★★★ 벤더 커널 드라이버 `zotac_zone_hid`가 드디어 OGC에 실려서 이 기기에 들어왔다 (Bazzite
44.20260907, 커널 7.2.3-ogc3.1, InputPlumber 0.78.0-5 → 0.79.0-4). 예고돼 있던 "실기기 재검증
세션"을 이날 진행했고, 컨트롤러를 정상 동작 상태로 되돌렸다.** 상세는 아래 "★ 2026-09-08 세션"
참고. 하드웨어 토폴로지·F16-F19 배치·진짜 게임패드 노드가 전부 바뀌었으므로, **이 문서와
CLAUDE.md에서 09-08 이전에 쓰인 하드웨어 관련 서술은 전부 `hid-generic` 시절 기준임에 유의**할 것
(CLAUDE.md는 09-08에 갱신 완료, 새 섹션 "After the vendor driver landed"가 정본).

**(구) 상태 요약 (2026-09-04)**

**★★ PR #664, #668 둘 다 머지 없이 CLOSED됨 (각각 09-02, 09-03) — pastaq가 주말 실기기 검증 후
"벤더 드라이버를 OGC 커널에 번들 탑재하는 방향으로 간다"고 결정, 이 두 PR의 config/hidraw
워크어라운드 자체가 채택 안 됨.** 상세는 아래 "★ 2026-09-04 세션" 참고. 이 리포의 로컬 워크어라운드
(`/etc` override + `zotac-zone-paddles` 스크립트)는 OGC가 실제로 벤더 드라이버를 번들할 때까지
**계속 유지해야 함** — CLAUDE.md의 "PR 머지되면 정리" 조건은 이제 성립하지 않으므로 그 섹션의
전제가 바뀌었음을 유의할 것 (아직 CLAUDE.md 자체는 미수정).

**(구, 09-04 이전 최신) PR #664: 2026-08-30에 pastaq의 매핑 제안(MORE→QuickAccess, HOME짧게→QuickAccess2,
HOME길게→Keyboard)을 실기기 TUI로 직접 테스트함 — pastaq의 예상과 반대로 Screenshot/QuickAccess는
그 매핑으로 바꿔도 여전히 TUI에 안 나타남, 코드 근거(pastaq 본인이 링크한 코드)도 우리 쪽 분석을
뒷받침함. 상세는 "★ 2026-08-30 세션" 참고. pastaq의 주말 실기기 검증 결과는 아직 안 옴.**

**PR #664: pastaq가 08-28T14:04:16Z에 답변함 (issuecomment-5453477541) — 리뷰가 코멘트 공방에서
"pastaq 본인이 이번 주말 실기기로 직접 검증"하는 국면으로 전환됨.** Screenshot/QuickAccess 매핑
컨벤션에 대해서는 여전히 이견 존재(pastaq는 본인 안 MORE→QuickAccess / HOME짧게→QuickAccess2 /
HOME길게→Keyboard를 재주장). 5초 파워메뉴 설명과 터치패드 entry 제거는 반박 없이 수용된 것으로
보임(다만 터치패드는 명시적 OK는 아직 없음 — 계속 미커밋 유지). 상세는 아래 "★ 2026-08-29 세션"
참고. **다음 세션은 pastaq의 주말 실기기 검증 결과(추가 코멘트)가 왔는지 확인하는 것부터 시작할 것.**

**⚠️ 지금 `/etc/inputplumber/devices.d/50-zotac-zone.yaml`은 터치패드(`group: mouse`) entry가
제거된 상태로 배포돼 있음.** 리포 작업트리도 같은 상태지만 **커밋은 안 함** — pastaq 확인 후
커밋하기로 사용자가 결정. 즉 이 시점의 배포 config는 PR #664 브랜치의 커밋된 내용과 다름.

**PR #668: 08-28 확인 시점에도 여전히 OPEN, 미병합.** CI 3종 전부 SUCCESS,
`mergeStateStatus: CLEAN`(08-27 시점의 UNSTABLE에서 해소됨). pastaq의 "Reminder for me to squash
merge this PR"(08-27 12:05 UTC) 이후 PR에 아무 활동 없음 — 막힌 게 아니라 리뷰어가 아직 실행을
안 한 것. 우리 쪽 액션 없음, 병합 여부만 확인하면 됨.

**(구) PR #664: pastaq가 08-27 12:01 UTC에 답변함 — PR을 계속 진행하기로 함, 접을 필요 없음.**
아래 "★ 2026-08-27 오후: pastaq 답변 및 다음 액션" 섹션 참고. (08-28에 실기기 테스트 3건 완료함)

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

### -1. ~~pastaq 08-27 답변에 대한 실기기 검증 3건~~ → **2026-08-28에 3건 전부 완료, 답변 게시함.**
결과는 "★ 2026-08-28 세션" 섹션이 정본. 아래 내용은 당시 계획/배경 기록으로만 남김.

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

## ★ 2026-08-28 세션 — pastaq 요청 검증 3건 완료 및 답변 게시

### 0. 결과 요약

pastaq의 08-27 12:01 코멘트가 요청한 3가지를 전부 실기기에서 검증하고, 영어 코멘트를 작성해
사용자가 직접 게시함 (2026-08-28T12:54:36Z, issuecomment-5452750998). 이후 마크다운 서식이
깨진 걸 발견해 코멘트를 수정함(내용 동일, 서식만 복원).

### 1. `inputplumber device 0 test` — 전체 버튼 스윕

- **결과**: South/East/North/West(ABXY), D-pad 4방향, LB/RB, L3/R3, Select/Start, ZOTAC버튼(Guide),
  HOME 길게(Guide) — **전부 정상 반응**.
- **`Screenshot`/`QuickAccess`는 이 도구로 확인 불가**(박스 자체가 안 뜸). 코드로 확인한 이유:
  TUI의 Buttons 패널은 `composite_device::get_capabilities()`(= 각 소스의 **원시 선언
  capability** 합집합)로 채워지는데, `KeyboardEventDevice::get_capabilities()`
  (`src/input/source/evdev/keyboard.rs:143`)는 `device.supported_events()`/`supported_keys()`만
  순회해 `EvdevEvent::as_capability()`로 변환할 뿐 **translator를 전혀 참조하지 않음**.
  `SourceDriver::get_capabilities()`(`source/mod.rs:326`)의 event filter도 빼기만 하고 더하지
  않음. 즉 chord 번역 **결과물로만** 존재하는 capability는 구조상 이 패널에 나올 수 없음.
- `device test`가 로드하는 `profiles/debug.yaml`은 `mapping: []` 빈 프로파일이라 리매핑 원인 아님
  (혹시나 해서 확인함).
- **간접 증거**: HOME 길게가 `Guide` 박스를 반응시킴 — 짧게 매핑(`Screenshot`)과 **동일한 chord
  번역 파이프라인**이고 다만 원시 capability이기도 한 값으로 귀결된 것.

### 2. 5초 파워메뉴 — LEFTMETA 유출설 반증

- `evtest`(키 식별 가능) 캡처: 누르는 내내 `KEY_LEFTCTRL`+`KEY_LEFTALT`+`KEY_KPDOT`만
  (KPDOT auto-repeat), **`KEY_LEFTMETA`는 단 한 번도 없음**.
- ⚠️ **`libinput debug-events`는 키코드를 `***`로 마스킹하므로 키 식별 증거가 될 수 없음.**
  개수/타이밍(7.905초간 키 3개)만 뒷받침. 처음 초안에 "두 캡처 모두에서 LEFTMETA 없음"이라고
  썼다가 게시 전 검증에서 잡아 수정함 — pastaq가 직접 추천한 도구라 이 부정확은 위험했음.
- **분리 실험**: InputPlumber 꺼짐 → 파워메뉴 안 뜸(2회 확인) / InputPlumber 켜짐(`HOME 길게 →
  Guide`) → 파워메뉴 뜸(**1회만** 확인). 초안의 "reliably appears"는 과장이라 "observed once"로 수정.
- **steamos-manager 인과관계는 미확정**: DBus 연동 사실(`ResetInterceptModes` 호출, 자체 가상
  입력 장치 등록)은 확인했으나, 파워메뉴를 띄운 주체라는 **직접 로그 증거는 못 찾음**
  (해당 시간대 `journalctl -u steamos-manager` → "No entries"). 코멘트에도 추정임을 명시함.
- 결론: raw 키가 아니라 **가상 Guide 버튼 홀드**가 트리거. 매핑 버그가 아니라 의도대로 동작한 것.

### 3. 터치패드 entry 제거 — 안전함 확인 (커밋은 보류)

- entry 제거(native/ungrabbed) 상태에서 데스크톱 + **Gaming Mode 양쪽** 검증: 오른쪽 패드 포인터
  정상, 왼쪽 패드 세로 스크롤 정상, Gaming Mode 기존 버튼 전부 회귀 없음.
- **이 터치패드는 `REL_WHEEL`만 보냄** — `evtest` 2회(가로로만 문지르기 / 좌↔우 반복) 모두
  `REL_HWHEEL` 전무. 추가로 `HIDIOCGRDESC`로 **HID 리포트 디스크립터를 직접 파싱**해서 하드웨어
  레벨로 확정: rid=4는 `size=8 count=3`(정확히 3바이트) = `X`/`Y`/`Wheel`뿐이라 4번째 축이 들어갈
  자리가 없고, "AC Pan"(Consumer `0x238`)은 디스크립터 전체 어디에도 없음. 즉 hid-generic/OS의
  해석 한계가 아니라 **펌웨어가 애초에 가로 스크롤 데이터를 안 보냄**.
  - 이 파싱에 쓴 스크립트를 새로 작성함: `~/zotac-zone-tools/rdesc_usage.py` (기존 `rdesc.py`는
    usage를 안 보여주고 리포트 크기만 출력함). ⚠️ 이 스크립트의 usage_page 라벨은 부정확함
    (rid=3을 "Generic Desktop"으로 표시하지만 실제 usage 값들은 Consumer 페이지) — usage 목록
    자체는 전부 열거되므로 "0x238 없음" 결론에는 영향 없으나, 페이지 이름은 신뢰하지 말 것.
- **커밋 안 함**: pastaq가 먼저 제거를 제안했지만, 원 작성자(#410)의 확인을 받고 커밋하기로
  사용자가 결정. 작업트리와 `/etc` 배포본에만 반영된 상태.

### 4. 이번 세션에서 새로 밝혀진 함정/사실 (다음 세션 필독)

- **`inputplumber device N test`를 `~/InputPlumber` 안에서 실행하면 실패한다**:
  `Error: Failed("service encountered an error processing the request: Could not read: No such file
  or directory (os error 2)")`. 원인은 `config::path::get_profiles_path()`가 상대경로
  `./rootfs/usr/share/inputplumber/profiles`를 먼저 반환하고, CLI가 그 문자열을 그대로 데몬에
  넘기는데 **데몬의 cwd가 달라서** 못 읽는 것. **리포 밖(`cd ~`)에서 실행할 것.**
- **게임패드 소스(event10)가 죽은 채 복구가 안 되는 현상을 실제로 겪음.** USB 재열거
  (`inotify DELETE`→`CREATE`) 직후 InputPlumber가 다시 열다가
  `Failed to fetch events: Os { code: 19 ... "No such device" }`(ENODEV) → `Detected source device
  stopped` 후 재시도 없음. 그 결과 `SourceDevicePaths`에 event10이 빠져서 **`device test`에서
  게임패드 버튼이 하나도 안 눌리는** 상태가 됨(처음엔 매핑 문제로 오인함). `systemctl restart
  inputplumber`로 복구. **upstream 이슈감** — 증상이 "컨트롤러가 갑자기 안 됨"으로 나타남.
- **`device test`에서 X→`North`, Y→`West`로 보이는 건 정상이다.** 리눅스 `BTN_NORTH`/`BTN_WEST`
  관례가 직관과 반대(North=Xbox X, West=Xbox Y). `capability.rs`의 주석
  (`North action, ... Xbox X`)이 맞는 설명임. 세션 중 이걸 버그로 오판하고 raw evtest까지 떠서
  "커널이 뒤바꿔 보고한다"고 결론냈다가, **사용자가 "Eden 등 에뮬에서도 정상"이라고 반증해서
  철회함** — Steam/Eden 둘 다 InputPlumber의 가상 출력을 읽으므로 둘 다 정상이면 출력이 맞는 것.
- **node 번호가 또 바뀌었다**: 게임패드가 `event14` → **`event10`**. 이름/`phys_path`로 찾을 것.
- **`~/zotac-zone-tools/getcmd.py`의 `GET_DEVICE_INFO` 오프셋 버그를 고침** (`resp[5:]` →
  `resp[6:]`). 고치기 전엔 `vid:pid e900:901e`, `fw 0.1.3` 같은 한 바이트 밀린 쓰레기값이 나왔음.
  수정 후 `vid:pid = 1ee9:1590`(오프셋이 맞다는 결정적 검증), **`fw 1.3.9`, `hw 1.5.0`** 확인.
  (`led zones = 0`은 여전히 이상하지만 이 필드만 위치가 다를 수 있음 — 미조사.)
- **"스크롤이 옆으로/대각선으로 간다"는 느낌의 정체가 밝혀짐 — InputPlumber와 무관.**
  `group: mouse` entry를 **완전히 제거한**(InputPlumber가 경로에 아예 없는) 상태에서도 같은
  현상이 그대로 남음. 진짜 원인은 패드 자체: `REL_WHEEL`만 내는데, 스트립의 축과 **다른 방향으로**
  문지르면 아무것도 안 내는 게 아니라 세로 틱을 **위/아래 무작위로 섞어서** 냄(raw 캡처에서
  `+1`/`-1`이 번갈아 찍히는 것으로 확인). 이 널뛰는 세로 스크롤이 "방향이 이상하다"로 체감된 것.
  사용자가 apexcheck.com의 horizontal-scroll-check 페이지로도 가로 신호가 없음을 별도 확인함.
  ⚠️ **세션 중 내가 "블루투스 마우스가 혼입된 관찰이라 신뢰 불가"라고 잘못 기록했다가 사용자가
  정정함 — 터치패드 테스트 중 마우스는 움직이지 않았다.** (그 마우스도 가로 스크롤이 이상하다는
  건 사실이나 Windows에선 정상이며, 이 건과는 무관한 별개 이슈다.)
  따라서 코드에서 찾은 InputPlumber Wheel 복제 버그(`REL_WHEEL`/`REL_HWHEEL`이 같은
  `Mouse::Wheel`로 합쳐지고 출력 시 양쪽 코드에 같은 값이 써지는 것 — CLAUDE.md에 기록)는
  **코드상 실재하지만 이 현상의 원인은 아니다.** 그래서 PR 코멘트 초안에서 해당 문단을 **삭제함**.
  upstream 이슈로 낼 때는 코드 근거로만 쓸 것.
- **CONTRIBUTING.md의 AI 조항 범위를 정확히 확인함** (세션 중 내가 잘못 안내했다가 사용자가
  지적해서 원문 확인): 고지 요구는 **커밋 메시지/코드에 한정**(34-49행, `Co-developed-by:`
  트레일러). PR 코멘트에는 적용 안 됨. 다만 **54행에 별개 조항**: *"Using AI to respond to human
  reviewers is strictly prohibited."* — 이건 고지로 면제되는 게 아니라 조건 없는 금지.
  이번 코멘트에는 이전과 달리 AI 번역 고지 문구를 **넣지 않기로 사용자가 결정**함.
- **스크린샷은 첨부하지 않기로 함**: `Screenshot_20260828_174352.png`는 터미널 투명도 때문에 TUI
  뒤로 이 세션의 한국어 대화(테스트 지시 내용 포함)가 읽힐 정도로 비쳐 보임. pastaq가 AI PR에
  민감하다고 밝힌 직후라 부적절 판단. 필요하면 투명도 없는 깨끗한 화면으로 새로 캡처할 것.
- **터미널에서 복사해 붙여넣으면 마크다운이 깨진다**: 렌더링된 상태(굵게/백틱 제거, 줄머리 2칸
  들여쓰기, 문단 중간 강제 줄바꿈)로 붙여짐. 코멘트 Edit으로 원본 마크다운을 다시 붙여 수정함.
  다음에도 게시용 텍스트는 **파일로 만들어서 `cat`으로 복사**할 것.

### 5. 다음 세션 TODO

1. ~~pastaq가 08-28 코멘트(issuecomment-5452750998)에 답했는지 확인.~~ → **2026-08-29에 확인함,
   답변 있었음.** 상세는 아래 "★ 2026-08-29 세션" 참고.
2. **PR #668 병합됐는지 확인** (`gh pr view 668 --repo ShadowBlip/InputPlumber --json state,mergedAt`).
3. pastaq가 터치패드 entry 제거에 확인을 주면 **커밋** (지금은 작업트리/`/etc`에만 반영, 미커밋).
4. **미착수로 남아있는 것**: PR 코드 라인별 설명 듣기(아래 "다음 세션 작업 0" 참고) — 사용자가
   여러 세션에 걸쳐 요청했으나 아직 진행 안 됨.
5. upstream 이슈 후보 3건: Wheel 복제 버그, event10 ENODEV 후 미복구, `devices` vs `devices.d`
   네이밍 트랩.

### 6. 미커밋 로컬 변경사항 (세션 종료 시점)

- `CLAUDE.md` — event4의 `REL_HWHEEL` 오기 정정(+ 오늘 확인 근거), node 번호 불안정성 경고 추가,
  Wheel 복제 버그 문단 추가, Dials 항목의 관련 문장 정정.
- `rootfs/usr/share/inputplumber/devices/50-zotac-zone.yaml` — 터치패드 `group: mouse` entry 주석
  처리(제거). **pastaq 확인 전까지 커밋 보류.**
- 리포 밖: `~/zotac-zone-tools/getcmd.py` 오프셋 수정, `~/zotac-zone-tools/rdesc_usage.py` 신규.

## ★ 2026-08-29 세션 — pastaq의 08-28 답변 확인 (GitHub API로 조회, 코드 변경 없음)

### 결과 요약

`gh` CLI가 이 toolbox/host 양쪽 다 설치돼 있지 않아서(둘 다 `command not found`), GitHub REST API를
`curl`로 직접 조회해서 확인함. pastaq가 2026-08-28T14:04:16Z에 답변함
(https://github.com/ShadowBlip/InputPlumber/pull/664#issuecomment-5453477541). PR 상태는 여전히
`state: open`, `mergeable_state: clean`.

답변은 우리 08-28 코멘트 원문을 블록쿼트로 인용하면서 그 사이사이에 pastaq 자신의 코멘트를 삽입하는
형식. 인용된 부분(별다른 반박 문구 없이 그대로 인용만 된 부분)은 사실상 이견 없이 수용한 것으로
해석함.

**1) `inputplumber device 0 test`에서 Screenshot/QuickAccess가 안 보이는 것 — pastaq는 납득하지 않음.**
- "capability map의 모든 항목이 composite device의 capabilities HashSet에 들어간다"며
  `composite_device/mod.rs` 233-256행(당시 upstream `main` 기준 `23f84b7`)을 직접 링크. "본인은
  매일 이렇게 config를 검증하는데 이렇게 동작 안 한다면 뭔가 이상한 것"이라는 취지로 우리 쪽 코드
  분석(chord 번역 결과물 capability는 TUI 패널에 구조적으로 안 뜬다는 설명, 위 "★ 2026-08-28 세션"
  1번 참고)에 의구심을 표함. 이 부분은 **재반박 여부를 검토할 여지가 있음** — 다만 서두르지 않기로
  함(아래 "다음 액션" 참고).
- **매핑 컨벤션 재주장**: MORE(F17)→`QuickAccess`, HOME 짧게→`QuickAccess2`, HOME 길게→`Keyboard`
  (자신이 원래 제안했던 안 그대로, 우리가 채택한 `Screenshot`/`Guide`가 아님). 근거로 "다른 모든
  기기에서 Guide/QuickAccess/QuickAccess2/Keyboard의 배치 컨벤션을 일관되게 유지한다"는 자신의
  일반 원칙을 재설명함(왼쪽=Guide, 오른쪽 첫 여분 버튼=QuickAccess, 추가 여분 버튼=QuickAccess2 →
  OpenGamepadUI 미실행 시 Screenshot / 실행 시 OGUI Quick Bar로 동작, 네 번째 버튼이 있으면
  Keyboard). "당신이 지금 매핑 대신 내 제안대로 맞추면 테스터에서 더 잘 나올 것"이라는 뉘앙스의
  문장도 있음 — 즉 **현재 매핑 자체에 뭔가 문제가 있다는 암시**를 하고 있음.
- **★ 가장 중요한 신규 사실: pastaq가 이번 주말(다음 주말, 정확한 날짜 미명시) 직접 이 브랜치를
  pull해서 본인 소유 Zotac Zone 실기기로 검증하겠다고 선언함.** 벤더 드라이버 없는 OGC 커널과
  벤더 드라이버 있는 Valve 커널 둘 다 테스트할 예정이라고 명시.

**2) 5초 파워메뉴 — 반박 없음.** 우리 설명(가상 Guide 버튼 홀드 트리거로 추정, `KEY_LEFTMETA`/
`powerbuttond`와 무관함을 실기기로 반증)을 그대로 인용만 하고 이견 제시 안 함. 사실상 종결로 판단.

**3) 터치패드 `source_devices` entry 제거 — 반박 없음, 그러나 명시적 승인 문구도 없음.** 우리가
쓴 원문("당신(#410 원저자)이 확인해주면 커밋하겠다")을 그대로 인용만 함. **"가서 커밋해라"는 명확한
문장은 없으므로, 계속 미커밋 상태 유지가 안전하다고 판단** — pastaq 본인이 이번 주말 검증에 이
부분도 포함시킬 가능성이 높아 보임.

### 다음 액션 (판단, 사용자 액션 필요 — CONTRIBUTING.md상 리뷰어 답변은 AI 작성 금지)

- **지금 당장 답변할 필요는 없어 보임** — pastaq가 본인 실기기로 직접 검증하겠다고 이미 선언했으므로,
  그 결과를 기다리는 게 합리적. 짧은 인사성 답글(대기하겠다는 정도) 정도는 선택 사항.
- **Screenshot/Guide vs pastaq의 QuickAccess/QuickAccess2/Keyboard 컨벤션 — 순수 사용자 판단 필요.**
  기술적 근거 대립이 아니라 UX 취향/컨벤션 일관성 문제라 코드로 결론 낼 수 없음. pastaq가 주말에
  직접 테스트해볼 것이므로 지금 미리 코드를 바꿀 필요는 없어 보임.
- **터치패드 entry 커밋은 계속 보류.**
- **PR #668 병합 여부는 이번 세션에서 미확인** — 다음 세션에서 `gh` 대신 GitHub REST API
  (`curl -s "https://api.github.com/repos/ShadowBlip/InputPlumber/pulls/668"`)로 확인할 것.
  참고: 이 환경엔 `gh` CLI가 toolbox/host 어디에도 없음 — `curl` + GitHub REST API가 유일한
  조회 수단(비인증 상태라 public repo 읽기 전용, rate limit 있음에 유의).

## ★ 2026-08-30 세션 — pastaq 제안 매핑을 실기기에서 직접 테스트

### 배경

08-28 pastaq 코멘트의 1번 항목("TUI에서 Screenshot/QuickAccess가 안 보이는 건 이상하다, 내 제안
컨벤션(MORE→QuickAccess, HOME짧게→QuickAccess2, HOME길게→Keyboard)으로 맞추면 tester에서 결과가
더 잘 나올 것")을 실기기로 직접 검증함. 추가로 pastaq가 반박 근거로 링크한
`composite_device/mod.rs` L233-256(커밋 `23f84b7`)을 코드로 직접 읽어봤는데, 그 블록은
**최상위 `capability_map_id`(`device.capability_map`)가 있을 때만** 도는 경로(`manager.rs:583`의
`config.capability_map_id`로 세팅됨)이고, `50-zotac-zone.yaml`엔 최상위 `capability_map_id`가
없음(`source_devices[]` 안에만 있음) — 즉 **pastaq가 링크한 코드 자체가 오히려 우리 쪽 분석(소스별
translator만 쓰이고 device-wide 경로는 이 기기에서 아예 안 돈다)을 뒷받침함.** 이걸 실기기 실측으로도
확인하기 위해 테스트 진행.

### 방법

`zone_type1.yaml`을 pastaq 컨벤션대로 임시 수정한 테스트 버전 작성(저장소는 건드리지 않고
scratchpad에 별도 파일로): MORE(F17) `Guide→QuickAccess`, HOME짧게(real chord)
`Screenshot→QuickAccess2`, HOME길게(real chord) `Guide→Keyboard`. ZOTAC(F16)→Guide와 패들 매핑은
그대로 둠. `/etc/inputplumber/capability_maps.d/zone_type1.yaml`을 백업 후 이 테스트 버전으로 교체,
`systemctl restart inputplumber` 후 리포 밖(`~`)에서 `inputplumber device N test` TUI로 확인.
**저장소(`rootfs/.../devices/50-zotac-zone.yaml`의 터치패드 entry 제거 미커밋 변경 등)는 이 작업
중 전혀 건드리지 않음.**

### 결과 — pastaq의 예상과 반대로 나옴

- **Screenshot/QAM(QuickAccess, QuickAccess2) 박스: pastaq 컨벤션으로 바꿔도 TUI에 여전히 전혀 안
  나타남**(박스 자체가 안 생김). pastaq의 "내 config로 맞추면 tester에서 더 잘 나올 것"이라는 예상이
  **실측으로 반증됨** — 08-28에 확인한 "Screenshot/QuickAccess는 이 패널 구조상 절대 안 뜬다"는
  결론이 매핑을 바꿔도 그대로 유지됨.
- **Guide 박스: 존재는 하되(다른 소스가 별도로 raw declare하는 것으로 추정) 눌러도 무반응.** 버그가
  아니라 예상된 결과 — 이 테스트 config에서 Guide로 가는 실제 물리 트리거(F17/HOME길게)를 전부
  QuickAccess/Keyboard로 옮겨버렸고, Guide엔 F16(실제로 신호를 안 보내는 것으로 보이는 죽은 소스
  이벤트)만 남아서 눌러도 반응이 없는 게 당연함.
- **나머지 버튼(ABXY/D패드/숄더/트리거/스틱/패들 등): 전부 정상, 회귀 없음.**

### 원상복구 및 검증

`/etc` 백업본으로 복구 후 `systemctl restart inputplumber`, 이어서 배포본과 저장소 원본
(`rootfs/usr/share/inputplumber/capability_maps/zone_type1.yaml`)이 diff 없이 완전히 일치함을
확인함. 백업 파일도 정리 완료. **git 변경사항 없음** — 이 세션은 `/etc` 배포본만 임시로 건드렸다가
원복한 것이라 커밋할 것도 없음(기존 미커밋 변경 3건은 그대로 유지: `CLAUDE.md`, `handoff.md`,
`50-zotac-zone.yaml`의 터치패드 entry 제거).

### 의미

pastaq의 1번 항목 반박에 쓸 수 있는 근거가 두 겹으로 갖춰짐: (a) pastaq 본인이 링크한 코드가
사실은 우리 쪽 분석(device-wide 경로는 이 기기에서 안 쓰인다)을 뒷받침한다는 코드 근거, (b) 그
코드 분석을 실기기로 직접 검증해서 pastaq의 매핑으로 바꿔도 여전히 안 뜬다는 실측 근거. 다만
**답변 게시는 여전히 사용자 직접 작성 필요**(CONTRIBUTING.md), 그리고 pastaq가 이번 주말 본인
실기기로 검증하겠다고 예고한 상태라 그 결과를 먼저 볼지, 이 근거로 먼저 재반박할지는 사용자 판단
필요.

### 다음 세션 TODO (갱신)

1. pastaq의 주말 검증 결과(추가 코멘트) 왔는지 확인.
2. 이번 세션에서 확보한 반박 근거(코드+실측)를 코멘트에 반영할지 사용자와 상의.
3. PR #668 병합 여부 확인 (`curl -s "https://api.github.com/repos/ShadowBlip/InputPlumber/pulls/668"`).
4. 터치패드 entry는 pastaq 확인 오면 커밋.
5. 여전히 미착수: PR 코드 라인별 설명 듣기.

## ★ 2026-09-04 세션 — PR #664, #668 둘 다 CLOSED 확인 (GitHub API 조회, 코드 변경 없음)

### 0. 결과 요약

이 세션에서 `gh` CLI가 이 toolbox에 새로 설치돼 있는 걸 확인함(08-29 세션 때는 없었음). `gh`와
GitHub API(`curl`)로 PR #664, #668 상태를 조회한 결과 **둘 다 머지되지 않고 CLOSED됨**을 확인—
직전 세션(08-30)까지는 둘 다 OPEN이었으므로 이 세션에서 처음 발견한 사실.

### 1. PR #664 — 09-02T22:37:52Z CLOSED (pastaq)

09-02에 pastaq가 인라인 리뷰 코멘트 2건을 남기고 곧바로 PR을 닫음:

- `rootfs/usr/share/inputplumber/devices/50-zotac-zone.yaml:59` — "This matches on multiple
  devices now, add unique: false" (터치패드 관련 entry가 여러 장치에 매칭되는 문제 지적).
- `rootfs/usr/share/inputplumber/devices/50-zotac-zone.yaml:35`(`phys_path: "*/input1"`) —
  **"This breaks the config when the driver is present"**. 벤더 드라이버(`zotac_zone_hid`)가 로드된
  Valve 커널에서 실제로 캡처한 `/proc/bus/input/devices` 항목을 첨부:
  ```
  N: Name="ZOTAC Gaming Zone Gamepad"
  P: Phys=usb-0000:c4:00.3-4/input3
  H: Handlers=kbd event11 js0
  ```
  즉 벤더 드라이버가 있으면 게임패드가 `input1`이 아니라 **`input3`**(interface 3)으로 잡힘 —
  `phys_path: "{*/input[1|3]}"`로 고쳐야 벤더 드라이버 유무 양쪽에서 다 맞는다는 지적.
- **클로징 코멘트(09-02T22:37:52Z)**: *"After extensive evaluation with the zotac_zone_hid driver
  blacklisted I can confidently state that this is not a viable configuration. The gamepad does
  not enumerate without the driver present at all without a udev rule for xpad."*
  — 벤더 드라이버를 블랙리스트한 조건(이 사용자 기기와 동일한, 벤더 드라이버 없는 OGC 커널 조건)에서
  **게임패드 자체가 xpad용 udev 규칙 없이는 열거되지 않는다**는 걸 pastaq 본인 실기기로 확인했다는
  결론. 이 사용자 실기기(kernel xpad가 `event14`/`event10`로 정상 노출됨, 이 리포 전체에 걸쳐 검증
  완료)와는 다른 결과인데, 정확히 어떤 차이(예: udev 규칙 유무, 커널 버전, xpad 모듈 자동로드 설정
  등) 때문인지는 이 코멘트만으로는 특정 안 됨 — **원인 미상, 다음 세션 조사 후보.**

리뷰 이력 전체(`gh pr view 664 --json reviews`)도 재확인: CHANGES_REQUESTED가 08-25, 08-26 두
차례였고, 09-02 리뷰 2건은 모두 `COMMENTED`(인라인 코멘트 형태) 상태로 남음 — 별도의 최종
CHANGES_REQUESTED/REJECTED 리뷰 없이 코멘트 직후 바로 PR을 닫은 것으로 확인됨.

### 2. PR #668 — 09-03T01:53:24Z CLOSED (pastaq)

08-27 APPROVED + "squash merge하겠다" 예고 이후 아무 활동이 없다가, 09-03에 다음 코멘트만 남기고
닫힘: *"We're going to ship the driver in OGC as this isn't a viable path forward."*

패들 hidraw 프로토콜 구현(리뷰 반영 완료, CI 전부 SUCCESS 상태였음)이 코드 결함 때문이 아니라
**전략 변경**(config/hidraw 워크어라운드 대신 벤더 커널 드라이버를 OGC에 번들)으로 폐기된 것으로
읽힘. 인라인 코드 리뷰 코멘트는 이 클로징 코멘트 외에 추가로 없음.

### 3. 종합 — 두 PR이 같은 결정으로 묶여서 닫힘

두 코멘트를 합쳐 읽으면 pastaq의 결론은: *"config/hidraw 레벨 워크어라운드로는 이 기기(벤더 드라이버
없는 조건)를 완전히 커버할 수 없다(게임패드 열거 자체가 안 됨) → 근본 해결은 벤더 커널 드라이버
(`zotac_zone_hid`)를 OGC(Universal Blue/Bazzite 커널)에 번들 탑재하는 것 → 그러면 다이얼/패들/
게임패드 열거가 전부 커널 레벨에서 정식으로 해결되므로 이 두 PR의 접근 자체가 불필요해진다."*
라는 방향 전환으로 보임. **"언제" OGC가 벤더 드라이버를 실제로 번들할지는 이 코멘트들만으로는
알 수 없음 — 추적 필요.**

### 4. 이 리포 로컬 워크어라운드에 미치는 영향

CLAUDE.md "This machine" 섹션의 "PR이 머지되고 이미지에 반영되면 워크어라운드 정리" 전제가 더 이상
성립하지 않음. `/etc/inputplumber/devices.d/50-zotac-zone.yaml`, `capability_maps.d/zone_type1.yaml`
override와 `~/zotac-zone-tools/zotac-zone-paddles` 수동 실행 워크어라운드는 **OGC가 실제로 벤더
드라이버를 번들할 때까지 무기한 유지**해야 함.

**★ CLAUDE.md는 이후 같은 세션에서 갱신 완료함** ("PR이 병합되면" → "OGC가 벤더 드라이버를 번들하면"
전제로 교체, `grep -c "phys_path" ...` 체크 제거 — 더 이상 유효한 신호가 아님). 추가로 사용자 질문
("드라이버가 통합되면 지금까지 한 작업은 되돌려야 하나?")에 답하며 정리한 **3단계 분류를
CLAUDE.md "After OGC ships the vendor driver" 섹션에 반영함**:

1. **벤더 드라이버가 대체하는 hidraw/유저스페이스 우회** (`zotac-zone-paddles` 스크립트, 철회된
   다이얼 폴링 구현) — 되돌릴 필요 없이 **자연 폐기**. 패들은 upstream 코드의 기존 sysfs 경로
   (`configure_via_sysfs()`)가 그때부터 작동하고, 다이얼은 벤더 드라이버가 노출하는 별도 evdev
   장치(`wheel_input`)에 이미 만들어둔 `zone_type1_dial.yaml`(`zone1_dial`)을 그냥 붙이면 됨.
2. **벤더 드라이버와 무관한 InputPlumber config 매칭 수정** (`c00deba` evdev name, `193328a` 중복
   composite device, `399510c` F17/F18 스왑, 터치패드 entry 제거, HOME chord 매핑) — **무작정
   되돌리지 말고 실기기 재검증 후 필드별로만 조정**. 패키지 config는 PR 미병합으로 여전히 원래
   버그가 있는 상태이지만, 벤더 드라이버가 로드되면 커널이 보고하는 토폴로지 자체가 바뀔 수 있어서
   (예: `phys_path`가 `input1`→`input3`로 바뀐다는 pastaq의 09-02 리뷰 보고, 아직 이 기기에서
   미검증) 그대로 다 맞는다는 보장도 없음.
3. **CLAUDE.md/handoff.md 자체** — 되돌릴 대상 아님, 드라이버가 실제로 이 기기에 들어오면 그 시점에
   새 세션 기록을 추가.

**결론: "전부 되돌린다"는 개념 자체가 안 맞고, 드라이버가 실제로 이 기기에 도착하는 시점에 실기기
재검증 세션이 한 번 더 필요하다**는 게 확정된 방침. 상세 문구는 CLAUDE.md가 정본.

두 PR의 브랜치/커밋(다이얼 프로토콜 확정 내용, 패들 hidraw `PackedStruct` 구현 등)는 upstream
제출용으로는 사실상 종료됐지만, 로컬 참고 자료로는 여전히 유효함.

### 5. 리뷰어 응답 관련

CONTRIBUTING.md상 리뷰어에게 AI로 답변하는 것은 금지(고지 여부와 무관하게 조건 없는 금지)이므로,
이 결과에 어떻게 대응할지(PR을 닫힌 채로 둘지, "OGC 드라이버가 언제쯤 반영되는지" 정도만 물을지 등)는
**사용자가 직접 판단·작성**해야 함 — 이 세션에서는 조회만 하고 어떤 코멘트도 게시하지 않음.

### 6. 다음 세션 TODO (갱신, 이전 항목 대체)

1. ~~CLAUDE.md의 "This machine" 섹션을 이 결과에 맞춰 갱신~~ → **이번 세션에서 완료함**(머지 전제를
   "OGC 벤더 드라이버 번들" 전제로 교체 + "되돌릴 필요 없는 것 vs 재검증 필요한 것" 3단계 분류 반영).
2. pastaq가 언급한 "게임패드가 udev 규칙 없이 열거 안 됨" 현상이 이 사용자 실기기와 왜 다른지
   원인 조사(선택 사항 — upstream 제출 계획이 없어졌으므로 우선순위 낮음, 다만 이 기기 자체의
   향후 안정성엔 참고 가치 있음).
3. OGC(Universal Blue/Bazzite 커널)가 `zotac_zone_hid` 벤더 드라이버를 실제로 번들했는지 추적할
   방법 마련 (예: 커널 패키지 changelog, ublue-os 관련 리포 이슈/PR 검색 — 아직 미착수).
4. 터치패드 entry 제거, HOME/QAM 매핑 등 PR #664에서 다투던 개별 이슈들은 이제 upstream 제출
   목적이 사라졌으므로, 로컬 `/etc` override 전용으로 계속 쓸지 사용자와 확인 필요.
5. (이전부터 미착수) PR 코드 라인별 설명 듣기 — upstream 제출 계획은 종료됐지만 사용자가 요청한
   학습 목적 자체는 여전히 유효할 수 있음, 필요 여부 사용자 확인.

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

### 9. 리부트 검증 (TODO 1) — 완료, 정상

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

### 10. 다음 세션 TODO (09-08 목록에서 갱신)

1. ~~리부트 후 컴포짓 1개 + 전 버튼 정상인지 확인~~ → **완료(이 세션). 정상.**
2. ~~다이얼 실동작 확인~~ → **완료(이 세션).** 볼륨/밝기로 동작, capability 매핑은 no-op.
3. ~~`~/zotac-zone-tools/zotac-zone-paddles` 정리~~ → **완료(이 세션).**
   `~/zotac-zone-tools/obsolete/`로 이동(삭제 아님, 벤더 프로토콜 CRC/프레임 구현이 참고 가치가
   있어서) + 이유를 적은 README 동봉. CLAUDE.md의 두 군데 경로 서술도 갱신함.
   ⚠️ **Steam 비-Steam 게임 등록은 아직 남아 있음** — `shortcuts.vdf`에 `zotac-zone-paddles.sh`
   항목 존재 확인. 바이너리 VDF이고 Steam 실행 중엔 덮어써지므로 직접 편집하지 않았다. 사용자가
   Steam 라이브러리에서 우클릭 → 관리 → 비-Steam 게임 제거로 지워야 함. (스크립트를 옮겼으므로
   지금 실행하면 실패한다 — 어차피 hidraw2가 root 전용 + InputPlumber 점유라 동작 불가)
4. upstream 이슈 후보 **5건** 정리해서 올릴지 결정(§4에서 1건 추가됨).
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
