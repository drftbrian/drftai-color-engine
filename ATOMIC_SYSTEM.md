# Atomic System — Formula · Logic · Skill

0–1000 step 체계의 공식, 토큰이 만들어지는 논리, 그리고 실제로 쓰는 법.
전체 엔진 명세는 [SPEC.md](SPEC.md)에, 여기는 Atomic만 담습니다.

---

## 0. 한 줄

**색을 고르지 않습니다. step을 고릅니다.** 색은 공식이 만듭니다.

```
브랜드색 1개 + 커브 1개 + 틴트 1개  →  팔레트 5벌  →  토큰 전부
```

---

# PART 1 — FORMULA

## 1.1 step ⇄ grade

`step`은 0(흰색)에서 1000(검정)으로 가는 밝기 축입니다. 엔진 내부는 반대 방향의 `grade`로
돌아갑니다.

```
grade = (1000 − step) / 10
step  = 1000 − grade × 10
```

```
step    0   50  100  200  300  400  500  600  700  800  900  950 1000
grade 100   95   90   80   70   60   50   40   30   20   10    5    0
```

**양 끝이 촘촘한 이유** — 0·50·100은 50 간격, 그 뒤로는 100 간격입니다. 거의 흰색과 거의
검정 구간에서 1단계 차이가 눈에 더 크게 들어오기 때문입니다. 균등 간격으로 깔면 밝은 쪽
카드 레이어가 전부 같아 보입니다.

> 보간은 **step 숫자**로 합니다. 배열 인덱스로 하면 간격이 불균등해서 틀립니다.

## 1.2 A. Common Palette (neutral)

```
gray(step) = clamp(round(255 × curve.neutral(grade)))

Surface Mode = PRIMARY TINT 이면
  neutral(step) = lerp(gray(step), desaturate(brand, ×0.5), alpha(grade)/100)
  alpha(g)      = strength × shape(g)
```

브랜드를 **채도 절반**으로 낮춰 섞습니다. 그대로 섞으면 중성색이 아니라 색이 됩니다.

13스텝: `0 50 100 200 300 400 500 600 700 800 900 950 1000`

## 1.3 B. Opacity

```
0  4  8  12  16  24  32  40  48  56  64  72   (%)
```

앞 5단은 4% 간격, 그 뒤는 8% 간격입니다. 얇은 막(비활성 상태, 미묘한 구분)은 낮은 쪽에서
세밀하게, 덮는 막(모달 뒤)은 높은 쪽에서 거칠게 다루면 충분합니다.

## 1.4 C/D. Muted · Saturated Palette

11개 hue × 11 step. 각 hue는 **step 500에 HSL 앵커 하나**만 정의하고 나머지는 램프가 만듭니다.

| hue | H | S | L | | hue | H | S | L |
|---|---|---|---|---|---|---|---|---|
| `brown` | 24 | 38 | 44 | | `teal` | 176 | 64 | 38 |
| `red` | 358 | 70 | 52 | | `blue` | 222 | 80 | 52 |
| `orange` | 26 | 92 | 52 | | `purple` | 272 | 58 | 52 |
| `yellow` | 45 | 94 | 50 | | `pink` | 332 | 70 | 54 |
| `citron` | 66 | 78 | 46 | | `green` | 140 | 62 | 42 |
| `sprout` | 88 | 70 | 46 | | | | | |

```
base(hue, chroma) = hsl(H, S × chroma, L)
ramp(step)        = colorAt(base, grade(step), pivot = 50)

Saturated : chroma = 1.0
Muted     : chroma = CHROMA 슬라이더 (기본 0.42)
```

**pivot 50**이 핵심입니다. step 500이 앵커이므로 `hue/500`은 언제나 정의한 그 색이고,
위로는 흰색·아래로는 검정으로 갑니다.

hue 램프는 `50 … 950`의 11스텝입니다 — 0과 1000은 순백·순흑이라 hue가 의미를 잃습니다.

## 1.5 E. Primary Palette (brand)

```
primary(step) = colorAt(브랜드색, grade(step), pivot = 50)
```

