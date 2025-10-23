# `AP_Core_Rules_v1.0.md` — **Single‑File Edition (No Lint/CI)**

**Single Source of Truth** · **Updated:** 2025‑10‑24 (KST) · **Scope:** ANSYS PROJECT (Ansys 2024 R2; Fluent · Mechanical(Not APDL) · System Coupling; Windows/Linux)

---

## 0) Purpose & Operating Philosophy

* **Purpose:** 본 문서는 별도 메모리에 의존하지 않고, 이 한 파일만으로 **병렬‑세이프**, **결정론적**, **릴리스‑레디** 코드를 작성할 수 있게 합니다.
* **Normative keywords:** 이 문서는 RFC 스타일의 용어를 사용합니다. **MUST(필수)** / **MUST NOT(금지)** / **SHOULD(권장)** / **MAY(허용)**.
* **Authority & Precedence:**

  1. **Fluent 2024 R2 UDF Manual** & **Fluent Theory Guide** (최상위),
  2. 본 문서,
  3. 예시 코드.
     매뉴얼과 본 문서가 충돌하면 **매뉴얼이 우선**입니다.
* **Default Application:** “AP”가 붙은 작업/대화에는 **항상 본 문서 규칙을 기본 적용**합니다. 추가 규정이 필요하면 본 문서에 **직접 기록**해 반영합니다.
* **Deviation Protocol (예외 적용 절차):** 불가피하게 규칙을 어겨야 한다면, (a) 목적/범위/기간을 명시, (b) **노드 계산 원칙**은 유지, (c) 재현성 확보(난수 시드/정렬 키 고정), (d) 실험 종료 후 **원복**. 예외는 문서 하단 **Deviation Log**에 1~3줄로 기록합니다.
* **Non‑Goals:** 본 문서는 C 언어 교재가 아니며, 매뉴얼을 대체하지 않습니다. 매크로/시그니처는 **반드시** 매뉴얼을 교차 확인하세요.

---

## 1) Baseline & Scope (고정 전제)

* **Version:** Ansys **2024 R2**
* **OS:** Windows / Linux (identical behavior required)
* **Solvers:** **Fluent**, **Mechanical**, **System Coupling** (※ **MAPDL excluded**)
* **Manual Compliance (Strict):**

  * `#include "udf.h"` MUST be the **very first include**.
  * Use only **documented** macros/types (`cell_t`, `face_t`, `Thread*`, `Domain*`, `real`, `cxboolean`, etc.).
  * **MUST NOT** invent PRF/RP macros or typedefs. 매뉴얼에 없는 것은 사용하지 마세요.

---

## 2) The 10 Core Rules (Enforced)

1. **Host vs Node separation**
   **RP_NODE**: loops, field access, compute, UDM/UDS R/W
   **RP_HOST**: file I/O, settings parsing, name→ID/metadata resolution, final **one‑line** logging only
2. **Global reductions: assignment‑form only**
   Use: `sum = PRF_GRSUM1(sum);`
   Avoid: `PRF_GRSUM1(sum);`, `sum += PRF_GRSUM1(sum);`
3. **Orchestrator pattern (mandatory)**
   **Node compute → PRF reduction (assignment) → (optional) node_to_host_* → Host Message() ×1**
4. **Ownership & Interfaces**
   Write to **owned entities only**. For faces, guard writes with **`PRINCIPAL_FACE_P(f,t)`** to avoid duplicates across partitions.
5. **No entity pointers/IDs across ranks**
   **MUST NOT** transmit local IDs, `Thread*`, `cell_t/face_t` across host/ranks. Transmit **global scalars only**, and **re‑resolve by name/attributes on nodes**.
6. **Logging rules**
   Host logging uses **`Message("...\n")`** only. `Message0()` is **forbidden in release**.
   Node‑side debug logging, if any, **SHOULD** be guarded (e.g., `if (I_AM_NODE_ZERO_P) Message("...\n");`).
7. **Determinism**
   Fix sort keys (name, coordinate+ID), prefer **double** precision, optionally fix RNG seeds.
8. **Restart hygiene**
   Reset state in **`DEFINE_INIT`** before `ADJUST` / `EXECUTE_AT_END` hooks run.
9. **Comments & Encoding (ASCII‑only)**
   Comments and string literals **MUST be English ASCII only**. Avoid any non‑ASCII characters that can break under ANSI/code‑page differences: Korean, emojis, smart quotes (" “ ” ‘ ’ "), en/em dashes (– —), non‑breaking spaces, math/Greek glyphs and symbols (≤ ≥ ≠ ± × · ∞ ° µ π), superscripts/subscripts, etc.
   **Use ASCII replacements:** `<=`, `>=`, `!=`, `+/-`, `*` (or `dot(a,b)`), `degC`/`deg`, `um` (not `µm`), `pi` (not `π`), `micro` (not `µ`).
   File names, macro names, and identifiers **MUST** be ASCII‑only.
