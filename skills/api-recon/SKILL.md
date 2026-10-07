---
name: api-recon
description: 웹사이트 API 인터페이스를 수집할 때 이 skill 을 호출한다.
---

# API Recon (프런트엔드 인터페이스 정찰)

**승인된** 전제하에, 다음을 최대한 완전하게 발견한다: **백엔드 API**(경로·메서드·파라미터·응답 본문), **프런트엔드 라우트**, **UI 기능 트리거 지점**(Tab·팝업·테이블 조작 등).

---

## 경계와 금지 (Agent 필독 · 위반 시 범위 이탈)

이 skill 은 **API / 파라미터 면 정찰만** 수행하며, 취약점 발굴이나 침투 악용 단계가 아니다.

### 작업 경계

| 범위 | 허용 | 금지 |
|---|---|---|
| **목표** | path·method·파라미터·라우트·UI 트리거 지점 열거 | SQLi/XSS/권한우회/브루트포스/fuzz 취약점, 패킷 변조 공격, 파괴적 작업 |
| **인증** | Hook + stub/mock 으로 **클라이언트** 로그인 게이트 우회 | 사용자에게 계정·비밀번호를 요구하거나 추측; 실제 로그인 폼 제출 시도 |
| **런타임** | 자격증명 없이 인터페이스를 hook, mock 응답으로 SPA 를 로그인 후 쉘로 진입 | 실제 백엔드 세션에 의존해야만 진행되는 흐름 |

### 자격증명 없는 동적 분석 (Phase 3 기본)

1. `preload.js` / `runtime_harvest.js` 로 로그인·권한·메뉴 등 bootstrap 인터페이스를 **가로채 stub** 한다;
2. 업무 조회 인터페이스에는 **구조가 올바르고, 업무 코드가 성공이며, 데이터는 비어 있을 수 있는** mock body 를 반환한다;
3. 프런트엔드가 백엔드 없이 또는 401 환경에서도 로그인 후 페이지를 렌더하게 하여, 더 많은 XHR/fetch/WebSocket 을 유발한다;
4. **빈 데이터·빈 테이블·플레이스홀더 UI 는 모두 예상된 것**이다 — 이 때문에 실제 로그인이나 취약점 테스트로 전환하지 말 것.

**한 문장**: mock 으로 프런트엔드 라우트와 컴포넌트 마운트를 펼쳐놓고, **outbound 요청만 기록**한다. 백엔드가 무엇을 반환하는지는 중요하지 않고, 프런트엔드가 **또 어떤 인터페이스를 보내는지**가 중요하다.

### 프로세스 하드 금지

| 금지 | 대체 방법 |
|---|---|
| Phase 1 완료 전 메인 entry `index-*.js` 를 grep/curl/Read 해서 API path 추출 | `OUTDIR/harvest_static.py` 실행 |
| harvest 를 대신하는 `extract_apis.py` 등 스크립트를 직접 작성 | `OUTDIR/harvest_static.py` 를 고친 뒤 재실행 |
| 같은 grep/명령이 2회 이상 실패했는데 계속 반복 | 전략 전환: tool_logs 읽기, harvest 수정, reference 조회 |
| 게이트 A/B 를 건너뛰고 `scripts/` 원본을 바로 실행 | OUTDIR 로 복사한 뒤 목표에 맞게 수정 |
| 실제 사용자명/비밀번호, OTP, OAuth 등 인증 | stub/mock (위 참조) |
| 「실제 데이터를 얻는다」는 이유로 stub 을 건너뛰고 권한우회/주입 테스트 | outbound 만 기록, recon 경계 |
| 삭제·민감 데이터 내보내기·대량 쓰기 등 비가역 작업 | coverage 클릭도 동일 |
| runtime + 동적 열거를 끝내지 않고 모든 페이지·인터페이스를 얻었다고 주장 | 「완료 정의」 참조 또는 한계 표기 |
| 파라미터 트리거 매트릭스 + diff 를 끝내지 않고 모든 파라미터를 파악했다고 주장 | Phase 3b 매트릭스 + Phase 5 diff |
| 단일 runtime 샘플로 필수/선택 추론 | 다중 샘플 diff 또는 검증 규칙/에러 역추론 |

---

## 2층 모델 + 실행 모드

