# DRFT_1 — Color Engine

[drft_1](https://github.com/minus31/drft_1)의 컬러 엔진만 떼어내 만든 단일 파일 웹앱.
의존성이 없고 빌드도 필요 없습니다 — `index.html`을 브라우저로 열면 끝입니다.

원본 로직: `components/DesignSystemLogic/Colors/colorSystem.ts` + `colorStore.ts`

---

## 화면

좌측 패널에 모든 컨트롤, 우측에 `UI Preview` / `Tokens` 두 탭.

- **UI Preview** — iPhone 프레임 안에 실제 앱 화면(마이페이지)이 라이브로 그려집니다.
  전부 semantic 토큰으로만 칠해져 있어서 브랜드 색을 바꾸면 폰 안이 즉시 리테마됩니다.
  하드코딩된 hex는 0건.
- **Tokens** — 좌측 `System` 세그에 따라 DRFT Engine 5섹션 / Atomic System 7섹션이 전환됩니다.

셸 자체도 엔진 출력으로 칠해집니다. 디자인 파일(Figma)의 변수 이름이 우리 토큰 이름과
같아서(`Base Color/Surface/*Surface`, `Primary/*Primary Variant` …) 그대로 연결했습니다.

## 구조

```
LAYER 0  Color Source   13스톱 (100·98·95·90·80·70·60·50·40·30·20·10·0) × 6램프
   ↓ grade 조회
LAYER 1  Semantic       primary ← primary/40 · surface ← gray/95 · errorContainer ← error/90 …
```

모든 semantic 토큰은 layer 0의 특정 grade를 가리키는 참조입니다. 커브를 바꾸면 layer 0이
다시 계산되고, 그 위의 토큰이 전부 따라 움직입니다.

## 그라데이션 커브

grade(0–100)를 출력 레벨(0–1)로 보내는 함수 하나가 layer 0 전체를 만듭니다. 채널은 둘입니다.

| 채널 | 하는 일 |
|---|---|
| `NEUTRAL` | `gray = round(255 × level)` |
| `COLOR` | 기준색이 흰색(40 위) / 검정(40 아래)으로 가는 이동량 |

어떤 커브를 쓰든 **grade 40은 항상 입력색 그대로**입니다.

### 프리셋

| 이름 | 공식 | grade 50 |
|---|---|---|
| `SOURCE` | 그레이 `round(2.575g − 2.75)`, 컬러 선형 | `#7E7E7E` |
| `REPO` | `round(g/100 × 255)`, 채널 반올림 | `#808080` |
| `PERCEPTUAL` | `L* = g → sRGB` | `#777777` |
| `SOFT` | `(g/100) ^ 0.72` | `#9B9B9B` |
| `CONTRAST` | S-curve, k 0.55 | `#808080` |
| `CUSTOM` | monotone cubic spline | 직접 편집 |

### CUSTOM — 커브 편집기

왼쪽 패널의 **Gradation Curve → EXPAND**에 있습니다. 레일은 두 탭이 공유하므로 DRFT·ATOMIC
어느 탭에서도 같은 커브를 편집할 수 있고, 편집 결과는 양쪽에 동시에 반영됩니다.
그래프에 찍히는 13개 점은 현재 탭의 스톱을 따릅니다 (DRFT는 98, ATOMIC은 95·5).
ATOMIC 탭에서는 x축 아래에 step 눈금(1000→0)이 함께 표시됩니다 — 팔레트는 왼쪽이 step 0이라
읽는 방향이 반대이기 때문입니다.

포토샵 커브 툴과 같은 조작입니다. 프리셋 상태에서 그래프를 드래그하면 그 커브를 그대로
이어받아 CUSTOM으로 전환됩니다.

- 드래그 = 포인트 이동 / 빈 곳 클릭 = 추가 / 더블클릭·Delete = 삭제 / 방향키 = 미세조정 (Shift ×5)
- 양 끝점은 x 고정, y는 이동 가능
- 보간은 **Fritsch–Carlson monotone cubic** — 오버슈트가 구조적으로 불가능해서 드래그 중
  램프 순서가 뒤집히지 않습니다

## Primary 틴트

그레이에 primary를 얼마나 섞을지를 **grade 번호 기준 선형**으로 계산합니다.

```
alpha = strength × (1 − |grade − 50| / 50)
```

스톱 인덱스가 아니라 grade 숫자로 보간하는 게 핵심입니다 — 100·98·95·90은 2·3·5 간격인데
그 아래는 전부 10 간격이라, 인덱스로 계산하면 값이 어긋납니다.

- **STRENGTH** 슬라이더: 0–60%, grade 50에서의 정점 값. 나머지는 전부 여기에 비례
- **SHAPE**
  - `SHEET` — 소스 시트 원본. 90–99 구간을 정점의 0.2배로 유지
  - `TENT` — 예외 없는 완전 선형 (98 → 0.4%, 95 → 1%)

## ATOMIC SYSTEM 탭

두 번째 taxonomy. **같은 커브 엔진을 다르게 샘플링**한 것이라 컬러 로직은 하나입니다.

```
step = 1000 − grade × 10
step:    0   50  100  200  300  400  500  600  700  800  900  950 1000
grade: 100   95   90   80   70   60   50   40   30   20   10    5    0
```

### Layer 0 — Atomic

| | 내용 |
|---|---|
| **A. Common Palette** | neutral 13스텝. Surface Mode가 `PRIMARY TINT`면 틴트 슬라이더가 그대로 적용됩니다 |
| **B. Opacity** | `0 4 8 12 16 24 32 40 48 56 64 72` (%) |
| **C. Muted Palette** | 11 hue × 11 step. 채도는 CHROMA 슬라이더로 조절 |
| **D. Saturated Palette** | 같은 hue 앵커, 채도 100% |
| **E. Primary Palette** | 입력한 브랜드 색의 램프. 레퍼런스에 없는 우리 시스템만의 레이어 |

hue는 step 500을 앵커로 정의하고(`brown red orange yellow citron sprout green teal blue purple pink`),
거기서 커브를 따라 양쪽으로 램프됩니다. 즉 램프 로직은 DRFT 탭의 primary 램프와 동일하고
피벗만 grade 40 → 50으로 바뀝니다.

`E. Primary Palette`도 step 500 앵커라 `primary/500`이 곧 입력색입니다. DRFT 탭에서는 같은 색이
grade 40에 앉으므로 스와치는 동일하고 라벨만 다릅니다.

### Layer 1 — Semantic / Status / Brand / Overlay · Frost

- **Semantic** — Background(9) · Foreground(6) · Border(6), light/dark 각각
- **Status** — Error·Warning·Success·Info × Background(Base/Active/Tint/Tint Active) + Foreground + Border
- **Brand** — status와 완전히 같은 문법을 primary 팔레트에 적용
- **Overlay** — `neutral/1000` × Opacity 스케일 (subtle 8 · light 16 · medium 32 · default 48 · strong 64)
- **Frost** — light는 `neutral/0`, dark는 `neutral/1000` × (light 48 · default 64 · strong 72)
- Elevation은 제외

### Status는 어느 팔레트에서 뽑을지 고를 수 있습니다

왼쪽 패널의 **STATUS PALETTE** 토글로 `SATURATED`(기본) / `MUTED`를 고릅니다. 고른 쪽이
status 토큰의 원본이 되고, 팔레트 셀의 바인딩 표시와 토큰 참조(`← muted/red/500`)도 함께 옮겨갑니다.

### Status foreground / border는 step이 고정이 아닙니다

같은 step이라도 hue마다 대비가 크게 다릅니다. yellow/600은 자기 tint 위에서 2.58:1로
AA에 한참 못 미치는 반면 blue/600은 6.65:1입니다. 그래서 **자기 tint 대비 기준을 넘는 첫
step을 자동으로 고릅니다** (foreground 4.5:1, border 3:1).

| light | error | warning | success | info |
|---|---|---|---|---|
| foreground | 600 (5.83) | **800** (7.91) | **700** (6.16) | 600 (6.65) |
| border | 500 | 700 | 600 | 500 |

dark는 네 hue 모두 400에서 통과합니다 (최소 4.82:1).

MUTED로 바꾸면 자동 선택 결과도 따라 바뀝니다 — 예를 들어 warning은 muted 쪽이 이미 덜 밝아서
800이 아니라 **700**(6.07:1)에서 통과합니다. 어느 팔레트를 골라도 네 상태 모두 AA를 넘습니다.

### 양방향 링크

DRFT 탭의 Color Source와 동일한 개념입니다. layer 0의 각 스톱 칸에 그 값을 쓰는 토큰이
`L·fg/secondary` / `D·bg/tertiaryActive` 형태로 찍힙니다(L = light, D = dark, 현재 테마는 진하게).

- 토큰 카드의 `← neutral/700` 라벨 클릭 → layer 0으로 스크롤 + 해당 칸 강조
- layer 0 칸 클릭 → 그 값을 쓰는 토큰 카드 전부 강조

바인딩은 `atomicBindings()`가 매핑 테이블을 한 번 순회해 만들고, 양방향 표시·JSON export의
`bindings` 맵이 모두 이 인덱스 하나를 씁니다.

## 그 외

- 라이트/다크 × default/primary-tint 서피스 시스템
- WCAG 대비율 검증 뱃지
- 생성된 토큰만으로 구성한 UI 프리뷰
- 내보내기 — CSS 변수 / Tailwind / JSON / Figma Tokens / SwiftUI (layer 0 램프 포함)
- 팔레트 전체가 URL 해시에 저장되어 링크로 공유됩니다
- 앱 UI 자체가 자기 출력으로 칠해집니다

---

## 소스 시트와 리포 코드의 차이

`Color Source (Do Not Edit)` 시트를 수치로 검증하는 과정에서 배포된 코드와 세 군데가
어긋나는 걸 확인했습니다. `SOURCE` / `REPO` 프리셋으로 둘 다 재현할 수 있게 해뒀습니다.

**1. 그레이스케일 공식** — 시트 13개 값은 `round(2.575g − 2.75)`로 13/13 정확히 재현됩니다.
리포의 `round(g/100 × 255)`는 중간 구간이 1–2단계 밝습니다.

| grade | 90 | 80 | 70 | 60 | 50 | 40 | 30 | 20 | 10 |
|---|---|---|---|---|---|---|---|---|---|
| 시트 | E5E5E5 | CBCBCB | B2B2B2 | 989898 | 7E7E7E | 646464 | 4B4B4B | 313131 | 171717 |
| 리포 | E6E6E6 | CCCCCC | B3B3B3 | 999999 | 808080 | 666666 | 4D4D4D | 333333 | 1A1A1A |

**2. 채널 반올림** — 시트는 버림(floor). `#B3261E`의 90은 시트가 `#F2DAD9`,
리포의 `Math.round`는 `#F2DBDA`.

**3. 다크 outline이 30 grade 밝습니다.** 디자인 파일의 변수를 엔진으로 역산하면
`outline`은 grade 40(`#65686D`), `outlineVariant`는 20(`#353639`)에서 나옵니다. 배포된 코드는
70/50이라 거의 흰색 헤어라인이 됩니다. 둘 다 정확히 30씩 차이나서 드리프트가 아니라 의도된
보정으로 보고 디자인 파일 쪽을 따랐습니다. 라이트 모드와 램프는 영향 없습니다.

**4. 다크 서피스 틴트 알파** — 시트의 알파는 50에서 정점인 선형 텐트인데, 리포의 다크 모드는
`alpha = grade`로 하드코딩돼 있습니다 (`surfaceHighest`: grade 22 → 22%, 텐트 기준으론 4%).
라이트 모드는 텐트와 일치합니다.

## 검증

- `SOURCE` — 그레이 13개 + status 앵커 6개(`#F2DAD9` `#2C0907` `#FDE9D4` `#3D1F00`
  `#DCE9DC` `#0B1F0C`) 전부 시트와 일치
- `REPO` — `#E6E6E6`, `#F2DBDA` / `#2D0A08` 등 배포본과 바이트 일치
- 틴트 strength 10% + SHEET → 시트 알파 13개(`0 2 2 2 4 6 8 10 8 6 4 2 0`)와 일치
- 틴트 strength 0% → 틴트 램프가 순수 그레이 램프와 완전히 일치
- 프리셋 5종 모두 단조 증가, 양 끝이 정확히 `#FFFFFF` / `#000000`

## 디자인 파일과의 대조

기본 상태(primary `#276EF1` · SOURCE 커브 · 틴트 10% · dark · Primary Tint)에서 셸의 계산색이
Figma 변수와 채널당 최대 **Δ4/255** 안에 들어옵니다. primary 3종(`#276ef1` `#dbe6fc` `#ffffff`)은
완전 일치. 즉 디자인 파일은 배포본이 아니라 **교정된 설정**으로 만들어졌습니다.

미리보기 안의 아이콘은 원본이 통짜 PNG라 벡터를 추출할 수 없어 단순 스트로크로 대체했습니다.
정확한 아이콘이 필요하면 Figma에서 아이콘 컴포넌트를 별도 노드로 노출해주세요.

## 라이선스

MIT