10. **Release format**
    Each file starts with **[Source of Truth · Version · Intent]** and includes a **UDM/UDS index table**. Release builds remove debug prints and keep formatting clear.

---

## 3) Strong Manual Compliance (How to avoid “wrong macro/type”)

**Goal:** UDF Manual에 없는 기능/함수/타입/매크로 사용을 **제로**로. 아래를 항상 수행합니다.

* **S1. 이름 확인:** 쓰려는 매크로/함수 이름을 **그대로** UDF Manual 색인/장 제목에서 검색합니다.
* **S2. 시그니처 확인:** 인자·반환형·전처리 조건이 문서와 **정확히 일치**하는지 확인합니다.
* **S3. 파일 상단 규칙:** `#include "udf.h"`는 **항상 1번째**. 임의 typedef/매크로로 문서 타입/매크로를 **덮어쓰지 말 것**.
* **S4. PRF/RP 접두 경계:** `PRF_*/RP_*`는 **문서에 나온 것만** 사용. 이름 추정/조합 금지(예: 문서에 없는 `_MAX1`/`_MIN1` 변형 **추측 금지**).
* **S5. 예제 우선:** 문서 예제와 **동일한 호출 순서/가드**로 구성(특히 `#if RP_NODE` / `#if RP_HOST`).

> 반복 실패의 80%는 **S1~S3**만 제대로 지켜도 제거됩니다.

---

## 4) Reference Patterns (Copy‑Paste Safe)

### 4.1 Node‑compute → Global‑reduce → Host‑log

```c
/* Example pattern — comments are in English only */

real gsum = 0.0; /* global metric container */

#if RP_NODE
  /* 1) Compute on nodes: loops/field access live here */
  /* ... your local accumulation into gsum ... */

  /* 2) Global reduction: assignment form only */
  gsum = PRF_GRSUM1(gsum);

  /* 3) If the host needs it as a scalar, transmit via node_to_host_* API */
  /* node_to_host_* usage is model‑specific; follow the UDF Manual exactly */
#endif

#if RP_HOST
  /* 4) Final single‑line logging on the host */
  Message("AP: gsum = %.12g\n", gsum /* value received on host */);
#endif
```

### 4.2 Face write with interface guard

```c
/* Write only to owned faces; guard with PRINCIPAL_FACE_P to avoid duplicates */

begin_f_loop(f, t)
{
  if (PRINCIPAL_FACE_P(f,t))
  {
    /* safe to write: UDM/BC profile/etc. */
    /* F_UDM(f,t,udm_i) = value; */
  }
}
end_f_loop(f,t)
```

### 4.3 Restart state hygiene

```c
/* Reset flags/caches here so a restarted run is consistent */
DEFINE_INIT(ap_reset_state, domain)
{
  /* e.g., ap_heater_off = 0; ap_max_t = -1.0e30; */
}
```

---

## 5) Anti‑Patterns (do not do this)

* Host block with loops/field access (e.g., `#if RP_HOST` containing `begin_c_loop`, `C_T`, `F_UDM`)
* Non‑assignment PRF reduction (e.g., `PRF_GRSUM1(sum);`, `sum += PRF_GRSUM1(sum);`)
* Face writes without `PRINCIPAL_FACE_P(f,t)` guard
* Passing local IDs/`Thread*` to host/other ranks instead of name/attribute re‑lookup
* Flood logging inside loops without `I_AM_NODE_ZERO_P`
* Missing `#include "udf.h"` at the top or using types/macros not in the manual

---

## 6) Manual Pre‑Commit Checklist (Human‑run)

* [ ] `#include "udf.h"` is the **first include**

* [ ] All loops/field access under **`#if RP_NODE`**

* [ ] PRF reductions in **assignment form** only: `X = PRF_G*1(X);`

* [ ] Face writes guarded by **`PRINCIPAL_FACE_P(f,t)`**

* [ ] Host logging via **`Message()`**; no `Message0()` in release

* [ ] No IDs/`Thread*` sent across ranks; **re‑resolve by name/attributes on nodes**

* [ ] `DEFINE_INIT` resets state for restart consistency

* [ ] **ASCII‑only comments/strings** — no Korean/emoji/math glyphs; use ASCII replacements (<=, >=, !=, +/-, *, degC, um)

> 2분 셀프체크: 위 7항목만 눈으로 스캔합니다. 자동 린트/CI는 사용하지 않습니다.