C/D와 완전히 같은 규칙입니다. **`primary/500` = 입력한 브랜드색** 그 자체.

> DRFT Engine에서는 같은 색이 grade 40에 앉습니다. 스와치는 동일하고 라벨만 다릅니다.

## 1.6 알파 합성

```
withAlpha(hex, pct) = hex + round(pct / 100 × 255) → 8자리 hex
```

Overlay·Frost·`border/frost`가 이걸 씁니다.

---

# PART 2 — LOGIC

팔레트에서 토큰으로. 모든 semantic 토큰은 **팔레트의 한 칸을 가리키는 참조**입니다.

## 2.1 Semantic — background / foreground / border

| group | 토큰 | light | dark |
|---|---|---|---|
| **background** | `primary` | `n/0` | `n/950` |
| | `primaryActive` | `n/50` | `n/900` |
| | `secondary` | `n/50` | `n/900` |
| | `secondaryActive` | `n/100` | `n/800` |
| | `tertiary` | `n/100` | `n/800` |
| | `tertiaryActive` | `n/200` | `n/700` |
| | `inverse` | `n/1000` | `n/0` |
| | `inverseActive` | `n/900` | `n/50` |
| | `disabled` | `n/100` | `n/800` |
| **foreground** | `primary` | `n/1000` | `n/0` |
| | `secondary` | `n/700` | `n/300` |
| | `tertiary` | `n/500` | `n/500` |
| | `quarternary` | `n/400` | `n/600` |
| | `inverse` | `n/0` | `n/1000` |
| | `disabled` | `n/300` | `n/700` |
| **border** | `default` | `n/200` | `n/800` |
| | `light` | `n/100` | `n/900` |
| | `strong` | `n/400` | `n/600` |
| | `disabled` | `n/100` | `n/900` |
| | `inverse` | `n/1000` | `n/0` |
| | `frost` | `n/0 @ 12%` | `n/0 @ 12%` |

**다크는 뒤집기가 아닙니다.** `foreground/tertiary`는 양쪽 다 `n/500`이고(중간값이라 양쪽에서
작동), `background/primary`는 `n/0 ↔ n/950`으로 비대칭입니다 — 다크에서 순흑을 쓰면 그 위에
올릴 레이어가 없어집니다.

## 2.2 Status · Brand — 같은 문법

두 섹션이 **구조가 동일**합니다. Status는 saturated(또는 muted) 팔레트, Brand는 primary 팔레트.

| role | light | dark |
|---|---|---|
| `base` | 500 | 500 |
| `active` | 600 | 400 |
| `tint` | 50 | 900 |
| `tintActive` | 100 | 800 |

hue: `error → red` · `warning → yellow` · `success → green` · `info → blue`

### foreground·border는 step이 고정이 아닙니다

**자기 tint 대비 기준을 넘는 첫 step을 자동으로 고릅니다.**

| | 후보 (순서대로) | 기준 |
|---|---|---|
| foreground | light `600 700 800 900` · dark `400 300 200` | 4.5:1 |
| border | light `500 600 700` · dark `400 500 300` | 3:1 |

같은 step이라도 hue마다 대비가 완전히 다르기 때문입니다:

| light · saturated | error | warning | success | info |
|---|---|---|---|---|
| 선택된 fg step | 600 | **800** | **700** | 600 |
| tint 대비 | 5.83 | 7.91 | 6.16 | 6.65 |

`yellow/600`은 자기 tint 위에서 **2.58:1**입니다. 고정 step을 박으면 warning 하나가 깨집니다.
dark는 네 hue 모두 400에서 통과합니다(최소 4.82:1).

## 2.3 Overlay · Frost

Opacity 스케일에서 직접 파생됩니다.

| | 베이스 | 값 |
|---|---|---|
| **Overlay** | `n/1000` (양 테마) | `subtle 8` · `light 16` · `medium 32` · `default 48` · `strong 64` |
| **Frost** | light `n/0`, dark `n/1000` | `light 48` · `default 64` · `strong 72` |