| 층 | 산출물 | 상한 |
|---|---|---|
| **정적**(JS bundle) | 전체 endpoint 경로, 라우트 초안, 패키징 지점 필드 후보 | HTTP 메서드 없음; 파라미터는 Phase 1b 필요; 런타임 조합 URL 누락 |
| **런타임**(활성 세션) | 메서드 + body + 응답 + 동적 URL + WS/SSE; 다중 샘플 diff 로 파라미터 보완 | 페이지가 실제로 렌더돼야 요청 발생; 단일 샘플로 필수/선택 확정 불가 |

| 실행 모드 | 엔진 | 적용 |
|---|---|---|
| **depth** | `runtime_harvest.js`(Puppeteer) | API 목록, METHOD/params/응답 본문, WS/SSE, 재현 가능한 배치 실행 |
| **coverage** | browser + `preload.js` | Tab/팝업/테이블 클릭, 기능 지점 커버리지를 더 깊이 |
| **both** | depth 먼저, 그다음 coverage | 가장 완전, 가장 오래 걸림 |

**파라미터 방법론**(범용 스크립트 없음): path 는 harvest/정규식으로; 파라미터는 **앵커 확장창 + UI 바인딩 체인 + 다중 샘플 diff + 에러 역추론**(grep 레시피는 [reference.md](reference.md) J 절 참조).

---

## 완료 정의

아래를 전부 만족해야 recon 완료라고 주장할 수 있다:

- [ ] **정적**: Phase 1 harvest 가 `api_static.txt`, `routes.txt`, `js/` 산출
- [ ] **런타임**: depth 또는 coverage 중 최소 하나; coverage/both 는 **Hook 적용 + 동적 열거 루프** 필수
- [ ] **쉘 진입**: 업무 path 접근 시 `/login` 이 아님(hash 라우트 주의)
- [ ] **파라미터**: coverage/both 는 파라미터 트리거 매트릭스 + `param_samples.json` 완료; Phase 5 에서 `params_merged.json` 병합
- [ ] **심도**(모듈 페이지가 비어 있으면): Phase 4 권한 트리 복원 후 재실행하여, **module 급 API**(단순 locale/bootstrap 이 아닌)가 나올 때까지
- [ ] **산출물 전달**: Phase 5 산출물 완비(Phase 5 산출물 표 참조); `insert_assets` 로 서비스·엔드포인트 자산 기록

---

## 스크립트와 게이트

`scripts/` 는 참조 템플릿일 뿐이며, 원본을 바로 돌려 최종 결과로 삼는 것은 **금지**.

**규칙**: 먼저 읽기 → 목표에 맞게 수정 → `OUTDIR`(예: `recon/`)에 쓰기 → `CHANGES.md` 기록; 맞지 않으면 방법론대로 재작성하고 구조만 빌린다.

| 게이트 | 시점 | 참조 스크립트 → OUTDIR 사본 | 자주 고치는 항목 |
|---|---|---|---|
| **A(정적)** | Phase 0 후, **처음으로** harvest/spider 돌리기 전 | `harvest_static.py` / `spider_mpa.py` | **대부분 사이트는 기본 regex 로 바로 실행 가능**; manifest/방언이 안 맞을 때만 endpoint 정규식, webpack/Vite `publicPath`, MPA exclude/cookie 수정 |
| **B(런타임)** | Phase 2 후, depth/coverage 돌리기 전 | `runtime_harvest.js` / `preload.js` + `config.json` | Cookie/localStorage 키, neutralize 성공값, stubs, login 정규식, api 접두사, hash/history |

**SPA 강제 순서**(교환 불가; Phase 번호가 「먼저 탐색 후 스크립트」보다 우선):

| 단계 | 필수 | 금지 |
|---|---|---|
| Phase 0 완료 후 | 다음 Bash = `python3 OUTDIR/harvest_static.py <URL> OUTDIR` | 메인 entry `index-*.js` 를 curl/grep/Read(보통 >500KB) |
| 게이트 A | 스크립트 복사 → 필요 시 소폭 수정 → **즉시 실행** | 수작업으로 API 먼저 추출한 뒤 harvest 여부 결정 |
| Phase 1 완료 전 | `wc -l` 로 산출물 검증; 404 면 harvest 수정 후 재시도 | extract 스크립트 수작성; 미다운로드 URL 반복 grep |
| Phase 1b 부터 | grep 은 `OUTDIR/js/*.js` 만 | 메인 bundle 로 harvest 대체 |

