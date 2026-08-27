# DRFT AI — Color Engine

[drft_1](https://github.com/minus31/drft_1)의 컬러 엔진만 떼어내 만든 단일 파일 웹앱.
의존성이 없고 빌드도 필요 없습니다 — `index.html`을 브라우저로 열면 끝입니다.

원본 로직: `components/DesignSystemLogic/Colors/colorSystem.ts` + `colorStore.ts`

---

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

**3. 다크 서피스 틴트 알파** — 시트의 알파는 50에서 정점인 선형 텐트인데, 리포의 다크 모드는
`alpha = grade`로 하드코딩돼 있습니다 (`surfaceHighest`: grade 22 → 22%, 텐트 기준으론 4%).
라이트 모드는 텐트와 일치합니다.

## 검증

- `SOURCE` — 그레이 13개 + status 앵커 6개(`#F2DAD9` `#2C0907` `#FDE9D4` `#3D1F00`
  `#DCE9DC` `#0B1F0C`) 전부 시트와 일치
- `REPO` — `#E6E6E6`, `#F2DBDA` / `#2D0A08` 등 배포본과 바이트 일치
- 틴트 strength 10% + SHEET → 시트 알파 13개(`0 2 2 2 4 6 8 10 8 6 4 2 0`)와 일치
- 틴트 strength 0% → 틴트 램프가 순수 그레이 램프와 완전히 일치
- 프리셋 5종 모두 단조 증가, 양 끝이 정확히 `#FFFFFF` / `#000000`

## 라이선스

MIT