Overlay는 **양 테마 모두 검정**입니다 — 덮는 게 목적이라 테마를 따라가면 안 됩니다.
Frost는 반대로 표면색이라 테마를 따라갑니다.

## 2.4 바인딩 인덱스

매핑 테이블을 한 번 순회해 **양방향** 인덱스를 만듭니다.

```
토큰 → atomic        foreground/secondary  →  neutral/700
atomic → 토큰        neutral/700           →  L·fg/secondary
                                              D·bg/tertiaryActive
                                              D·fg/disabled
```

한 step을 여러 토큰이 공유하는 게 정상입니다. 같은 밝기를 다른 역할로 쓰는 것이지,
중복이 아닙니다.

---

# PART 3 — SKILL

## 3.1 토큰 고르기

### background

| 상황 | 토큰 |
|---|---|
| 페이지·앱 바탕 | `primary` |
| 올라온 카드·패널·시트 | `secondary` |
| 파인 영역 — 인풋, 코드블록, well | `tertiary` |
| 위 셋의 hover·pressed | 같은 이름 + `Active` |
| 반전 — 토스트, 툴팁, 강조 배너 | `inverse` |
| 비활성 컨트롤 | `disabled` |

> 레이어를 세 단계 넘게 쌓고 있다면 토큰이 모자란 게 아니라 **화면이 너무 깊은** 겁니다.

### foreground

| 상황 | 토큰 | 대비 |
|---|---|---|
| 본문·제목 | `primary` | 최고 |
| 보조 설명·메타 | `secondary` | 충분 |
| 플레이스홀더·캡션 | `tertiary` | 경계 |
| 장식적 최하위 | `quarternary` | **AA 미달** |
| 반전 배경 위 | `inverse` | — |
| 비활성 | `disabled` | **AA 미달(의도됨)** |

`quarternary`와 `disabled`는 대비 기준을 통과하지 못합니다. **읽혀야 하는 글자에 쓰면 안 됩니다.**
비활성 컨트롤은 WCAG 예외라 괜찮고, `quarternary`는 구분선 대용·장식에만 씁니다.

### border

| 상황 | 토큰 |
|---|---|
| 기본 구분선·카드 윤곽 | `default` |
| 더 약하게 | `light` |
| 포커스·강조·선택됨 | `strong` |
| 유리·반투명 표면 위 | `frost` |

### status

| 상황 | 토큰 |
|---|---|
| 채워진 버튼·솔리드 배지 | `backgroundBase` + `foreground`는 그 위에 올리지 말 것 |
| 연한 알림 배너·태그 | `backgroundTint` + `foreground` |
| 테두리만 있는 경고 | `border` + `backgroundTint` |
| hover | `backgroundActive` / `backgroundTintActive` |

> `foreground`는 **tint 위에서** 대비가 계산됩니다. `base`(솔리드) 위에 올리면 보장되지 않습니다.
> 솔리드 버튼의 글자는 별도로 대비를 확인하세요.

## 3.2 새 색이 필요할 때

**순서대로 시도하고, 위에서 멈추세요.**

1. **기존 토큰으로 되나?** — 대부분 됩니다. 레이어가 깊어서 안 되는 거라면 화면을 줄이세요.
2. **같은 hue의 다른 step인가?** — 팔레트에 이미 11단이 있습니다. 토큰만 추가하면 됩니다.
3. **새 hue인가?** — step 500에 HSL 앵커 하나만 정의하면 11단이 자동 생성됩니다.
4. **어디에도 안 맞나?** — 예외로 표시하고 시스템 밖 리터럴로 내보내세요. 억지로 넣지 마세요.

**4번이 반드시 있어야 합니다.** 예외를 못 받는 시스템은 버려집니다. 브랜드 일러스트,
서드파티 로고, 마케팅 그라데이션은 원래 시스템 밖입니다.

## 3.3 확장

