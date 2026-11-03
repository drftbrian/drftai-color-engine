# DRFT_1 Color Engine — Specification

생성 규칙 하나에서 두 개의 토큰 체계와 제품 색이 전부 파생되는 구조의 전체 명세.
모든 수치는 `index.html`의 실제 상수에서 추출했습니다.

---

## 0. 한 장 요약

```
LAYER 0   생성 규칙      커브 + 브랜드색 + 틴트  →  grade 0–100 위의 연속 색
   ↓
LAYER 1   토큰 체계      grade를 집어 이름을 붙임 (두 taxonomy가 각자)
   ↓
LAYER 2   제품 role      제품이 필요한 역할을 선언, 각 체계가 자기 어휘로 채움
```

입력은 셋뿐입니다 — **브랜드색 1개 · 커브 1개 · 틴트 강도 1개.**
나머지는 전부 계산됩니다.

---

## 1. Layer 0 — 생성 규칙

### 1.1 grade

`grade`는 0(검정) ~ 100(흰색)의 밝기 축입니다. 모든 토큰은 이 축 위의 한 점입니다.

| | 스톱 |
|---|---|
| DRFT 13스톱 | `100 98 95 90 80 70 60 50 40 30 20 10 0` |
| Atomic 13스텝 | `0 50 100 200 300 400 500 600 700 800 900 950 1000` |

두 체계의 관계: **`step = 1000 − grade × 10`**

```
step    0   50  100  200  300  400  500  600  700  800  900  950 1000
grade 100   95   90   80   70   60   50   40   30   20   10    5    0
```

### 1.2 그라데이션 커브

`grade → 레벨(0–1)`. 중성색은 `gray = clamp(round(255 × neutral(g)))`.

| 프리셋 | 공식 | 100 | 90 | 70 | 50 | 30 | 10 | 0 |
|---|---|---|---|---|---|---|---|---|
| `SOURCE` | `(2.575g − 2.75) / 255` | 255 | 229 | 178 | **126** | 75 | 23 | 0 |
| `REPO` | `g / 100` | 255 | 230 | 179 | **128** | 77 | 26 | 0 |
| `PERCEPTUAL` | `L* = g → sRGB` | 255 | 226 | 171 | **119** | 71 | 27 | 0 |
| `SOFT` | `(g/100) ^ 0.72` | 255 | 236 | 197 | **155** | 107 | 49 | 0 |
| `CONTRAST` | `t + 0.55·(3t² − 2t³ − t)`, `t = g/100` | 255 | 240 | 190 | **128** | 65 | 15 | 0 |
| `CUSTOM` | monotone cubic spline (Fritsch–Carlson) | 직접 편집 | | | | | | |

모든 커브는 `g=0`에서 `#000000`, `g=100`에서 `#FFFFFF`로 고정되고 단조 증가합니다.
CUSTOM의 보간은 오버슈트가 구조적으로 불가능해, 드래그 중에도 램프 순서가 뒤집히지 않습니다.

### 1.3 유채색 램프 — `interpolatePrimary(grade, base, pivot)`

기준색을 `pivot`에 두고 위로는 흰색, 아래로는 검정으로 보냅니다. `pivot`의 기본값은 **40**이고,
Atomic의 hue 팔레트는 **50**(= step 500)을 씁니다.

```
target = grade > pivot ? 255 : 0

factor =
  참조 커브(SOURCE·REPO)      |(grade − pivot) / (grade > pivot ? 100 − pivot : −pivot)|
  그 외                       grade > pivot ? (L − Lp)/(1 − Lp) : (Lp − L)/Lp
                              L = colorLevel(grade), Lp = colorLevel(pivot)

channel = quant(lerp(base, target, factor))
quant   = REPO ? round : floor
```

SOURCE·REPO가 정수 연산을 그대로 쓰는 이유는 §6의 재현 보증 때문입니다. 부동소수점으로
유도하면 `.5` 경계에서 1/255씩 어긋납니다.

### 1.4 Primary Tint — 중성색에 브랜드를 섞기