- ✅ `harvest_static.py` 복사 → (선택) regex 수정 → **즉시 실행**
- ❌ 메인 bundle curl → 여러 번 grep → 임시 extract 작성 → 마지막에야 harvest
- **MPA**: Phase 0 후 다음 Bash = `python3 OUTDIR/spider_mpa.py ...`

---

## 도구와 출력 제약

| 제약 | 설명 |
|---|---|
| 큰 파일 | >100KB 인 `index-*.js` 는 Read/grep 으로 컨텍스트에 넣는 것 **금지**; OUTDIR 스크립트로 배치 처리 |
| grep 출력 | 반드시 `\| head -20` 또는 `-m 5`; 대화에는 path 요약만 남기고 bundle 조각 붙이지 말 것 |
| 검증 | `wc -l`, `ls \| wc -l` 사용; 디렉터리 전체 Read 금지 |
| regex 초탐 | 선택, ≤1회, ≤50KB 소 chunk 또는 HTML 에만; 정식 정적 분석은 harvest 기준 |
| reference | 레시피/템플릿/트러블슈팅은 [reference.md](reference.md) 참조, 전문을 inline 중복 금지 |

---

## 실행 로드맵

```
Phase 0 분류 + OUTDIR
  → 게이트 A → Phase 1 harvest(★ 즉시 실행 ★)
  → Phase 1b 파라미터 역추적
  → Phase 2 인증 3관문 → config.json
  → 게이트 B → Phase 3 런타임 + 파라미터 매트릭스
  → Phase 4 권한 트리(필요 시) → Phase 3 재실행
  → Phase 5 병합 보고 + insert_assets 로 발견한 모든 서비스·엔드포인트 api 자산을 일괄 삽입. 어떤 경우에도 이미 발견한 자산을 빠뜨리면 안 됨
```

순서대로 체크; **앞 항목을 끝내기 전에는 다음 Phase 진입 불가**.

