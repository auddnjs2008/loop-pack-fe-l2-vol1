# 10주차 CI 측정 기록

## 1단계 - CI 파이프라인을 측정하고 병목만 줄이기

### Before 측정 조건

측정 대상은 기존 `Quality` workflow다. PR이 열리면 `quality` job 하나가 Ubuntu runner에서 실행되고, 의존성을 설치한 뒤 `pnpm check`를 실행한다.

- workflow: `.github/workflows/quality.yml`
- event: `pull_request`
- runner: `ubuntu-latest`
- Node.js: `.nvmrc` 기준 `24.17.0`
- package manager: `pnpm 10.15.1`
- install: `pnpm install --frozen-lockfile`
- 검증 명령: `pnpm check`
- 측정 커밋: `b3f34851 ci: quality job timeout 설정`

`pnpm check`는 `pnpm test && pnpm lint && pnpm typecheck && pnpm test:e2e`를 실행한다. `pnpm test:e2e` 내부에서 production build와 Playwright E2E가 함께 실행되므로, 현재 CI는 lint, typecheck, test, build, E2E를 모두 포함한다.

### Before Cold 측정

cold는 `Set up Node.js` step에 pnpm cache가 없다고 나온 실행으로 잡았다.

캐시 근거:

```txt
pnpm cache is not found
```

#### Cold Raw Data

| 구분       | Total duration | Quality job | Set up job | Checkout | Set up pnpm | Set up Node.js | Install dependencies | Install Playwright Chromium | Run quality checks | Post Set up Node.js |
| ---------- | -------------- | ----------- | ---------- | -------- | ----------- | -------------- | -------------------- | --------------------------- | ------------------ | ------------------- |
| cold try 1 | 1m 45s         | 1m 42s      | 1s         | 2s       | 3s          | 4s             | 6s                   | 23s                         | 56s                | 5s                  |
| cold try 2 | 1m 49s         | 1m 44s      | 2s         | 4s       | 8s          | 5s             | 9s                   | 25s                         | 44s                | 4s                  |
| cold try 3 | 1m 55s         | 1m 51s      | 1s         | 2s       | 4s          | 7s             | 7s                   | 23s                         | 59s                | 5s                  |

#### Cold Summary

| 항목           | Raw                 | Median | Range         |
| -------------- | ------------------- | ------ | ------------- |
| Total duration | 1m45s, 1m49s, 1m55s | 1m49s  | 1m45s ~ 1m55s |
| Quality job    | 1m42s, 1m44s, 1m51s | 1m44s  | 1m42s ~ 1m51s |
| Run quality    | 56s, 44s, 59s       | 56s    | 44s ~ 59s     |
| Playwright     | 23s, 25s, 23s       | 23s    | 23s ~ 25s     |
| Install deps   | 6s, 9s, 7s          | 7s     | 6s ~ 9s       |

### Before Warm 측정

warm은 `Set up Node.js` step에서 pnpm cache hit과 restore 로그가 나온 실행으로 잡았다.

캐시 근거:

```txt
Cache hit for: node-cache-Linux-x64-pnpm-...
Cache restored successfully
Cache restored from key: node-cache-Linux-x64-pnpm-...
```

#### Warm Raw Data

| 구분       | Total duration | Quality job | Set up job | Checkout | Set up pnpm | Set up Node.js | Install dependencies | Install Playwright Chromium | Run quality checks | Post Set up Node.js |
| ---------- | -------------- | ----------- | ---------- | -------- | ----------- | -------------- | -------------------- | --------------------------- | ------------------ | ------------------- |
| warm try 1 | 1m 50s         | 1m 45s      | 1s         | 1s       | 6s          | 8s             | 2s                   | 27s                         | 57s                | 0s                  |
| warm try 2 | 1m 47s         | 1m 41s      | 0s         | 2s       | 4s          | 8s             | 2s                   | 24s                         | 58s                | 0s                  |
| warm try 3 | 2m 06s         | 2m 00s      | 1s         | 2s       | 3s          | 12s            | 2s                   | 35s                         | 1m 00s             | 0s                  |

#### Warm Summary

| 항목           | Raw                 | Median | Range         |
| -------------- | ------------------- | ------ | ------------- |
| Total duration | 1m50s, 1m47s, 2m06s | 1m50s  | 1m47s ~ 2m06s |
| Quality job    | 1m45s, 1m41s, 2m00s | 1m45s  | 1m41s ~ 2m00s |
| Run quality    | 57s, 58s, 1m00s     | 58s    | 57s ~ 1m00s   |
| Playwright     | 27s, 24s, 35s       | 27s    | 24s ~ 35s     |
| Install deps   | 2s, 2s, 2s          | 2s     | 2s ~ 2s       |

### 병목 지점

가장 긴 step은 모든 실행에서 `Run quality checks`였다.

- cold: 56s, 44s, 59s
- warm: 57s, 58s, 1m00s

두 번째로 긴 step은 `Install Playwright Chromium when used`였다.

- cold: 23s, 25s, 23s
- warm: 27s, 24s, 35s

pnpm cache는 `Install dependencies` 시간을 줄이는 데 효과가 있었다. cold에서는 6~9초가 걸렸고, warm에서는 2초로 줄었다. 하지만 전체 wall-clock 중앙값은 cold 1m49s, warm 1m50s로 거의 차이가 없었다. 현재 병목은 의존성 설치보다 `pnpm check` 내부 검증과 Playwright Chromium 설치에 가깝다.

### Before 결론