```
alpha(g) = strength × shape(g)
mixed    = lerp(gray(g), desaturate(brand, ×0.5), alpha / 100)
```

브랜드색을 **채도 절반**으로 낮춘 뒤 섞습니다. 그대로 섞으면 회색이 아니라 색이 됩니다.

`shape`는 grade 기준 선형 보간입니다. **스톱 인덱스가 아니라 grade 숫자**로 보간해야 합니다 —
`100·98·95·90`은 2·3·5 간격인데 그 아래는 전부 10 간격이기 때문입니다.

| shape | 제어점 `(grade, 배율)` |
|---|---|
| `OFF-WHITE` (sheet) | `(0,0) (50,1) (90,0.2) (99,0.2) (100,0)` |
| `PURE WHITE` (tent) | `(0,0) (50,1) (100,0)` |

두 shape은 **grade 90 아래에서 완전히 동일**합니다. 차이는 91–99 구간, 즉 라이트 모드의
`background` `surfaceLow` `surface` 세 토큰뿐이고 다크 모드는 영향이 없습니다.

`strength` 10%일 때 sheet의 값이 소스 시트 원본과 일치합니다:
`0 2 2 2 4 6 8 10 8 6 4 2 0`

---

## 2. Layer 1 — DRFT Engine taxonomy

### 2.1 Primary

| 토큰 | grade |
|---|---|
| `onPrimaryVariant` | 10 |
| `primary2` | 25 |
| `primary` | **40** (입력색) |
| `primaryVariant` | 90 |
| `onPrimary` | 100 |

### 2.2 Surface

| 토큰 | light | dark |
|---|---|---|
| `background` | 99 | 6 |
| `backgroundBright` | 100 | 24 |
| `backgroundDim` | 87 | 0 |
| `surfaceLowest` | 100 | 8 |
| `surfaceLow` | 98 | 10 |
| `surface` | 95 | 12 |
| `surfaceHigh` | 90 | 17 |
| `surfaceHighest` | 80 | 22 |
| `onSurface` | 10 | 90 |
| `onSurfaceVariant` | 30 | 80 |
| `outline` | 50 | **40** |
| `outlineVariant` | 80 | **20** |
| `text` | 8 | 95 |
| `text2` | 25 | 80 |

`scrim`만 고정값입니다 — light `#12121266`, dark `#121212cc`.

다크 `outline` / `outlineVariant`는 배포본의 70 / 50이 아니라 40 / 20입니다. 근거는 §6.

### 2.3 Status

`error` `warning` `success` 각각 입력색을 grade 40에 두고 램프합니다.

| 토큰 | grade |
|---|---|
| `{name}` | 40 |
| `on{Name}` | 100 |
| `{name}Container` | 90 |
| `on{Name}Container` | 10 |

### 2.4 REPO 호환 틴트 알파

`REPO` 커브를 고르면 커브 대신 토큰별 하드코딩 알파를 씁니다(배포본 재현용). `strength/10`에
비례해 스케일됩니다.

| | light | dark |
|---|---|---|
| `background` | 2 | 2 |
| `backgroundBright` | 0 | 6 |
| `backgroundDim` | 4 | 0 |
| `surfaceLowest` | 0 | **8** |
| `surfaceLow` | 2 | **10** |
| `surface` | 2 | **12** |
| `surfaceHigh` | 2 | **17** |
| `surfaceHighest` | 4 | **22** |
| `onSurface` | 2 | 2 |
| `onSurfaceVariant` | 6 | 4 |
| `outline` | 10 | 6 |
| `outlineVariant` | 4 | 10 |
| `text` | 2 | 2 |
| `text2` | 6 | 4 |

다크 쪽 굵은 값들은 `alpha = grade`로 박혀 있습니다. 커브 기준값과 어긋납니다 — §6 참고.

---

## 3. Layer 1 — Atomic System taxonomy

### 3.1 팔레트