| 확장 | 바꿀 곳 | 자동으로 따라오는 것 |
|---|---|---|
| 새 hue | `HUES`에 `{name, h, s, l}` 한 줄 | Muted·Saturated 11단, 바인딩 인덱스 |
| 새 상태 (예: `pending`) | `A_STATUS_HUE`에 매핑 한 줄 | 4단 배경 + fg/border 자동 step 선택 |
| 새 테마 | 바인딩 표에 열 추가 | 모든 semantic 토큰 |
| 브랜드 변경 | 입력 hex 하나 | 팔레트 E, brand 토큰, 틴트된 neutral 전부 |

**구조를 건드리는 게 아니라 데이터를 한 줄 추가합니다.** 그게 안 되면 설계가 잘못된 겁니다.

## 3.4 흔한 실수

| 실수 | 왜 틀렸나 |
|---|---|
| `background/primary`를 브랜드색으로 이해 | 그건 페이지 바탕(거의 흰색/검정)입니다. 브랜드는 `brand/*` |
| status의 `tint`와 Surface Mode의 `PRIMARY TINT`를 혼동 | 전자는 연한 배경색, 후자는 중성색에 브랜드를 섞는 기능 |
| status foreground를 고정 step으로 박기 | yellow가 2.58:1로 깨집니다 |
| Muted chroma를 30% 아래로 | 네 상태가 회색으로 수렴해 **색으로 의미를 못 전합니다** (대비는 지켜지지만) |
| Overlay 대신 surface 토큰 사용 | 알파가 없어 겹침을 표현할 수 없습니다 |
| 다크를 라이트의 단순 반전으로 생각 | `foreground/tertiary`는 양쪽 같고, background는 비대칭입니다 |
| `quarternary`를 본문에 사용 | AA 미달입니다 |

## 3.5 체크리스트

새 화면·컴포넌트를 만들 때:

- [ ] 모든 색이 토큰 참조인가 — 리터럴 hex가 0개인가
- [ ] 배경 레이어가 3단 이내인가
- [ ] 글자는 `primary`·`secondary`만 쓰고 있는가
- [ ] status는 `tint` 위에 `foreground`를 올렸는가 (`base` 위가 아니라)
- [ ] 라이트·다크 **양쪽에서** 확인했는가
- [ ] 시스템 밖 색이 있다면 예외로 표시했는가

시스템을 바꿀 때:

- [ ] 데이터 한 줄 추가로 됐는가 (구조를 고쳤다면 멈추고 다시 생각)
- [ ] 자동 step 선택이 여전히 4.5:1 / 3:1을 만족하는가
- [ ] 바인딩 인덱스에 누락이 0인가
- [ ] 그리드 밖 step 참조가 0인가

---

## 부록 — 전체 토큰 목록

```
Layer 0
  neutral/{0 50 100 200 300 400 500 600 700 800 900 950 1000}
  opacity/{0 4 8 12 16 24 32 40 48 56 64 72}
  muted/{hue}/{50 100 200 300 400 500 600 700 800 900 950}
  saturated/{hue}/{…}
  primary/{…}
    hue ∈ brown red orange yellow citron sprout green teal blue purple pink

Layer 1
  background/{primary primaryActive secondary secondaryActive
              tertiary tertiaryActive inverse inverseActive disabled}
  foreground/{primary secondary tertiary quarternary inverse disabled}
  border/{default light strong disabled inverse frost}
  status/{error warning success info}/{backgroundBase backgroundActive
              backgroundTint backgroundTintActive foreground border}
  brand/{backgroundBase backgroundActive backgroundTint backgroundTintActive
              foreground border}
  overlay/{subtle light medium default strong}
  frost/{light default strong}
```

**합계**

| Layer 0 | | Layer 1 | |
|---|---|---|---|
| neutral | 13 | semantic | 21 |
| opacity | 12 | status (4 × 6) | 24 |
| muted (11 × 11) | 121 | brand | 6 |
| saturated (11 × 11) | 121 | overlay · frost | 8 |
| primary | 11 | | |
| **합** | **278** | **합** | **59** |

278개를 손으로 고르지 않습니다. 입력 세 개에서 나옵니다.