---

## 7) High‑Risk Mistakes Catalog (자주 틀리는 것들 — 사례 중심)

> 오류가 반복될 때마다 **여기에 항목을 추가**하세요. 각 항목은 “증상 → 원인 → 올바른 패턴” 3줄로 요약합니다.

* **H1. 호스트 블록에 루프/필드 접근 배치**
  증상: 병렬에서 값 뒤섞임, 크래시 또는 무의미 값
  원인: `#if RP_HOST` 내부에 `begin_c_loop/C_T/F_UDM` 존재
  해결: 루프/필드 접근은 **전부** `#if RP_NODE`로 이동. 호스트는 I/O·요약 출력만

* **H2. PRF 전역 축약 비대입형 사용**
  증상: 값 2중 합산/무효 결과
  원인: `PRF_GRSUM1(x);` 또는 `x += PRF_GRSUM1(x);`
  해결: **`x = PRF_GRSUM1(x);`** 로 통일

* **H3. 인터페이스 face 중복 쓰기**
  증상: UDM/BC가 인터페이스에서 2번 기록되어 불일치
  원인: `PRINCIPAL_FACE_P(f,t)` 가드 누락
  해결: face 기록 구간을 **항상** `if (PRINCIPAL_FACE_P(f,t)) { ... }`로 감쌉니다

* **H4. 문서에 없는 매크로/타입 사용**
  증상: 컴파일 에러, 런타임 비정상
  원인: 매뉴얼 미등재 PRF/RP/타입/함수 임의 사용
  해결: S1~S5 절차로 **이름·시그니처** 확인, 미등재면 사용 금지

* **H5. 재시작 시 상태 변수 고착**
  증상: heater off 상태가 초기화 안 됨
  원인: `DEFINE_INIT` 미구현
  해결: 초기화는 **항상** `DEFINE_INIT`에 배치

> 필요 시 H6+ 항목을 계속 추가하세요(날짜, 시뮬레이션 이름, 커밋 링크 등 메모 권장).

---

## 8) 2‑Minute Self‑Review Flow (수동 검증 루틴)

0. **ASCII‑only 확인:** 주석/문자열/식별자에 비ASCII 문자가 없는지 확인(≤ ≥ × µ π 등은 금지; <= >= * um 등으로 대체)
1. 파일 맨 위: `#include "udf.h"` 확인
2. `#if RP_HOST` 블록 검색 → 루프/필드 접근이 **없는지** 확인
3. `PRF_G` 검색 → 모두 **대입형**인지 확인
4. `begin_f_loop` 검색 → 내부에 **`PRINCIPAL_FACE_P`** 있는지 확인
5. `Message0(` 검색 → 릴리스에서 **없음** 확인
6. `DEFINE_INIT` 존재/초기화 변수 확인
7. “로컬 ID/Thread 포인터 전달” 흔적이 없는지 스캔(있다면 이름/속성 재해석으로 수정)

---

## 9) Required File Header & UDM/UDS Index Table

**Header template** (must appear at the top of each source file):

```
[Source of Truth]    repo/AP_Core_Rules_v1.0.md
[Version]            vX.Y (YYYY-MM-DD)
[Intent]             One‑line purpose of this file
[Compatibility]      Ansys Fluent 2024 R2; Win/Linux; Clang‑safe
```

**UDM/UDS index table** (keep a single registry across the project):

```
| Symbol                | Index | Purpose                        |
|-----------------------|-------|--------------------------------|
| AP_UDM_HEATER_MAXT    |   0   | Track max temperature in solid |
| AP_UDM_Q2_STATE       |   7   | Q2 trigger state               |
| ...                   |  ...  | ...                            |
```

---

## 10) Change Control & Versioning

* **Versioning:** `vMAJOR.MINOR` (예: v1.0, v1.1). 규정 강화·해석 명확화는 **MINOR** 증가, 원칙 변경은 **MAJOR** 증가.
* **Update process:** 변경 시 본문과 체크리스트·카탈로그를 **동시에 갱신**하고, Changelog에 1~2줄로 요약 추가.
* **Single‑file policy:** 본 파일만 유지합니다. 다른 문서는 만들지 않습니다.

---

## Deviation Log (optional)

* *(예시)* 2025‑10‑28 · 프로젝트 X · 노이즈 억제 실험 위해 루프 내 Message() 허용(노드0 가드 적용). 결과 검토 후 원복.

---

## Changelog

* **v1.0 (2025‑10‑24):** Initial memory‑less single‑file baseline; expanded Purpose with precedence/Default Application/Deviation Protocol. Integrated checklist & mistakes catalog; removed lint/CI and external memories.