| | 내용 |
|---|---|
| **A. Common** | neutral 13스텝 (`step → grade` 변환 후 Layer 0) |
| **B. Opacity** | `0 4 8 12 16 24 32 40 48 56 64 72` (%) |
| **C. Muted** | 11 hue × 11 step, 채도 = CHROMA 슬라이더 |
| **D. Saturated** | 같은 hue 앵커, 채도 100% |
| **E. Primary** | 브랜드 램프. 레퍼런스에 없는 우리 시스템만의 레이어 |

hue 앵커는 step 500에 정의됩니다 (HSL):

| hue | H | S | L | | hue | H | S | L |
|---|---|---|---|---|---|---|---|---|
| `brown` | 24 | 38 | 44 | | `teal` | 176 | 64 | 38 |
| `red` | 358 | 70 | 52 | | `blue` | 222 | 80 | 52 |
| `orange` | 26 | 92 | 52 | | `purple` | 272 | 58 | 52 |
| `yellow` | 45 | 94 | 50 | | `pink` | 332 | 70 | 54 |
| `citron` | 66 | 78 | 46 | | `green` | 140 | 62 | 42 |
| `sprout` | 88 | 70 | 46 | | | | | |

### 3.2 Semantic 바인딩

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

`n/x` = Common Palette의 step x.

### 3.3 Status · Brand

둘이 **같은 문법**을 씁니다. Status는 saturated(또는 muted) 팔레트, Brand는 primary 팔레트.

| role | light | dark |
|---|---|---|
| `base` | 500 | 500 |
| `active` | 600 | 400 |
| `tint` | 50 | 900 |
| `tintActive` | 100 | 800 |

hue 매핑: `error → red`, `warning → yellow`, `success → green`, `info → blue`

**foreground와 border는 step이 고정이 아닙니다.** 자기 tint 대비 기준을 넘는 첫 step을
자동 선택합니다.

| | 후보 (순서대로 시도) | 기준 |
|---|---|---|
| foreground | light `600 700 800 900` · dark `400 300 200` | **4.5:1** |
| border | light `500 600 700` · dark `400 500 300` | **3:1** |

같은 step이라도 hue마다 대비가 크게 다르기 때문입니다 — `yellow/600`은 자기 tint 위에서
2.58:1인데 `blue/600`은 6.65:1입니다. 고정 step으로는 어느 쪽이든 깨집니다.

### 3.4 Overlay · Frost

Opacity 스케일에서 직접 파생됩니다. Elevation은 범위 밖입니다.

| | 베이스 | 값 |
|---|---|---|
| **Overlay** | `n/1000` (양 테마) | `subtle 8` · `light 16` · `medium 32` · `default 48` · `strong 64` |
| **Frost** | light `n/0`, dark `n/1000` | `light 48` · `default 64` · `strong 72` |

---

## 4. Layer 2 — 제품 role

제품은 역할만 선언하고, 각 taxonomy가 자기 어휘로 채웁니다. 제품 코드는 토큰 이름을 모릅니다.

| role | 쓰이는 곳 | DRFT Engine | Atomic System |
|---|---|---|---|
| `appBg` | 앱 배경 | `background` | `background/primary` |
| `raised` | 아바타 · 추가 버튼 | `surfaceHigh` | `background/tertiary` |
| `textPrimary` | 제목 · 본문 | `text` | `foreground/primary` |
| `textSecondary` | 보조 텍스트 | `text2` | `foreground/secondary` |
| `textTertiary` | 셰브론 · 비활성 탭 | `onSurfaceVariant` | `foreground/tertiary` |
| `divider` | 구분선 · 탭바 경계 | `outlineVariant` | `border/default` |
| `accent` | 활성 탭 | `primary` | `brand/backgroundBase` |
| `petA` | 펫 카드 1 | primary 램프 @45 *(파생)* | `primary/500` |
| `petB` | 펫 카드 2 | hue +150° @45 *(파생)* | `saturated/teal/500` |

**두 체계가 같은 값을 내도록 맞추지 않습니다.** `petB`가 그 차이를 그대로 드러냅니다 —
DRFT에는 이름 붙은 hue 팔레트가 없어서 색을 만들어내야 하고, Atomic에는 11개가 있어서
골라 쓰면 됩니다.