현재 workflow는 lint, typecheck, test, build, E2E를 한 job에서 모두 실행한다. pnpm cache hit은 확인했지만, 전체 실행 시간에는 큰 영향을 주지 못했다. Before 기준에서 줄일 후보는 `Run quality checks` 내부를 쪼개 병렬화할 수 있는지, 또는 Playwright Chromium 설치를 E2E job으로 분리해 필요한 경우에만 실행할 수 있는지다.

다만 1단계 최적화에서는 검증을 제거하지 않고, 같은 검증을 유지한 채 병목만 줄여야 한다. 그래서 다음 변경 후보는 `lint`, `typecheck`, `unit test`, `build/e2e`를 독립 job으로 나누는 방식이다.

### 전략 선택

Before에서 지목한 병목은 `Run quality checks`였다. 이 step 안에서 `pnpm test`, `pnpm lint`, `pnpm typecheck`, `pnpm test:e2e`가 직렬로 실행되므로 1단계에서는 job 병렬화를 적용한다.

| 전략                       | 적용 여부 | 판단 근거                                                                                                                                               |
| -------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| job 병렬화                 | 적용      | `Run quality checks`가 cold/warm 모두 최장 step이었다. 독립적인 검증을 job으로 나누면 같은 검증을 유지하면서 직렬 시간을 줄일 수 있다.                  |
| `concurrency` 그룹         | 제외      | 같은 PR에 연속 push가 쌓이는 문제를 줄이는 전략이다. 이번 Before 측정의 병목은 단일 실행 내부의 직렬 검증이므로 직접적인 wall-clock 개선 전략은 아니다. |
| setup-node/pnpm store 캐시 | 제외      | 이미 적용되어 있고 warm 실행에서 cache hit이 확인됐다. `Install dependencies`는 warm 기준 2초라 현재 병목이 아니다.                                     |
| path filter                | 제외      | 안 돌려도 되는 검증을 제외하는 조건부 실행 전략이므로 2단계에서 다룬다. 1단계에서는 돌리기로 한 검증을 더 빠르게 만드는 데 집중한다.                    |

job을 병렬화하면 각 job에서 `pnpm install --frozen-lockfile`이 반복된다. 다만 warm 기준 `Install dependencies` 중앙값이 2초였기 때문에 이 반복은 현재 wall-clock 병목으로 보지 않았다. 대신 `Install Playwright Chromium when used`는 24~35초로 상대적으로 크므로 E2E job에서만 실행하도록 분리한다.

`concurrency`를 나중에 적용한다면 main push 실행까지 취소하지 않도록 `group: ${{ github.workflow }}-${{ github.ref }}`처럼 ref를 포함해야 한다.

### 캐시 hit 자가 검증

캐시가 실제로 hit 되는지 확인하려고 정상 lockfile 상태와 lockfile hash를 일부러 바꾼 상태를 각각 실행했다.

#### Warm hit 확인

정상 lockfile 상태의 warm 실행에서는 `Set up Node.js` step에 pnpm store cache 복원 로그가 남았다.

```txt
Cache hit for: node-cache-Linux-x64-pnpm-0f6fe750c5f71cdd5aa0c6b0f5fa1379245d0dc4ba658cbdc88391dc29ba5cc
Cache restored successfully
Cache restored from key: node-cache-Linux-x64-pnpm-0f6fe750c5f71cdd5aa0c6b0f5fa1379245d0dc4ba658cbdc88391dc29ba5cc
```

- `Set up Node.js`: 12s
- `Install dependencies`: 2s

#### Cache miss 재현

`pnpm-lock.yaml` 맨 위에 실험용 주석을 추가해 lockfile hash만 바꿨다. 이 변경은 `9efc3270 test: cache miss 재현` 커밋으로 PR에 올려 Actions를 실행했다.

```diff
+# cache-miss experiment
 lockfileVersion: '9.0'
```

해당 실행의 `Set up Node.js` step에서는 pnpm store cache miss가 찍혔다.

```txt
pnpm cache is not found
```

- `Total duration`: 1m45s
- `Quality job`: 1m41s
- `Set up Node.js`: 6s
- `Install dependencies`: 8s

실험 후 `1cf4ffcd chore: cache miss 실험 원복` 커밋으로 `pnpm-lock.yaml`을 원래 상태로 되돌렸다.

#### Hit/Miss 비교

| 조건 | 캐시 로그                  | Set up Node.js | Install dependencies |
| ---- | -------------------------- | -------------- | -------------------- |
| hit  | `Cache restored from key:` | 12s            | 2s                   |
| miss | `pnpm cache is not found`  | 6s             | 8s                   |

cache hit에서는 pnpm store가 복원되어 `Install dependencies`가 2초로 끝났다. cache miss에서는 store를 복원하지 못해 `Install dependencies`가 8초로 늘어났다. 다만 cache hit일 때는 `Set up Node.js` step 안에서 약 200MB cache archive를 내려받고 압축까지 해제한다. 이 시간이 setup step에 포함되기 때문에, 이번 실행에서는 `Set up Node.js + Install dependencies` 합산 시간이 hit과 miss 모두 14초로 같았다. pnpm store cache의 동작은 확인했지만, 현재 wall-clock 병목은 cache 복원이 아니라 `Run quality checks`와 Playwright Chromium 설치다.

### 적용 내용

### After 측정

### Before/After 비교

## 2단계 - 조건부 실행 설계

### 대상 변경 범위

### 실행 조건

### 검증 결과

## 3단계 - 예산 게이트와 결과 표시

### 예산 기준

### 적용 내용

### 검증 결과

## 4단계 - AI 코드리뷰 활용

### 리뷰 대상

### AI 피드백

### 반영 결과

## 회고