1. [ ] **Phase 0**: SPA/MPA 초탐; `OUTDIR` 생성 → [Phase 0](#phase-0--분류)
2. [ ] **게이트 A + Phase 1**: 스크립트 복사 → **즉시** harvest → `wc -l` 검증 → [Phase 1](#phase-1--정적)
3. [ ] **Phase 1b**: 앵커 확장창 + 바인딩 층 → `param_candidates.json` → [Phase 1b](#phase-1b--파라미터-역추적)
4. [ ] **Phase 2**: 인증 3관문 → `config.json` → [Phase 2](#phase-2--인증-3관문)
5. [ ] **게이트 B**: runtime 스크립트 조정 → [Phase 3](#phase-3--런타임)
6. [ ] **Phase 3**: depth / coverage / both; 쉘 진입 확인; 파라미터 트리거 매트릭스 → `param_samples.json`
7. [ ] **Phase 4**(필요 시): 권한 트리 → stubs 패치 → Phase 3 재실행 → [Phase 4](#phase-4--권한-트리-복원)
8. [ ] **Phase 5**: 산출물 병합 + 보고 + `insert_assets` → [Phase 5](#phase-5--병합과-보고)

---

## Phase 0 — 분류

진입 HTML 을 가져오고, **`OUTDIR` 생성**(skill 내 `scripts/` 는 수정 금지):

- **SPA**: 빈 쉘 + `<div id=app>` + chunk → Phase 1–5
- **MPA**: SSR + `<form>`, endpoint bundle 없음 → 게이트 A 후:

```bash
python3 recon/spider_mpa.py <BASE_URL> <OUTDIR> [--cookie "session=..."] [--max 300] [--depth 5] [--exclude "logout|delete|destroy"]
```

`forms.txt`, `links.txt`, `api_inline.txt` 산출. SPA 인데 forms ≈ 0 이면 → Phase 1 로 전환.

---

## Phase 1 — 정적

[스크립트와 게이트](#스크립트와-게이트) · [도구와 출력 제약](#도구와-출력-제약) 준수.

```bash
python3 recon/harvest_static.py <BASE_URL> <OUTDIR>
```

harvest: HTML script 파싱 → webpack/Vite manifest → 모든 lazy chunk 다운로드 → `js/`, `api_static.txt`, `routes.txt`, `chunkmap.txt` 산출.

```bash
wc -l OUTDIR/api_static.txt OUTDIR/routes.txt
ls OUTDIR/js | wc -l
```

- chunk 수 vs manifest: 404 면 harvest 를 고쳐 재시도, chunk 를 수작업 curl 로 하나씩 받지 말 것
- `api_static.txt` 가 너무 적음 → OUTDIR 내 endpoint 정규식을 완화한 뒤 재실행(reference 참조)

### Phase 1b — 파라미터 역추적

path 는 Phase 1 에서 온다; 파라미터 필드는 따로 recon 해야 한다. grep 규칙은 [도구와 출력 제약](#도구와-출력-제약) 참조.

**완료 기준**: 중요 인터페이스에 대해 답할 수 있어야 함 — 필드명, 전송 위치, 타입 추론, 필수 여부, 샘플 값, 신뢰도.

#### 1b.0 — 전송 형태

| 형태 | 파라미터 위치 | 정적에서 먼저 볼 것 |
|---|---|---|
| REST JSON | body + query | path 앵커 옆 `(params\|data\|body)\s*:\s*\{` |
| GraphQL | `variables` | gql 템플릿, `$page: Int` |
| 전통 form | urlencoded | `<form>`, `FormData` |
| 파일 업로드 | multipart | `FormData.append` |
| 경로 파라미터 | `/user/:id` | 라우트 테이블 + `useParams` / `$route.params` |
| 암호화/서명 | `sign`/`data` 로 포장 | 암호화 함수 입력인자 Hook(reference D 절) |

산출: 인터페이스마다 `transport: query|json|form|graphql|encrypted` 표기.

#### 1b.1 — 앵커 확장창

알려진 path 를 앵커로 삼아 창을 넓혀 패키징 객체를 찾는다:

```bash
grep -n '"/api/user/list"' OUTDIR/js/*.js | head -20
grep -rhoaE '.{0,120}("/api[^"]+").{0,200}' OUTDIR/js/*.js | head -20
grep -rhoaE '(params|data|body|payload)\s*:\s*\{' OUTDIR/js/*.js | head -20
```

| 래퍼 층 | 파라미터 단서 |
|---|---|
| axios 인스턴스 | `data` / `params` |
| 통합 request | 인터셉터가 전역 필드 주입 |
| OpenAPI 클라이언트 | 생성된 method 시그니처 |
| React Query / SWR | hook 두 번째 인자 |
| Vue composable | composable 입력인자 |

타입 잔재: `yup`/`zod`/rules, `Form.Item name=`, 내장 Swagger.

→ `param_candidates.json`: `{ path, fields[], source: "static-callsite", confidence }`

#### 1b.2 — 바인딩 층

```
Form field → onFinish/handleSubmit → transform → API payload
```

| 바인딩 소스 | 기법 |
|---|---|
| 폼 submit | submit → transform → API 추적 |
| 테이블 검색 | `getFieldsValue()` → `params` |
| 라우트 | `:id` / `?tab=` |
| 인터셉터 | 전역 `tenantId`, 페이지네이션, sign |
| 열거 select | `options` → API 열거값 |

DevTools call stack 에서 `fetch`/`XHR.send` 로부터 위로 패키징 함수를 추적.

#### 1b.3 — 패키징 3문(≠ Phase 2 인증 3관문)

| 질문 | 무엇을 답할지 |
|---|---|
| **조립** | payload 가 어디서 build 되고 transform 흔적은 |
| **검증** | required, pattern, enum |
| **전송** | path / query / body / multipart / 헤더 |

인터셉터 관문(Phase 2)에서 전역 주입 필드(Authorization, `X-Tenant-Id`, sign)도 함께 읽는다.

#### 1b.4 — Phase 3 와의 연결

후보 필드는 정적/바인딩 층에서 온다; **필수/선택/조건 의존**은 Phase 3 파라미터 매트릭스 + diff + Phase 5 에러 역추론으로.

---

## Phase 2 — 인증 3관문

`OUTDIR/js/` 에서 grep(`head` 동반)하여 `config.json` 에 기록(레시피는 reference):

| 관문 | 질문 | 키워드 |
|---|---|---|
| **렌더 관문** | 로그인 여부를 어떻게 판단? | `isLogin`, `getToken`, Cookie/localStorage |
| **인터셉터 관문** | 무엇이 `/login` 으로 튀게 하나? | `response_code`, `errno`, axios interceptor |
| **콘텐츠 관문** | 메뉴/권한은 어디서 오나? | `menu`, `permission`, `role`, `acl`, `routes` |

localStorage 키 이름을 자격증명으로 간주 금지 — chunk/요청 체인에서 확인해야 함.

**출구 = 게이트 B**: 결론을 `config.json` 에 반영하고, `OUTDIR/runtime_harvest.js` / `preload.js` 를 수정.

### Phase 2b — API 관찰(선택)

OUTDIR 내 `preload.js` 로 세션 키 이름, Authorization, 중첩 API URL 을 확인:

| 설정 | 산출 |
|---|---|
| `recordDetail: true` | `__API_RECON_DETAIL__` |
| `observe.xhrHeaders: true` | headers 관찰 |
| `extractUrlsFromResponse: true` | 응답 내 하위 API |
| `observe.storageReads/cookieReads: true` | config 역채움 |
| `neutralizeVueRouter: true` | `__API_RECON_ROUTES__` |

coverage 매 라운드 export: `__API_RECON_LOG__`, `__API_RECON_DETAIL__`, `__API_RECON_ROUTES__`, `__API_RECON_OBSERVE__`.

---

## Phase 3 — 런타임

게이트 B 를 통과했어야 함; [경계와 금지](#경계와-금지-agent-필독--위반-시-범위-이탈) · 자격증명 없는 mock 전략 준수.

`config.json` 에서 `"runtimeMode": "depth" | "coverage" | "both"` 설정(템플릿은 reference).

### Hook 과 stub(depth + coverage 공용)

| 층 | 범위 | 목적 |
|---|---|---|
| L1 정밀 | auth/권한/bootstrap stub | 첫 화면 인증 통과 |
| L2 네거티브 보정 | 모든 JSON 응답 | 미로그인 코드 → 성공 |
| L3 폴백 | L1 미적중 `/api` 등 | 빈 성공 본문, UI 펼침 |

- **depth**: fake auth + `forward` 로 업무 코드 수정 + `stubs`; `routes` 순회(hash/history); `runtime_api.json` 산출
- **coverage**: **document-start** 에 `preload.js` 주입(CDP `addScriptToEvaluateOnNewDocument` 또는 Userscript)

검증: `window.__API_RECON_PRELOAD__` 존재; 업무 path 가 `/login` 으로 돌아가지 않음.

```bash
cd recon && npm install
node runtime_harvest.js config.json
```

### 3b — coverage 동적 열거(필수)

1. 메인 내비/사이드바 — 항목마다 클릭, 네트워크 1–3s 대기
2. Tab — `role=tab`, `.ant-tabs-tab`
3. 테이블 — 첫 행 보기/편집/상세
4. 툴바 — 내보내기, 필터, 신규(**비가역 삭제 회피**)
5. 모듈 진입마다 — API/라우트 병합
6. SPA — `routes.txt` 미커버 path 에 대해 제어된 `pushState`(MPA 금지)

**파라미터 트리거 매트릭스**(필수): 모듈마다 조작 유형별로 한 번씩 기록, **다중 샘플 diff**:

| 조작 | 보통 더 생기는 파라미터 |
|---|---|
| 목록 첫 화면 | 페이지네이션 + 기본 필터 |
| 검색 클릭 | keyword, filter |
| 고급 필터 | 더 많은 optional |
| 신규/편집 | 완전한 entity |
| 배치/내보내기/정렬 | `ids[]`, `exportType`, `sortField` |

**stub 하에서도 outbound body/headers 는 여전히 진짜** — 요청을 기준으로. 녹화 → `scan_raw.json`, `param_samples.json`, `api_detail.json`.

- **Vue**: `neutralizeVueRouter: true` + document-start preload
- **React**: `routes.txt` + 사이드바 클릭 + `pushState`
- **both**: 3a depth 먼저, 그다음 3b coverage

---

## Phase 4 — 권한 트리 복원

**트리거**: 모듈 페이지가 비어 있음 / 라우트마다 bootstrap 만(예: locale) → 콘텐츠 관문 미통과.

| 현상 | 의미 |
|---|---|
| 쉘 진입 성공 | 렌더 관문 + 인터셉터 관문 통과 |
| 사이드바 항목 누락/클릭 시 공백 | stub shape 또는 권한 코드 불완전 |
| 라우트마다 API 동일하고 극소 | `v-if permission` 미통과 |
| `routes.txt` 가 bundle 보다 훨씬 적음 | auth 모듈에서 보완해야 함 |

```bash
grep -rhoaE '"/api[^"]*(permission|perm|role|menu|acl)[^"]*"' OUTDIR/js/*.js | sort -u | head -30
grep -rhoaE 'userRouteAuth|getResultTree|routeMap|routeLink|menuList|authList' OUTDIR/js/*.js | head -20
```

전형 체인: `role_permissions`(flat codes) + `permissions/all`(tree) → `getResultTree` → `userRouteAuth[CODE].url`.

```bash
python3 recon/extract_route_map.py recon/js recon/
python3 recon/build_perm_tree.py recon/js recon/ --config recon/config.json
```

중간 산출물: `route_map.json`, `userRouteAuth.json`, `permissions_tree.json`, `*_stub.json`, `perm_codes_all.txt`.

stub 점검: 외층 `response_code` 가 인터셉터 관문과 일치; flat codes 와 tree 정렬; `routes` 가 `route_map` 의 모든 link 를 커버.

`config.json` 갱신 후 **Phase 3 재실행**. 대형 SPA 는 `waitUntil`, `routeTimeout`, `perRouteMs` 조정 가능(reference A3/I 절).

---

## Phase 5 — 병합과 보고

### 산출물 표

| 파일 | 단계 | 내용 |
|---|---|---|
| `js/`, `api_static.txt`, `routes.txt`, `chunkmap.txt` | 1 | 정적 bundle 과 path |
| `param_candidates.json` | 1b | 정적 파라미터 필드 후보 |
| `config.json` | 2 | 3관문 + runtime 설정 |
| `runtime_api.json` | 3a | depth 상세 녹화(WS/SSE 포함) |
| `param_samples.json`, `scan_raw.json`, `api_detail.json` | 3b | 다중 샘플, 클릭 로그, detail |
| `route_map.json` 등 | 4 | 권한 트리 중간 파일(실행 시) |
| `params_merged.json` | 5 | 병합 파라미터 필드 + 신뢰도 |
| `api_merged.txt` | 5 | `METHOD /path [params] [static\|runtime\|both]` |
| `site_map.json` | 5 | 라우트, API, params, 기능 지점, 한계 |
| **insert_assets** | 5 | 모든 서비스·엔드포인트 자산을 자산 라이브러리에 기록 |

### 5b — 파라미터 병합

`param_samples.json` 에서 diff, **범용 병합 스크립트 없음**. 신뢰도 규칙은 reference J7(상/중/하/트리거 대기) 참조.

### 5c — 에러 역추론

승인 범위 내에서 불완전 요청을 보내 400 을 읽을 수 있음(**파라미터 recon 이지 취약점 테스트 아님**): `field 'x' is required`, 열거 에러 등. `data` 포장, `variables`, 암호화 전 `bizData` 주의.

보고에 명시할 것: runtimeMode, 정적/런타임 API 수, 파라미터 신뢰도, 미커버 모듈, 참조 스크립트 대비 `CHANGES.md` 요약.

`site_map.json` 권장 구조:

```json
{
  "site": "https://example.com",
  "runtimeMode": "both",
  "appType": "vue-spa",
  "routeGuardStrategy": ["nav-neutralize", "L1-auth", "L2-patch", "forward"],
  "apisFromStatic": [],
  "apisFromRuntime": [],
  "apis": [],
  "params": [{ "method": "POST", "path": "/api/user/list", "transport": "json", "fields": [] }],
  "frontendRoutes": [],
  "routesVerifiedByClick": [],
  "featuresTriggered": [],
  "limitations": ""
}
```

더 많은 필드와 grep 레시피는 [reference.md](reference.md) 참조.

---

## 일반 설명

- **프레임워크 무관**: webpack/Vite/Angular lazy load 방법 동일
- **전송**: REST/JSON, GraphQL, WebSocket, SSE; gRPC-web 은 범위 밖
- **SSR**: 클라이언트 fetch 는 녹화 가능; RSC/Server Actions 는 완전 열거 불가
- **사각지대**: JSVMP, WASM, HMAC/mTLS 강검증 → 정적 + 한계 표기
- **파라미터 사각지대**: 조건 연동, hidden params, WASM 패키징 → 「트리거 대기」/「도달 불가」
- **정적은 안전망**: runtime 이 막혀도 정적으로 endpoint 열거 가능

---

## 부가 자료

- Grep 레시피, `config.json` 템플릿, 트러블슈팅, Hook, 파라미터 역추적 J 절, site_map 템플릿: **[reference.md](reference.md)**
- 참조 스크립트 경로는 [스크립트와 게이트](#스크립트와-게이트) 표 참조