액센트 위 텍스트는 흑백이 아니라 **그 액센트 자신의 램프 양 끝**(grade 100 / 10)에서
대비가 높은 쪽을 고릅니다. 순수 검정·흰색은 시스템 밖 색이라 티가 납니다.

---

## 5. Audit — 남의 팔레트를 설정으로 역산

### 5.1 입력

| 경로 | 처리 |
|---|---|
| 텍스트 | CSS 변수 · 토큰 JSON · Tailwind config · hex 목록 → `[{name?, hex, count}]`. 이름이 있으면 보존 |
| 이미지 | 픽셀에서 추출 (아래) |
| Figma 링크 | **불가** — 브라우저 단일 페이지라 접속 경로가 없습니다. 안내만 표시 |

### 5.2 픽셀 추출

긴 변 1100px로 줄인 뒤, 아래 둘 중 하나를 만족하는 픽셀만 셉니다.

```
평면 채움   상하좌우 네 이웃이 모두 자신과 같음
헤어라인   한 축으로 양 이웃이 같고, 그 축을 가로지르는 두 픽셀은
           서로는 같고 자신과는 다름
```

첫 규칙이 안티에일리어싱·그라데이션·사진 노이즈를 걸러냅니다. 둘째가 1px 구분선과
테두리를 건집니다 — 이게 없으면 `outline` 계열 토큰을 통째로 놓칩니다.
세로 그라데이션이 둘째 규칙을 통과하지 못하는 이유는 "가로지르는 두 픽셀이 서로 같을 것"
조건 때문입니다.

이후 평면 픽셀의 0.008% 미만은 버리고, Δ≤2끼리 병합해 최대 48색을 남깁니다.

### 5.3 분류

**틴트 서명** — 시스템 색은 브랜드 쪽으로 틴트돼 있고(`max(rgb) − min(rgb) > 2`, 색상각이
브랜드와 45° 이내), 손으로 찍은 색은 중립입니다.

단, 이건 **사전 확률일 뿐입니다.** grade 0이나 100에서는 틴트가 드러날 여지가 없어 멀쩡한
시스템 색도 중립으로 보입니다. 그래서 최종 판정은 **잔차**로 합니다:

```
1. 틴트된 색으로 설정을 푼다
2. 그 설정으로 모든 중립색을 재현해본다
3. Δ ≤ 2 면 시스템, 아니면 드리프트
```

드리프트는 가장 가까운 시스템 색과의 채널 최대 거리로 나눕니다.

| Δ | 판정 | 동작 |
|---|---|---|
| ≤ 2 | 같은 색 | 자동 병합 |
| 3 – 6 | 드리프트 | 스냅 |
| > 6 | 다른 색 | **사람 결정** |

### 5.4 솔버

`커브 × shape × 강도(0–60)`를 훑어 **최대 Δ를 최소화**하는 조합을 찾습니다. 색마다
grade는 그 안에서 탐색합니다. 강도는 1% 간격으로 훑은 뒤 최적 주변 ±2%를 0.25%로 정밀화합니다.

결정 불가능한 건 그렇게 말합니다 — grade 90 위 색이 없으면 두 shape을 구분할 수 없으므로
`sheet?`로 표시합니다.

---

## 6. 알려진 불일치 5건

디자인 소스(Figma 변수 · `Color Source (Do Not Edit)` 시트)와 배포된 `drft_1` 코드가
어긋나는 지점입니다. `SOURCE` / `REPO` 프리셋으로 양쪽 모두 재현할 수 있습니다.

**① 그레이스케일 공식** — 시트 13개 값은 `round(2.575g − 2.75)`로 13/13 정확히 재현됩니다.
코드의 `round(g/100 × 255)`는 중간 구간이 1–2단계 밝습니다.

| grade | 90 | 80 | 70 | 60 | 50 | 40 | 30 | 20 | 10 |
|---|---|---|---|---|---|---|---|---|---|
| 시트 | E5E5E5 | CBCBCB | B2B2B2 | 989898 | 7E7E7E | 646464 | 4B4B4B | 313131 | 171717 |
| 코드 | E6E6E6 | CCCCCC | B3B3B3 | 999999 | 808080 | 666666 | 4D4D4D | 333333 | 1A1A1A |

**② 채널 반올림** — 시트는 버림. `#B3261E`의 90은 시트가 `#F2DAD9`, 코드의 `round`는 `#F2DBDA`.

**③ 다크 outline이 30 grade 밝습니다** — 디자인 파일 변수를 역산하면 `outline`은 grade 40
(`#65686D`), `outlineVariant`는 20(`#353639`)에서 나옵니다. 코드는 70 / 50이라 near-black 위에
거의 흰색 헤어라인이 됩니다. 둘 다 정확히 30씩 차이나 의도된 보정으로 보고 디자인 파일을
따랐습니다. 라이트 모드와 모든 램프는 영향 없습니다.

**④ 다크에서 `text2`와 `onSurfaceVariant`가 같은 색입니다** — 둘 다 grade 80이라 값이 완전히
동일합니다(`#C6C8CB`). 텍스트 계열 토큰이 4개인데 다크에서는 3단계만 나옵니다. 라이트에서는
25 / 30으로 갈라집니다. Atomic은 같은 자리에서 3단계가 모두 다릅니다.

**⑤ 다크 서피스 틴트 알파** — 시트의 알파는 50에서 정점인 선형 텐트인데, 코드의 다크 모드는
`alpha = grade`로 하드코딩돼 있습니다 (`surfaceHighest`: grade 22 → 22%, 텐트 기준으론 4%).
라이트 모드는 텐트와 일치합니다.

---

## 7. Export 포맷

| 포맷 | 내용 |
|---|---|
| CSS 변수 | layer 0 (`--src-*` / `--n-*`) + layer 1 + layer 2 (`--role-*`), 다크는 미디어쿼리와 `[data-theme]` 양쪽 |
| Tailwind | `colors` 확장, hue는 스케일로 |
| JSON | `meta` + `source` + `semantic` + `bindings` + `product` |
| Figma Tokens | `{value, type}` 중첩 구조 |
| SwiftUI | `enum Palette` (`Color.primary` 등 시스템 이름과 충돌 방지) |

---

## 8. 검증 앵커

변경 후 반드시 통과해야 하는 값들입니다.

**`SOURCE` 프리셋**

```
그레이 13      #FFFFFF #FAFAFA #F2F2F2 #E5E5E5 #CBCBCB #B2B2B2 #989898
               #7E7E7E #646464 #4B4B4B #313131 #171717 #000000
status 앵커    #F2DAD9 / #2C0907   (error   #B3261E)
               #FDE9D4 / #3D1F00   (warning #F57C00)
               #DCE9DC / #0B1F0C   (success #2E7D32)
틴트 알파      0 2 2 2 4 6 8 10 8 6 4 2 0        (strength 10 · sheet)
```

**`REPO` 프리셋** — `#E6E6E6` · `#F2DBDA` / `#2D0A08`

**Audit 라운드트립** — 엔진 자기 출력을 입력하면 넣은 설정을 **Δ0으로 복원**해야 합니다.
이게 게이트입니다. 못 되찾으면 나머지는 볼 것도 없습니다.

| 케이스 | 기대 |
|---|---|
| 990 Product Builder (Figma 변수 + 드리프트 6색) | `source · #005EB5 · sheet · 29%` · **Δ2**, 그레이 10개를 시스템 7 / 중립 3으로 분리 |
| DRFT_1 Figma 변수 9색 | `source · #276EF1 · sheet · 10.25%` · **Δ1** |
| 합성 이미지 (함정 포함) | 디자인 색 9개 전부 회수, 함정 0 통과, 텍스트 경로와 동일한 역산 |

**디자인 파일 대조** — 기본 상태(primary `#276EF1` · SOURCE · 틴트 10% · dark · Primary Tint)에서
셸의 계산색이 Figma 변수와 채널당 최대 **Δ4**, primary 3종(`#276ef1` `#dbe6fc` `#ffffff`)은
완전 일치.
