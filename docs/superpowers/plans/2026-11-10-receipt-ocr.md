# 게임 기록 영수증 OCR 앱 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 블루 아카이브(한국 서버) 기록 영수증 사진 15~20장을 기기 안에서 인식해, 영수증의 모든 값을 고정된 구조의 기록 JSON으로 만드는 무료 안드로이드 앱(웹은 나중)을 만든다.

**Architecture:** DOM 없는 공용 코어(`src/core/`)가 행 재조립 → 사진 이어 붙이기 → 섹션 해석 → 이름 매칭 → 빠진 줄 감지를 모두 처리한다. 앱은 Capacitor로 감싼 바닐라 TypeScript 웹앱이고, 촬영·사진 선택·OCR(ML Kit 한국어)·파일 저장은 자체 Java 플러그인이 맡는다. 연도별 영수증 차이는 읽기 양식 JSON과 기본 데이터(사전) JSON으로만 처리하고, 기록 JSON의 모양(기록 계약)은 코어에 고정한다.

**Tech Stack:** Vite, TypeScript(바닐라), Vitest, Ajv(CI·도구 전용, 기록 검증은 standalone 생성 코드), es-hangul, idb-keyval, Capacitor 8, Google ML Kit(text-recognition-korean 번들형, document-scanner), Cloudflare Pages, GitHub Actions. 웹 단계에서 @paddleocr/paddleocr-js.

**Spec:** `docs/superpowers/specs/2026-11-10-receipt-lists-design.md` (이하 "설계"). 실행자는 이 계획과 설계를 함께 읽는다.

**이 문서의 위치:** Claude Docs의 「게임 기록 영수증 OCR 도구 — 설계 및 구현 계획서」를 설계에 맞춰 다시 쓴 것이다. 이제 이 파일이 원본이다. Task 번호는 실행 순서대로 새로 매겼으므로 예전 문서의 번호와 다르다.

**실행 방법:** 단계마다 새 세션에서 "계획서의 N단계를 진행해"로 실행한다. 각 단계는 따로 테스트할 수 있는 결과를 낸다.

## 진행 순서

| 단계 | 내용 | Task | 선행 조건 |
|---|---|---|---|
| 1 | 공용 코어 | 1.1 ~ 1.15 | 없음 (가짜 데이터로 진행) |
| 2 | 앱 뼈대 | 2.1 ~ 2.2 | 1단계 |
| 3 | 앱 처리 흐름·기록 | 3.1 ~ 3.5 | 2단계 |
| 4 | 템플릿·기본 데이터·원격 갱신 | 4.1 ~ 4.5 | 3단계 |
| 5 | 앱 공개 | 5.1 ~ 5.7 | 4단계 + **원본 해상도 영수증 사진** |
| 6 | 디지털 영수증 | (자리만) | 5단계 + 글꼴·이미지 조각 결정 |
| 7 | 웹 | 7.1 ~ 7.4 | 6단계 |

## Global Constraints

모든 Task는 아래를 기본으로 지킨다. 1.1에서 저장소 루트 `CLAUDE.md`의 「5. 지킬 것」에 옮긴다(이미 있는 공개 저장소·버전 규칙은 유지).

- **서버 없음:** 백엔드, DB, 회원 계정을 만들지 않는다. 배포물은 정적 파일과 APK뿐이다.
- **사진 비전송·비저장:** 원본 사진은 기기 메모리와 OS 임시 파일에서만 다룬다. IndexedDB와 내보내는 파일에는 OCR 결과(OcrLine)와 기록만 저장하고 이미지·Blob·ImageBitmap은 저장하지 않는다.
- **AI는 선택 기능:** 키가 없어도 모든 기능이 동작한다. AI 재확인은 사용자가 버튼을 눌렀을 때 review·missing 값의 잘라낸 행 이미지만 보낸다. `private` 값과 그 값이 든 행은 절대 보내지 않는다.
- **키 처리(BYOK):** 사용자 본인의 Gemini 키를 `generativelanguage.googleapis.com`으로 직접 보낸다. 기본은 메모리 보관, "이 기기에 기억하기"를 켰을 때만 localStorage. 키는 `x-goog-api-key` 헤더로만 보낸다.
- **Gemini 모델 이름 하드코딩 금지:** 모델 목록 API로 고른다. 버전 숫자가 없는 별칭은 자동 선택에서 뺀다. 설정에서 수동 지정 가능.
- **지원 환경:** 안드로이드 앱 먼저(minSdk 29, Android 10). 웹(크롬·엣지·웨일·삼성 인터넷 최신)은 7단계. iOS는 지원하지 않는다.
- **언어:** UI는 한국어. 인식 대상은 한글, 영문, 숫자, 기호.
- **배포:** 앱은 GitHub Releases의 서명된 APK 하나(arm64-v8a·armeabi-v7a). 웹·템플릿·사전은 Cloudflare Pages 무료 플랜. 플레이스토어에는 올리지 않는다.
- **자동 백업 끔:** `android:allowBackup="false"`, `dataExtractionRules`는 모두 제외.
- **외부 CDN 금지(웹):** 라이브러리·모델·WASM은 같은 도메인에서 제공한다. 패키지 버전은 `package-lock.json`으로 고정한다.
- **파일 크기:** Cloudflare Pages 배포 파일 하나당 25MiB 미만.
- **1년 방치 전제:** 매년 바뀌는 것은 `public/templates/`(읽기 양식)와 `public/dict/`(기본 데이터)로 처리한다. 영수증에 기록 계약에 없는 내용이 생길 때만 코드(`src/core/record.ts`)를 고친다.
- **`src/core/`는 DOM 없이:** 브라우저 API 없이 순수 TypeScript로 쓰고 Node(Vitest)에서 테스트한다.
- **개인정보:** 실제 영수증 사진, 닉네임, 서클명은 저장소에 커밋하지 않는다. fixture는 익명화한 OCR 텍스트만 넣는다.
- **비밀값:** API 키와 서명 키스토어는 저장소에 넣지 않는다(.gitignore, GitHub Secrets).
- **공개 저장소:** 커밋은 `.githooks/pre-commit`을 통과해야 한다. `--no-verify` 금지. 커밋 작성자 이메일은 GitHub noreply 주소다.
- **버전:** A.B.CDE(예: `1.2.34`, 핫픽스 `1.2.34a`). push·커밋으로는 올리지 않고, 릴리스 때 사용자 확인을 받아 올린다.
- **기본 데이터:** SchaleDB에서 변환한 데이터는 소유자가 사용 허락을 확인하기 전까지 커밋하지 않는다.
- **앱 공개 조건:** 원본 해상도 영수증 사진으로 실측(Task 5.1)을 마치기 전에는 APK를 공개 릴리스하지 않는다.
- **공식 로고·이미지:** 사용자 정보가 전혀 없는 부분만 잘라 `public/receipt-art/`에 둔다(6단계). 앱과 README에 "넥슨·블루 아카이브" 출처와 비공식 팬메이드 도구임을 적는다.

## Review Focus

사용자가 가장 먼저 부딪힐 다섯 가지다. 각 줄의 테스트는 해당 Task에 들어 있다.

| 상황 | 기대 동작 | 고정하는 테스트 |
|---|---|---|
| 한 글자 차이 이름이 애매하게 읽힘 (`세리타` → 세리카? 세리나?) | 조용히 고르지 않고 `review` + `dict_ambiguous`, 후보 둘 다 보여 줌 | Task 1.4 `ambiguous` |
| 사진끼리 반복되는 줄(같은 스킬 줄)만 겹침 | 겹침으로 보지 않고(오병합 금지) 이음매 경고 | Task 1.9 `no anchor` |
| 사진을 추가하거나 순서를 바꿈 | 사용자가 고친 값이 원래 학생에 그대로 붙어 있음, 못 붙인 수정은 따로 보임 | Task 1.13 `reattach` · `stale` |
| 깨진 백업 파일이나 다른 앱의 JSON을 복원 | 거절하고 기존 기록은 그대로 | Task 3.4 `rejects invalid` |
| 다른 해 영수증이나 영수증이 아닌 사진 | 최신 양식을 억지로 쓰지 않고 `templateId: null` → 사용자에게 선택 요청 | Task 1.7 `adjacent year` · `null` |

---

## 파일 구조

```
receipt-ocr/ (저장소 루트)
├─ CLAUDE.md, README.md, LICENSE, THIRD_PARTY_NOTICES.md
├─ docs/  ANNUAL.md, spike-ocr.md, superpowers/{specs,plans}/
├─ public/
│  ├─ _headers                     4.4에서 CORS, 7.3에서 COOP/COEP/CSP
│  ├─ schema/record.schema.json    기록 계약 JSON Schema
│  ├─ templates/ index.json, schema.json, sample.json(index 미등록), 2026.json(5.2)
│  ├─ dict/<name>.json             기본 데이터 (SchaleDB 변환본은 허락 뒤 커밋)
│  └─ receipt-art/<연도>/          공식 이미지 조각 (6단계)
├─ src/
│  ├─ core/                        DOM 없음
│  │  ├─ types.ts                  OcrLine, Row, Seam, Correction, ReviewEntry, Warning, DictEntry, Template, SectionSpec…
│  │  ├─ record.ts                 기록 계약 타입, emptyRecordData, parseRecordFile
│  │  ├─ generated/validateRecord.js(+ .d.ts)  scripts/gen-validators.mjs가 생성
│  │  ├─ config.ts                 임계값
│  │  ├─ hangul.ts                 자모, 유사도, containsFuzzy, fuzzySpan
│  │  ├─ normalize.ts              숫자·날짜·기간·스킬·enum·값 정규화
│  │  ├─ names.ts                  matchName (사전 매칭)
│  │  ├─ templates.ts              isSupported, detectTemplate
│  │  ├─ validate.ts               validateTemplate (Ajv, CI·도구·테스트 전용)
│  │  ├─ rows.ts                   dedupeLines, assembleRows
│  │  ├─ merge.ts                  mergePhotos
│  │  ├─ parse/                    index.ts(parseReceipt), fields.ts, list.ts, records.ts, ranking.ts, letter.ts, corrections.ts
│  │  ├─ checks.ts                 findGaps
│  │  └─ export.ts                 toMaskedLines, maskRecord
│  ├─ engines/  types.ts, index.ts, mlkit.ts (paddle.ts는 7단계)
│  ├─ plugin/receipt.ts            Java 플러그인 TS 정의
│  ├─ llm/  types.ts, keyStore.ts, gemini.ts
│  ├─ store/  session.ts, records.ts
│  ├─ platform/  templateSource.ts, updateCheck.ts
│  ├─ ui/  upload.ts, order.ts, review.ts, records.ts, settings.ts
│  ├─ pipeline.ts                  사진 처리, 다시 계산
│  └─ main.ts
├─ tools/template-maker.html, template-maker.ts   개발 서버 전용
├─ tests/  core/, store/, llm/, platform/, scripts/, fixtures/<templateId>/<case>/
├─ scripts/  check-templates.mjs, check-assets.mjs, gen-validators.mjs, import-schaledb.mjs
├─ android/                        Capacitor + ReceiptPlugin.java
└─ .github/  workflows/ci.yml, release-android.yml, ISSUE_TEMPLATE/bug.md
```

의존 방향: `ui → pipeline·store·engines·llm·platform·plugin → core`. core는 다른 폴더를 가져오지 않는다.

## 핵심 인터페이스

모든 Task는 아래 이름과 타입을 그대로 쓴다.

### 공용 타입 (`src/core/types.ts`)
```ts
export type Point = [number, number];
export interface OcrLine { text: string; score: number; box: Point[]; photoId: string }

export interface Row { rowKey: string; text: string; parts: OcrLine[]; photoId: string; y: number; score: number }
export interface Seam { afterPhotoId: string; beforePhotoId: string; overlapRows: number }

export type Status = 'ok' | 'review' | 'missing';
export type Reason = 'low_score' | 'conflict' | 'corrected' | 'out_of_range' | 'bad_format' | 'not_found'
  | 'edited' | 'llm_read' | 'llm_rejected' | 'dict_ambiguous' | 'not_in_dict' | 'truncated' | 'duplicate_ref';
export interface NameValue { printed: string; ref: string | null }
export interface Candidate { ref: string; name: string; score: number }
export interface ReviewEntry { status: Status; reasons: Reason[]; rowKey?: string; candidates?: Candidate[] }
export interface Unparsed { rowKey: string; section: string; text: string; photoId: string }
export type Correction =
  | { rowKey: string; path: string; set: unknown }     // path = '<sectionKey>.<valueKey>', 예: 'students.name'
  | { rowKey: string; text: string };                  // 해석 못 한 행의 텍스트 수정
export type Warning =
  | { kind: 'seam_no_overlap'; afterPhotoId: string; beforePhotoId: string }
  | { kind: 'count_mismatch'; section: string; expected: number; found: number }
  | { kind: 'season_gap'; section: string; after: number }
  | { kind: 'not_descending'; section: string; index: number }
  | { kind: 'record_incomplete'; section: string; rowKey: string }
  | { kind: 'duplicate_part'; section: string; rowKey: string };

export interface DictEntry { id: string; name: string; aliases?: string[] }
export type Dicts = Record<string, DictEntry[]>;

export type ValueType = 'number' | 'text' | 'date' | 'datetime' | 'duration' | 'enum' | 'skills' | 'name';
export interface ValueSpec {
  type: ValueType; dict?: string; enum?: string[]; list?: boolean; private?: boolean;
  range?: [number, number]; pattern?: string;   // pattern: text 값의 형식 검사 (어긋나면 bad_format)
}
export interface FieldSpec extends ValueSpec { key: string; label: string; aliases?: string[]; value: 'right' | 'below' }
export interface RowSpec { pattern: string; values: Record<string, ValueSpec> }
export interface SentenceSpec { pattern: string; values: Record<string, ValueSpec> }
export type SectionSpec =
  | { key: string; kind: 'fields'; header?: string; fields: FieldSpec[] }
  | { key: string; kind: 'list'; header: string; rows: RowSpec[] }
  | { key: string; kind: 'records'; header: string; start: RowSpec; parts: RowSpec[] }
  | { key: string; kind: 'ranking'; header: string; row: RowSpec }
  | { key: string; kind: 'letter'; header: string; sentences: SentenceSpec[] };
export interface Template { schemaVersion: 1; id: string; title: string; year: number; detect: string[];
  ignore?: string[]; sections: SectionSpec[] }
export interface TemplateIndex {
  schemaVersion: 1;
  templates: { id: string; title: string; year: number; file: string }[];
  dicts: Record<string, string>;
}
export interface DetectResult { templateId: string | null; ranking: { id: string; score: number }[] }
```
- 예전 계획의 `Template.fields`는 `kind: 'fields'` 섹션으로 옮겼다. `value: 'inline'`은 행 재조립 뒤 `'right'`와 같아서 없앴다.
- 예전 `OcrLine.imageIndex`는 `photoId`(가져올 때 붙는 바뀌지 않는 문자열)로 바꿨다.
- 정규식(`pattern`, `ignore`)은 `row.text`(연속 공백만 하나로 줄인 원문)에 적용한다. `normalizeLabel`은 라벨·섹션 머리말·detect 문구 비교에만 쓴다.
- `Template.ignore`: 데이터가 없는 행(점선, 학생부 안내 줄, 로고 글자, 꼬리 문구, 대응하는 섹션이 없는 큰 헤더)을 건너뛰는 정규식 목록. 섹션 머리말 판정 다음, 해석·이어 붙이기 전에 적용한다. 맞은 행은 데이터도 unparsed도 아니다.
- 사전 파일(`public/dict/<이름>.json`, 내용은 `DictEntry[]`): `students`, `gifts`, `bosses`, `limitBreak`, `mainStories`, `events`, `miniStories`, `groupStories`, `collabItems`. 스토리는 목록마다 다른 파일이다(설계 5.4를 이렇게 정함).

### 기록 계약 (`src/core/record.ts`)
```ts
export const RECORD_SCHEMA_VERSION = 1;
export type SkillValue = 'MAX' | number;
type N = number | null;
export interface RecordData {
  header: { printedAt: string | null; nickname: string | null; firstDay: string | null; daysTogether: N;
            playTime: { hours: number; minutes: number } | null; level: N; studentCount: N; circle: string | null };
  students: { name: NameValue; level: number; star: number; weaponStar: N; skills: SkillValue[] | null; momotalkEp: N }[];
  favorites: { rank: number; names: NameValue[]; affection: number }[];
  totalAssault: { season: number; boss: NameValue; tier: string }[];
  grandAssault: { season: number; boss: NameValue; tier: string }[];
  counters: { schedule: N; circleAttendance: N; crafting: N; bounty: N; specialCommission: N; schoolExchange: N };
  tacticalExam: { clears: N; bestScore: N; totalScore: N };
  limitBreak: { season: number; boss: NameValue; stage: number }[];
  missions: { normal: N; normalSweeps: N; hard: N; hardSweeps: N };
  mainStory: { title: NameValue; chapter: number; episode: number }[];
  events: { title: NameValue; episodes: number }[];
  miniStories: { title: NameValue; episodes: number }[];
  groupStories: { title: NameValue; episodes: number }[];
  collabItems: { name: NameValue; owned: number }[];
  office: { collectionCards: N; collectionRate: N; cosmetics: N; cosmeticsRate: N; shaleCoins: N };
  currency: { pyroxeneSpent: N; creditsSpent: N; apSpent: N };
  gifts: { name: NameValue; uses: number }[];
  consumption: { eligma: N; raidCoins: N; raidCoinsHigh: N; grandCoins: N; grandCoinsHigh: N;
                 tacticalCoins: N; examCoins: N; mastery: N; recruit1: N; recruit10: N };
  letter: { nickname: string | null; years: N; schedule: N; storiesRead: N; lastStory: NameValue | null;
            lastEpisode: string | null; topBattle: NameValue | null; raidScore: N; grandScore: N;
            limitBreakBest: N; examBest: N; ap: N; directMissions: N; sweeps: N; newStudents: N;
            firstStudent: NameValue | null; latestStudent: NameValue | null; topGift: NameValue | null;
            topGiftCount: N; credits: N; pyroxene: N; year: N };
}
export interface ReceiptRecord {
  schemaVersion: 1; recordId: string; readingTemplate: string; savedAt: string;
  data: RecordData; review: Record<string, ReviewEntry>; unparsed: Unparsed[];
  corrections: Correction[]; warnings: Warning[];
}
export function emptyRecordData(): RecordData;   // 목록은 [], 단일 값은 null
export function parseRecordFile(text: string): { ok: true; record: ReceiptRecord } | { ok: false; errors: string[] };
```
- `private` 값: `header.nickname`, `header.circle`, `letter.nickname`.
- 리뷰 경로: 목록 값은 `'<section>[i].<key>'`(예: `students[12].name`), 단일 값은 `'<section>.<key>'`, 최애 순위의 이름 하나는 `'favorites[i].names[j]'`. Correction의 `path`는 `'<section>.<key>'`(순위 이름은 `'favorites.names[j]'`).

### 코어 함수 (각 Task에서 만든다)
- `hangul.ts`: `toJamo`, `similarity`, `normalizeLabel`, `containsFuzzy(hay, needle): number`, `fuzzySpan(text, label): { score: number; end: number }`
- `normalize.ts`: `normalizeNumber(raw): { value: number | null; changed: boolean }`, `normalizeDate(raw): string | null`, `normalizeDateTime(raw): string | null`, `normalizeDuration(raw): { hours: number; minutes: number } | null`, `normalizeSkills(raw): SkillValue[] | null`, `normalizeEnum(raw, options): string | null`, `normalizeValue(spec, raw, ctx): ValueResult`
  - `interface MatchContext { dicts: Dicts; prefer?: Record<string, string[]> }` — prefer: dict 이름 → 먼저 찾을 ref 목록
  - `interface ValueResult { value: unknown; status: Status; reasons: Reason[]; candidates?: Candidate[] }`
- `names.ts`: `matchName(raw, entries, opts?: { prefer?: string[] }): { value: NameValue; status: Status; reasons: Reason[]; candidates: Candidate[] }`
- `templates.ts`: `isSupported(x: { schemaVersion: number }): boolean`, `detectTemplate(lines: OcrLine[], templates: Template[]): DetectResult`
- `validate.ts`: `validateTemplate(json: unknown): { ok: true; template: Template } | { ok: false; errors: string[] }`
- `rows.ts`: `dedupeLines(lines): OcrLine[]`, `assembleRows(lines: OcrLine[], photoId: string): Row[]`
- `merge.ts`: `mergePhotos(rowsPerPhoto: Row[][]): { rows: Row[]; seams: Seam[]; conflicts: string[] }`
- `parse/index.ts`: `parseReceipt(t: Template, rows: Row[], dicts: Dicts, corrections: Correction[], conflicts: string[] = []): ParseResult` — `conflicts`는 `mergePhotos`가 돌려준 rowKey 목록, 그 행에서 나온 값은 `review` + `conflict`
  - `interface ParseResult { data: RecordData; review: Record<string, ReviewEntry>; unparsed: Unparsed[]; warnings: Warning[]; staleCorrections: Correction[]; privateRowKeys: string[] }`
- `checks.ts`: `findGaps(data: RecordData, seams: Seam[]): Warning[]`
- `export.ts`: `toMaskedLines(rows: Row[], privateRowKeys: string[]): string`, `maskRecord(r: ReceiptRecord): ReceiptRecord`

### 임계값 (`src/core/config.ts`, 초기값 — fixture와 실측으로 조정)

| 상수 | 초기값 | 쓰임 |
|---|---|---|
| `LABEL_MIN_SIM` | 0.75 | 라벨·섹션 머리말·detect 문구 인정 |
| `DICT_MIN_SIM` | 0.7 | 이 미만이면 `not_in_dict` |
| `DICT_OK_SIM` | 0.9 | 이 미만이면 `corrected`로 review |
| `DICT_MARGIN` | 0.1 | 1·2등 차이가 이 미만이면 `dict_ambiguous` |
| `OCR_REVIEW_SCORE` | 0.9 | 행 score가 이 미만이면 `low_score` |
| `DETECT_MIN_MATCH` | 2 | 판별에 필요한 detect 문구 수 |
| `ROW_OVERLAP` | 0.5 | 같은 행 판정(세로 겹침 비율) |
| `MERGE_MIN_SIM` | 0.85 | 사진 사이 겹친 행 판정 |
| `IMAGE_MAX_PIXELS` | 3,000,000 | 웹 단계 이미지 축소 |
| `TILE_HEIGHT` / `TILE_OVERLAP` | 1600 / 120 px | 웹 단계 세로 분할 |
| `LLM_BATCH` | 15 | AI 요청 하나에 넣는 값 수 |

### 테스트 헬퍼 (`tests/helpers.ts`, Task 1.1에서 만들고 1.5에서 `validRecord()`를 더한다)
- `L(text, x, y, w = 120, h = 20, score = 0.99, photoId = 'p1'): OcrLine` — 네 꼭짓점 박스
- `rowsOf(texts: string[], photoId = 'p1', score = 0.99): Row[]` — `rowKey`는 `${photoId}:r${i}`
- `SAMPLE_DICTS: Dicts` — `students: [{id:'s1',name:'세리카'},{id:'s2',name:'세리나'},{id:'s3',name:'하나코(수영복)'},{id:'s4',name:'미카'},{id:'s5',name:'미나'}]`, `bosses: [{id:'b1',name:'KAITEN FX Mk.0(시가전)'},{id:'b2',name:'KAITEN FX Mk.0(야전)'},{id:'b3',name:'비나(시가전)'},{id:'b4',name:'비나(야전)'}]`, `gifts: [{id:'g1',name:'고전 시집'}]`

---

## 1단계 — 공용 코어

UI 없이 `src/core/`만 만든다. 끝나면 손으로 만든 OcrLine(가짜 사진 여러 장)을 넣었을 때 행 재조립 → 이어 붙이기 → 해석 → 경고까지 테스트로 검증된다.

### Task 1.1: 프로젝트 뼈대

**Files:**
- Create: `package.json`, `tsconfig.json`, `vite.config.ts`, `index.html`, `src/main.ts`, `src/core/types.ts`, `src/core/config.ts`, `tests/helpers.ts`, `.github/workflows/ci.yml`
- Modify: `CLAUDE.md`(「3. 스택·실행」, 「5. 지킬 것」에 Global Constraints), `.gitignore`(빠진 항목만)

- [ ] **Step 1:** Vite `vanilla-ts` 구성을 직접 만든다(폴더가 비어 있지 않으므로 create-vite 대화형 대신 파일을 쓴다). `vitest`, `ajv`, `es-hangul`, `idb-keyval`를 설치한다. `package.json`에 `"private": true`, `version`은 `"0.0.0"`으로 두고 앱 버전은 쓰지 않는다(버전은 태그에서 온다).
- [ ] **Step 2:** `tsconfig.json`: `strict`, `noUncheckedIndexedAccess`, `resolveJsonModule`. `vite.config.ts`: 빌드 입력은 `index.html` 하나(`tools/` 제외), Vitest 환경 `node`.
- [ ] **Step 3:** `src/core/types.ts`에 「핵심 인터페이스 · 공용 타입」을, `src/core/config.ts`에 「임계값」 표 전부를 쓰고, `tests/helpers.ts`에 `L`, `rowsOf`, `SAMPLE_DICTS`를 만든다(「테스트 헬퍼」 참고).
- [ ] **Step 4:** `ci.yml`: push·PR마다 `npm ci` → `npm test` → `npm run build`.
- [ ] **Step 5:** CLAUDE.md 「3. 스택·실행」에 스택과 `npm run dev / test / build`, 「5. 지킬 것」에 Global Constraints를 옮긴다.
- [ ] **Step 6: 확인** — `npm run build`와 `npx vitest run --passWithNoTests`가 둘 다 0으로 끝난다. `npx tsc --noEmit`이 0.
- [ ] **Step 7: Commit** `chore: project scaffold`

### Task 1.2: 한글 유틸

**Files:** Create `src/core/hangul.ts` · Test `tests/core/hangul.test.ts`

**Interfaces:** Produces `toJamo(s): string`, `similarity(a, b): number`(자모 기준 1 − 편집거리/긴 쪽 길이), `normalizeLabel(s): string`(공백·문장부호 제거, 영문 소문자), `containsFuzzy(hay, needle): number`(needle 길이 ±1 음절 창의 최대 similarity), `fuzzySpan(text, label): { score; end }`(원문 `text`를 normalizeLabel한 뒤 label과 가장 비슷한 구간을 찾아 그 구간이 원문에서 끝나는 위치)

- [ ] **Step 1: 실패하는 테스트**
```ts
expect(toJamo('량')).toBe('ㄹㅑㅇ');
expect(similarity('플레이일수', '플레이일수')).toBe(1);
expect(similarity('플레이일슈', '플레이일수')).toBeGreaterThanOrEqual(0.75);
expect(similarity('량', '랑')).toBeGreaterThan(similarity('량', '곰'));
expect(normalizeLabel(' 플레이 일 수: ')).toBe('플레이일수');
expect(normalizeLabel('SAMPLE Receipt')).toBe('samplereceipt');
expect(containsFuzzy('3주넌기록영수증', '3주년')).toBeGreaterThanOrEqual(0.75);
expect(containsFuzzy('가입일', '플레이일수')).toBeLessThan(0.75);
expect(fuzzySpan('플레이 일수 1,234', '플레이일수')).toMatchObject({ score: 1, end: 6 });
expect(fuzzySpan('• 스킬 성장  EX. MAX', '스킬성장').end).toBe(7);
```
- [ ] **Step 2:** `npx vitest run tests/core/hangul.test.ts` → FAIL
- [ ] **Step 3: 구현** — 자모 분해는 es-hangul, 편집거리는 레벤슈타인 직접 구현. fuzzySpan은 정규화 문자열 위치 → 원문 위치 대응표를 만들어 끝 위치를 돌려준다.
- [ ] **Step 4:** 같은 명령 → PASS
- [ ] **Step 5: Commit** `feat(core): hangul similarity and fuzzy span`

### Task 1.3: 값 정규화

**Files:** Create `src/core/normalize.ts` · Test `tests/core/normalize.test.ts`

**Interfaces:** Consumes `similarity`, `normalizeLabel`(1.2), config(1.1). Produces 핵심 인터페이스의 `normalize.ts` 함수 전부. `type: 'name'` 분기는 Task 1.4가 연결한다(이 Task에서는 name을 받으면 `not_in_dict`로 둔다). `normalizeEnum`은 `normalizeLabel`한 입력과 각 선택지의 `similarity`가 가장 높은 것을 고르고, `LABEL_MIN_SIM` 미만이면 `null`.

- [ ] **Step 1: 실패하는 테스트** — 숫자 칸 혼동 문자 치환표: `O o D → 0`, `l I | → 1`, `Z → 2`, `S s → 5`, `B → 8`, `g → 9`. 숫자가 하나 이상 섞인 첫 덩어리만 치환하고 쉼표 제거는 `changed`로 치지 않는다.
```ts
expect(normalizeNumber('1,234회')).toEqual({ value: 1234, changed: false });
expect(normalizeNumber('1,2O4회')).toEqual({ value: 1204, changed: true });
expect(normalizeNumber('청휘석 2,000개').value).toBe(2000);
expect(normalizeNumber('없음').value).toBeNull();
expect(normalizeDate('2026.03.07')).toBe('2026-03-07');
expect(normalizeDate('2026년 3월 7일')).toBe('2026-03-07');
expect(normalizeDate('3월 7일')).toBeNull();
expect(normalizeDateTime('2026-11-07, 15:53')).toBe('2026-11-07T15:53');
expect(normalizeDuration('1,234시간 5분')).toEqual({ hours: 1234, minutes: 5 });
expect(normalizeSkills('EX. MAX / Lv. 7 / Lv.1 / MAX')).toEqual(['MAX', 7, 1, 'MAX']);
expect(normalizeSkills('EX, Lv. 1 | Lv. 2 I Lv.1 / LV.7')).toEqual([1, 2, 1, 7]);
expect(normalizeSkills('EX. MAX / MAX / MAX')).toBeNull();   // 4칸이 아니면 null
expect(normalizeEnum('플래티님', ['실버', '골드', '플래티넘'])).toBe('플래티넘');
expect(normalizeEnum('zzz', ['실버', '골드'])).toBeNull();
const ctx = { dicts: {} };
expect(normalizeValue({ type: 'number', range: [0, 3650] }, '9999', ctx).reasons).toContain('out_of_range');
expect(normalizeValue({ type: 'number' }, '1,2O4', ctx)).toMatchObject({ value: 1204, status: 'review', reasons: ['corrected'] });
expect(normalizeValue({ type: 'text', pattern: '^\\d+시간 \\d+분$' }, '12시 5분', ctx).reasons).toContain('bad_format');
expect(normalizeValue({ type: 'skills' }, 'EX. MAX', ctx)).toMatchObject({ value: null, status: 'review', reasons: ['bad_format'] });
```
- [ ] **Step 2:** FAIL 확인 → **Step 3:** 구현 → **Step 4:** PASS (`npx vitest run tests/core/normalize.test.ts`)
- [ ] **Step 5: Commit** `feat(core): value normalization`

### Task 1.4: 이름 매칭

**Files:** Create `src/core/names.ts` · Modify `src/core/normalize.ts`(`type: 'name'` → `matchName(raw, ctx.dicts[spec.dict] ?? [], { prefer: ctx.prefer?.[spec.dict] })`, `list: true`면 쉼표로 나눠 각각) · Test `tests/core/names.test.ts`

**Interfaces:** Consumes `similarity`, config. Produces `matchName`.

규칙(설계 5.5): 비교 전 공백 정리·`(` 앞 공백 무시·라틴 소문자화. 1등 점수 < `DICT_MIN_SIM` → `ref: null`, `printed` = 정리한 OCR 문자열, `review` + `not_in_dict`. 그 밖에는 `ref` = 1등. 1등 점수 < `DICT_OK_SIM`이면 `corrected`, 1·2등 차이 < `DICT_MARGIN`이면 `dict_ambiguous`; 둘 중 하나라도 있으면 `review`, 없으면 `ok`. `ok`일 때 `printed` = 사전 `name`. 끝이 `...` 또는 `…`인 입력은 앞부분과 앞부분이 맞는 항목만 후보로 하고, 하나면 `printed` = 사전 이름을 인쇄된 앞부분 길이에서 자르고 `...`, 여럿이면 `ref: null` + `ok` + `truncated`. `prefer`가 있으면 그 ref들 중에서 먼저 찾고(1·2등 차이도 그 안에서 잰다), 그 안에서 `DICT_MIN_SIM` 이상이 없으면 전체에서 찾는다. `candidates`는 상위 3개. `printed`의 "정리한 OCR 문자열"은 연속 공백 정리와 `(` 앞 공백 제거만 한다(대소문자 유지, 소문자화는 비교용). 잘린 입력의 앞부분과 맞는 항목이 하나도 없으면 `ref: null`, `review` + `not_in_dict`, `printed` = 정리한 OCR 문자열.

- [ ] **Step 1: 실패하는 테스트** (`SAMPLE_DICTS`는 `tests/helpers.ts`)
```ts
const S = SAMPLE_DICTS.students, B = SAMPLE_DICTS.bosses;
expect(matchName('세리카', S)).toMatchObject({ value: { printed: '세리카', ref: 's1' }, status: 'ok', reasons: [] });
expect(matchName('하나코 (수영복)', S).value).toEqual({ printed: '하나코(수영복)', ref: 's3' });
// ambiguous: 세리타는 세리카·세리나와 같은 거리
const amb = matchName('세리타', S);
expect(amb).toMatchObject({ status: 'review', reasons: expect.arrayContaining(['dict_ambiguous']) });
expect(amb.candidates.map(c => c.ref)).toEqual(expect.arrayContaining(['s1', 's2']));
expect(matchName('zzz', S)).toMatchObject({ value: { printed: 'zzz', ref: null }, status: 'review', reasons: ['not_in_dict'] });
// prefer: 학생부에 미카만 있으면 미타 → 미카 (corrected), 없으면 미카·미나 사이 dict_ambiguous
expect(matchName('미타', S).reasons).toContain('dict_ambiguous');
expect(matchName('미타', S, { prefer: ['s4'] })).toMatchObject({ value: { ref: 's4' }, status: 'review', reasons: ['corrected'] });
// truncated
expect(matchName('KAITEN FX Mk.0(시...', B)).toMatchObject({ value: { printed: 'KAITEN FX Mk.0(시...', ref: 'b1' }, status: 'ok' });
expect(matchName('KAITEN FX Mk.0(...', B)).toMatchObject({ value: { printed: 'KAITEN FX Mk.0(...', ref: null }, status: 'ok', reasons: ['truncated'] });
expect(matchName('ZZZ(...', B)).toMatchObject({ value: { printed: 'ZZZ(...', ref: null }, status: 'review', reasons: ['not_in_dict'] });
// normalizeValue 연결
expect(normalizeValue({ type: 'name', dict: 'students' }, '세리타', { dicts: SAMPLE_DICTS }).reasons).toContain('dict_ambiguous');
```
- [ ] **Step 2:** FAIL → **Step 3:** 구현 → **Step 4:** PASS (`npx vitest run tests/core/names.test.ts`)
- [ ] **Step 5: Commit** `feat(core): dictionary name matching`

### Task 1.5: 기록 계약과 검증 코드 생성

**Files:**
- Create: `src/core/record.ts`, `public/schema/record.schema.json`(JSON Schema 2020-12. 키가 고정된 객체는 `additionalProperties: false`, `review` 같은 맵은 `additionalProperties: { $ref: ReviewEntry }`), `scripts/gen-validators.mjs`, `src/core/generated/validateRecord.js` + `validateRecord.d.ts`(생성물, 커밋함)
- Modify: `package.json`(`"gen:validators"`), `.github/workflows/ci.yml`(`npm run gen:validators && git diff --exit-code src/core/generated`)
- Modify: `tests/helpers.ts`(`validRecord()` 추가: 아래 `valid`와 같은 최소 기록)
- Test: `tests/core/record.test.ts`

**Interfaces:** Produces 기록 계약 전부, `emptyRecordData()`, `parseRecordFile(text)`.

- [ ] **Step 1: 실패하는 테스트** (`valid()`는 `emptyRecordData()`로 만든 최소 기록)
```ts
const valid = () => ({ schemaVersion: 1, recordId: 'r1', readingTemplate: 'sample', savedAt: '2026-11-10T12:00:00+09:00',
  data: emptyRecordData(), review: {}, unparsed: [], corrections: [], warnings: [] });
expect(parseRecordFile(JSON.stringify(valid())).ok).toBe(true);
expect(parseRecordFile('not json')).toEqual({ ok: false, errors: ['not json'] });
const noId = valid(); delete (noId as any).recordId;
expect((parseRecordFile(JSON.stringify(noId)) as any).errors.join()).toContain('recordId');
expect(parseRecordFile(JSON.stringify({ ...valid(), schemaVersion: 2 }))).toEqual({ ok: false, errors: ['unsupported schemaVersion 2'] });
const badStudents = valid(); (badStudents.data as any).students = {};
expect(parseRecordFile(JSON.stringify(badStudents)).ok).toBe(false);
expect(Object.keys(emptyRecordData()).sort()).toEqual(['collabItems','consumption','counters','currency','events','favorites','gifts','grandAssault','groupStories','header','letter','limitBreak','mainStory','miniStories','missions','office','students','tacticalExam','totalAssault']);
```
- [ ] **Step 2:** FAIL → **Step 3:** 구현 — `gen-validators.mjs`는 Ajv(`ajv/dist/2020`)의 standalone 모드로 `record.schema.json`을 ESM 모듈로 만든다. `parseRecordFile`은 JSON 파싱 → `schemaVersion` 확인 → 생성된 검증기 순서로 검사한다. 앱·웹 코드는 `ajv`를 import하지 않는다.
- [ ] **Step 4:** PASS, `npm run gen:validators && git diff --exit-code src/core/generated` → 0
- [ ] **Step 5: Commit** `feat(core): record contract and generated validator`

### Task 1.6: 읽기 양식 스키마와 검사

**Files:**
- Create: `public/templates/schema.json`, `public/templates/sample.json`, `public/templates/index.json`(sample 미등록, `templates: []`, `dicts: {}`), `src/core/validate.ts`, `src/core/templates.ts`(`isSupported`), `scripts/check-templates.ts`
- Modify: `package.json`(devDependency `tsx`, `"check:templates": "tsx scripts/check-templates.ts"`), `ci.yml`(test 뒤)
- Test: `tests/core/templates.test.ts`

**Interfaces:** Produces `validateTemplate`, `isSupported`. `validate.ts`는 Ajv를 쓰므로 scripts·tools·테스트에서만 import한다(Ajv는 `new Function`을 써서 웹 CSP에 막힌다).

`validateTemplate`이 스키마 외에 추가로 검사하는 것: 섹션 `key` 중복, 섹션 `key`가 기록 계약(`record.schema.json`의 `data.properties`)에 있는지, 각 `values`/`fields`의 키가 그 섹션 항목의 속성에 있는지, `type: 'name'`에 `dict`가 있는지, 모든 `pattern`과 `ignore`가 정규식으로 컴파일되는지, `enum` 타입에 `enum` 목록이 있는지.

`sample.json`(가상 양식, 실제 키 사용):
```json
{ "schemaVersion": 1, "id": "sample", "title": "예시 기록 영수증", "year": 2000,
  "detect": ["예시주년", "SAMPLE RECEIPT"],
  "ignore": ["^[-=]{3,}$", "정보 순서"],
  "sections": [
    { "key": "header", "kind": "fields", "fields": [
      { "key": "nickname", "label": "닉네임", "type": "text", "private": true, "value": "right" },
      { "key": "firstDay", "label": "가입일", "type": "date", "value": "right" },
      { "key": "daysTogether", "label": "함께한 날", "type": "number", "range": [0, 20000], "value": "right" },
      { "key": "studentCount", "label": "학생 수", "type": "number", "value": "right" } ] },
    { "key": "students", "kind": "records", "header": "학생 기록",
      "start": { "pattern": "^(?<name>.+?)\\s*\\(\\s*L[vV][.,]?\\s*(?<level>\\d{1,2})\\s*[,.]\\s*(?<star>[1-5])\\s*[성신](?:\\s*무기\\s*(?<weaponStar>[1-4])\\s*[성신])?\\s*\\)?$",
                 "values": { "name": { "type": "name", "dict": "students" }, "level": { "type": "number", "range": [1, 90] },
                             "star": { "type": "number", "range": [1, 5] }, "weaponStar": { "type": "number", "range": [1, 4] } } },
      "parts": [
        { "pattern": "^[•·]?\\s*스킬\\s*(?<skills>EX.+)$", "values": { "skills": { "type": "skills" } } },
        { "pattern": "^[•·]?\\s*모모톡\\s*EP\\s*(?<momotalkEp>\\d+)", "values": { "momotalkEp": { "type": "number" } } } ] },
    { "key": "favorites", "kind": "ranking", "header": "최애 순위",
      "row": { "pattern": "^★?\\s*(?<rank>\\d+)\\s*등\\s*★?\\s*\\.\\s*(?<names>.+?)\\s*\\(\\s*호감도\\s*:\\s*(?<affection>\\d+)\\s*\\)$",
               "values": { "rank": { "type": "number" }, "names": { "type": "name", "dict": "students", "list": true }, "affection": { "type": "number" } } } },
    { "key": "totalAssault", "kind": "list", "header": "총력전 트로피",
      "rows": [ { "pattern": "^[•·]?\\s*시즌\\s*(?<season>\\d+)\\s*-\\s*(?<boss>.+?)\\s+(?<tier>\\S+)$",
                  "values": { "season": { "type": "number" }, "boss": { "type": "name", "dict": "bosses" },
                              "tier": { "type": "enum", "enum": ["브론즈", "실버", "골드", "플래티넘"] } } } ] },
    { "key": "counters", "kind": "fields", "fields": [
      { "key": "schedule", "label": "스케줄 횟수", "type": "number", "value": "right" } ] },
    { "key": "letter", "kind": "letter", "header": "교무일지",
      "sentences": [
        { "pattern": "올해 스케줄을\\s*(?<schedule>[\\d,]+)\\s*번", "values": { "schedule": { "type": "number" } } },
        { "pattern": "가장 많이 건네신 선물은\\s*「(?<topGift>[^」]+)」", "values": { "topGift": { "type": "name", "dict": "gifts" } } } ] } ] }
```

- [ ] **Step 1: 실패하는 테스트** (`withSections(...)`은 sample 복사본의 sections를 바꾸는 헬퍼)
```ts
expect(validateTemplate(sample).ok).toBe(true);
expect(errorsOf({ ...sample, sections: undefined }).join()).toContain('sections');
expect(errorsOf(withSections([students, students]))).toContain('duplicate section key: students');
expect(errorsOf(withSections([{ ...students, key: 'foo' }]))).toContain('unknown section key: foo');
expect(errorsOf(withSections([withValue(students, 'bogus', { type: 'number' })]))).toContain('unknown value key: students.bogus');
expect(errorsOf(withSections([withValue(students, 'name', { type: 'name' })]))).toContain('dict required: students.name');
expect(errorsOf(withSections([{ ...totalAssault, rows: [{ pattern: '(', values: {} }] }]))).toContain('invalid pattern: totalAssault.rows[0]');
expect(errorsOf({ ...sample, ignore: ['('] })).toContain('invalid pattern: ignore[0]');
expect(isSupported({ schemaVersion: 1 })).toBe(true);
expect(isSupported({ schemaVersion: 2 })).toBe(false);
```
- [ ] **Step 2:** FAIL → **Step 3:** 구현 — `check-templates.ts`: `public/templates/`의 모든 `.json`(`index.json`·`schema.json` 제외)을 `validateTemplate`로 검사하고, `index.json`이 가리키는 템플릿·사전 파일이 있는지 확인한다. 사전 파일마다 `id` 중복과 공백을 지우면 같아지는 `name`이 있으면 오류. 실패 시 종료 코드 1.
- [ ] **Step 4:** PASS. 확인: sample.json의 `sections`를 지우고 `npm run check:templates` → 1, 되돌리면 0
- [ ] **Step 5: Commit** `feat(core): reading template schema and validation`

### Task 1.7: 템플릿 판별

**Files:** Modify `src/core/templates.ts` · Test `tests/core/detect.test.ts`

**Interfaces:** Consumes `containsFuzzy`, `normalizeLabel`. Produces `detectTemplate`.

점수 = `containsFuzzy ≥ LABEL_MIN_SIM`으로 맞은 detect 문구 개수. **숫자가 든 detect 문구는 그 문구의 숫자열이 `normalizeLabel(줄 텍스트)` 안에 그대로 들어 있어야** 맞은 것으로 친다(퍼지 비교만 쓰면 `2주년`↔`3주년`, `2025`↔`2026`도 기준을 넘는다). 최고 점수가 `min(DETECT_MIN_MATCH, detect.length)` 이상일 때만 `templateId`를 정하고, 동점이면 `year`가 큰 쪽.

- [ ] **Step 1: 실패하는 테스트** (A: 2025, `['2주년','2025']` / B: 2026, `['3주년','2026']`, `lines(...)`은 문자열 → OcrLine[])
```ts
expect(detectTemplate(lines('3주년 기록', '2026.11.01'), [A, B]).templateId).toBe('B');
expect(detectTemplate(lines('3주넌 기록', '2026'), [A, B]).templateId).toBe('B');
expect(detectTemplate(lines('영수증'), [A, B]).templateId).toBeNull();
expect(detectTemplate(lines('2주년 3주년', '2025 2026'), [A, B]).templateId).toBe('B');
expect(detectTemplate(lines('3주년', '2026'), [A, B]).ranking.map(r => r.id)).toEqual(['B', 'A']);
// adjacent year
expect(detectTemplate(lines('2주년 기록', '2025.11.01'), [A, B]).templateId).toBe('A');
// null: 등록 안 된 해
expect(detectTemplate(lines('2주년 기록', '2025.11.01'), [B]).templateId).toBeNull();
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): template detection`

### Task 1.8: 행 재조립

**Files:** Create `src/core/rows.ts` · Test `tests/core/rows.test.ts`

**Interfaces:** Produces `dedupeLines`, `assembleRows`.

`assembleRows`: 사진 하나의 OcrLine을 세로 중심이 `ROW_OVERLAP × 높이` 안에서 겹치면 한 행으로 묶고, 행 안은 x 순서로 공백 하나를 끼워 잇는다. 행은 y 순서. `rowKey = '${photoId}:r${행 순번}'`(입력 순서와 상관없이 같은 값). `score` = parts의 최소 score. `dedupeLines`: 같은 사진에서 텍스트가 같고 박스 IoU > 0.5인 줄은 score가 높은 하나만 남긴다(웹 타일 겹침).

- [ ] **Step 1: 실패하는 테스트**
```ts
const a = assembleRows([L('• 스킬 성장', 10, 100), L('EX. MAX / MAX / MAX / MAX', 300, 102, 200)], 'p1');
expect(a).toHaveLength(1);
expect(a[0]).toMatchObject({ rowKey: 'p1:r0', text: '• 스킬 성장 EX. MAX / MAX / MAX / MAX', photoId: 'p1' });
expect(assembleRows([L('값', 300, 100), L('라벨', 10, 100)], 'p1')[0].text).toBe('라벨 값');
expect(assembleRows([L('A', 10, 100), L('B', 10, 130)], 'p1').map(r => r.rowKey)).toEqual(['p1:r0', 'p1:r1']);
expect(assembleRows([L('B', 10, 130), L('A', 10, 100)], 'p1').map(r => r.text)).toEqual(['A', 'B']);
expect(assembleRows([L('A', 10, 100, 120, 20, 0.6), L('B', 300, 101)], 'p1')[0].score).toBe(0.6);
expect(dedupeLines([L('A', 0, 0), L('A', 2, 1, 120, 20, 0.8)])).toHaveLength(1);
expect(dedupeLines([L('A', 0, 0), L('A', 0, 0, 120, 20, 0.99, 'p2')])).toHaveLength(2);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): row assembly`

### Task 1.9: 사진 이어 붙이기

**Files:** Create `src/core/merge.ts` · Test `tests/core/merge.test.ts`

**Interfaces:** Consumes `Row`, `similarity`, `normalizeLabel`. Produces `mergePhotos`.

알고리즘(설계 6.3): **같은 행**이란 정규화 텍스트 similarity ≥ `MERGE_MIN_SIM`이면서, 숫자 혼동 문자(`normalizeNumber`와 같은 표)를 바꾼 뒤 뽑은 숫자열이 같은 두 행이다(`시즌 32`와 `시즌 33`, `19화`와 `11화`는 다른 행). 이웃한 사진 i, i+1에서 i의 끝 k행과 i+1의 첫 k행이 모두 같은 행인 가장 큰 k를 찾는다. **겹침 구간에 두 사진 안에서 각각 한 번만 나오는 행(앵커, "같은 행" 관계로 셈)이 하나도 없으면 겹침으로 보지 않는다**(같은 스킬 줄처럼 반복되는 줄로 오병합하지 않기 위해). 겹친 행은 score가 높은 쪽을 남기고, 정규화 텍스트가 완전히 같지 않으면 그 rowKey를 `conflicts`에 넣는다. 이름(키)으로는 합치지 않는다.

- [ ] **Step 1: 실패하는 테스트**
```ts
const p1 = rowsOf(['이름A (Lv. 90, 5성)', '• 스킬 EX. MAX / MAX / MAX / MAX', '이름B (Lv. 1, 3성)'], 'p1');
const p2 = rowsOf(['이름B (Lv. 1, 3성)', '• 스킬 EX. Lv. 1 / Lv. 1 / Lv.1 / Lv. 1', '이름C (Lv. 2, 3성)'], 'p2');
const m = mergePhotos([p1, p2]);
expect(m.rows.map(r => r.text)).toEqual([p1[0].text, p1[1].text, p1[2].text, p2[1].text, p2[2].text]);
expect(m.seams).toEqual([{ afterPhotoId: 'p1', beforePhotoId: 'p2', overlapRows: 1 }]);
// no overlap
expect(mergePhotos([rowsOf(['a1', 'a2'], 'p1'), rowsOf(['b1', 'b2'], 'p2')]).seams[0].overlapRows).toBe(0);
// no anchor: 반복 줄만 겹치면 겹침 아님
const skill = '• 스킬 EX. MAX / MAX / MAX / MAX';
const n = mergePhotos([rowsOf(['이름A (Lv. 9, 5성)', skill], 'p1'), rowsOf([skill, '이름B (Lv. 9, 5성)', skill], 'p2')]);
expect(n.seams[0].overlapRows).toBe(0);
expect(n.rows.filter(r => r.text === skill)).toHaveLength(3);
// conflict: 겹친 행이 다르게 읽힘 → score 높은 쪽 + conflicts
const c = mergePhotos([rowsOf(['• 고전 시집 1,611회'], 'p1', 0.95), rowsOf(['• 고전 시집 1,6l1회', 'x'], 'p2', 0.7)]);
expect(c.rows[0].text).toBe('• 고전 시집 1,611회');
expect(c.conflicts).toEqual(['p1:r0']);
// 같은 이름의 다른 행은 남는다
expect(mergePhotos([rowsOf(['• Serenade Promenade 19화 완료'], 'p1'), rowsOf(['• Serenade Promenade 11화 완료'], 'p2')]).rows).toHaveLength(2);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): merge photos by row overlap`

### Task 1.10: 섹션 나누기 · fields · list · unparsed

**Files:** Create `src/core/parse/index.ts`, `src/core/parse/fields.ts`, `src/core/parse/list.ts` · Test `tests/core/parse-basic.test.ts`

**Interfaces:** Consumes `normalizeValue`, `fuzzySpan`, `containsFuzzy`, `emptyRecordData`. Produces `parseReceipt`(이 Task에서는 fields·list와 섹션 전환·ignore·conflicts·unparsed·review·privateRowKeys. records·ranking·letter·corrections는 1.11~1.13에서 붙인다).

동작:
- **섹션 전환:** 행의 `normalizeLabel` 텍스트가 어떤 섹션 `header`와 `LABEL_MIN_SIM` 이상이면 현재 섹션을 바꾼다(머리말 행 자체는 데이터가 아니다). 첫 머리말 전의 행은 `header`가 없는 첫 섹션(`kind: 'fields'`)에 속한다. 현재 섹션 규칙에 맞지 않는 행이 **머리말 없는 다른 `kind: 'fields'` 섹션의 필드 라벨**과 `LABEL_MIN_SIM` 이상으로 맞으면 그 섹션으로 바꾸고 그 행을 거기서 해석한다(실물 영수증의 횟수·소모량 묶음은 목록 바로 뒤에 머리말 없이 온다).
- **ignore:** 머리말 판정 다음, 해석 전에 `Template.ignore`에 맞는 행은 건너뛴다.
- **패턴 적용 대상:** `row.text`(연속 공백만 정리). 
- **fields:** 필드를 템플릿 순서대로 처리한다. 필드마다 그 섹션에서 아직 쓰이지 않은 행 중 `fuzzySpan(row.text, label).score ≥ LABEL_MIN_SIM`인 가장 높은 행을 쓴다(동점이면 앞 행). 한 행에 여러 라벨이 맞으면 점수가 높은 라벨, 같으면 긴 라벨이 가져간다. `fuzzySpan` 안에서 동점 구간은 가장 앞 구간. `'right'`는 같은 행의 라벨 뒤, `'below'`는 다음 행이 값이다. 못 찾은 필드는 `missing` + `not_found`. list: 각 행을 `rows` 패턴 순서대로 시도한다. 어느 섹션에서든 규칙에 맞지 않는 행은 `unparsed`(fields 섹션의 라벨 없는 행 포함). 행 score < `OCR_REVIEW_SCORE`면 그 행에서 나온 값 모두 `low_score`, rowKey가 `conflicts`에 있으면 `conflict`. `review` 항목에는 값이 나온 행의 `rowKey`를 넣는다(못 찾은 값은 없음). `private` 값이 나온 행의 rowKey는 `privateRowKeys`.

- [ ] **Step 1: 실패하는 테스트**
```ts
const r = parseReceipt(sample, rowsOf(['닉네임 실제닉네임', '가입일 2024.01.01', '함께한 날 1,000일',
  '총력전 트로피', '• 시즌 1 - 비나(시가전) 골드', '• 시즌 2 - 비나(야전) 플래티님', '• 알 수 없는 줄']), SAMPLE_DICTS, []);
expect(r.data.header).toMatchObject({ nickname: '실제닉네임', firstDay: '2024-01-01', daysTogether: 1000, studentCount: null });
expect(r.review['header.studentCount']).toMatchObject({ status: 'missing', reasons: ['not_found'] });
expect(r.data.totalAssault).toEqual([
  { season: 1, boss: { printed: '비나(시가전)', ref: 'b3' }, tier: '골드' },
  { season: 2, boss: { printed: '비나(야전)', ref: 'b4' }, tier: '플래티넘' }]);
expect(r.unparsed).toEqual([{ rowKey: 'p1:r6', section: 'totalAssault', text: '• 알 수 없는 줄', photoId: 'p1' }]);
expect(r.privateRowKeys).toEqual(['p1:r0']);
const low = parseReceipt(sample, rowsOf(['총력전 트로피', '• 시즌 3 - 비나(야전) 골드'], 'p1', 0.6), SAMPLE_DICTS, []);
expect(low.review['totalAssault[0].tier']).toMatchObject({ status: 'review', reasons: expect.arrayContaining(['low_score']) });
// 머리말 없는 fields 섹션으로 전환 + ignore
const sw = parseReceipt(sample, rowsOf(['총력전 트로피', '• 시즌 1 - 비나(시가전) 골드', '스케줄 횟수 12회', '-----']), SAMPLE_DICTS, []);
expect(sw.data.counters.schedule).toBe(12);
expect(sw.unparsed).toEqual([]);
// conflicts
const cf = parseReceipt(sample, rowsOf(['총력전 트로피', '• 시즌 1 - 비나(시가전) 골드']), SAMPLE_DICTS, [], ['p1:r1']);
expect(cf.review['totalAssault[0].season'].reasons).toContain('conflict');
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS (`npx vitest run tests/core`)
- [ ] **Step 5: Commit** `feat(core): parse sections, fields and lists`

### Task 1.11: records · ranking

**Files:** Create `src/core/parse/records.ts`, `src/core/parse/ranking.ts` · Modify `src/core/parse/index.ts` · Test `tests/core/parse-records.test.ts`

**Interfaces:** Consumes 1.10, `matchName`(prefer). Produces records·ranking 해석.

동작: records — `start`가 맞는 행이 새 레코드를 연다. `parts` 중 하나가 맞는 행은 현재 레코드에 붙는다. 레코드가 열리기 전의 part 행은 `unparsed`. 한 레코드에 같은 part가 두 번이면 `duplicate_part` 경고(뒤엣것 무시). 레코드 안에 없는 part 값은 `null`. 첫 part(`skills`)가 없는 레코드는 `record_incomplete` 경고(rowKey는 start 행). 학생부에서 두 레코드가 같은 `ref`면 둘 다 `review` + `duplicate_ref`. ranking — 버퍼 규칙: 행이 단독으로 `row.pattern`에 맞으면, 남아 있던 버퍼를 `unparsed`로 보내고 그 행을 확정한다. 맞지 않으면 버퍼 + 행(공백 없이 이음)을 시도해 맞으면 확정하고 버퍼를 비우며, 아니면 버퍼에 쌓는다. 섹션이 끝날 때 남은 버퍼는 `unparsed`. `names`는 쉼표로 나눠 각각 `matchName`(prefer = 이미 해석된 `students`의 ref 목록), review 경로는 `favorites[i].names[j]`. 교무일지의 학생 이름도 같은 prefer를 쓴다(1.12).

- [ ] **Step 1: 실패하는 테스트**
```ts
const r = parseReceipt(sample, rowsOf(['학생 기록', '세리카 (Lv. 90, 5성 무기 2성)', '• 스킬 EX. MAX / MAX / Lv.7 / MAX', '• 모모톡 EP 3',
  '세리나 (Lv. 1, 3성)', '• 스킬 EX. Lv. 1 / Lv. 1 / Lv.1 / Lv. 1',
  '최애 순위', '★1등★. 세리카 (호감도: 100)', '2등. 세리나, 하나코(수', '영복) (호감도: 50)']), SAMPLE_DICTS, []);
expect(r.data.students).toEqual([
  { name: { printed: '세리카', ref: 's1' }, level: 90, star: 5, weaponStar: 2, skills: ['MAX', 'MAX', 7, 'MAX'], momotalkEp: 3 },
  { name: { printed: '세리나', ref: 's2' }, level: 1, star: 3, weaponStar: null, skills: [1, 1, 1, 1], momotalkEp: null }]);
expect(r.data.favorites).toEqual([
  { rank: 1, names: [{ printed: '세리카', ref: 's1' }], affection: 100 },
  { rank: 2, names: [{ printed: '세리나', ref: 's2' }, { printed: '하나코(수영복)', ref: 's3' }], affection: 50 }]);
const dup = parseReceipt(sample, rowsOf(['학생 기록', '세리카 (Lv. 9, 5성)', '• 스킬 EX. MAX / MAX / MAX / MAX', '• 스킬 EX. MAX / MAX / MAX / MAX', '세리카 (Lv. 9, 5성)']), SAMPLE_DICTS, []);
expect(dup.warnings).toContainEqual({ kind: 'duplicate_part', section: 'students', rowKey: 'p1:r3' });
expect(dup.review['students[0].name'].reasons).toContain('duplicate_ref');
expect(dup.review['students[1].name'].reasons).toContain('duplicate_ref');
expect(dup.warnings).toContainEqual({ kind: 'record_incomplete', section: 'students', rowKey: 'p1:r4' });
const orphan = parseReceipt(sample, rowsOf(['학생 기록', '• 모모톡 EP 2']), SAMPLE_DICTS, []);
expect(orphan.unparsed[0]).toMatchObject({ rowKey: 'p1:r1', section: 'students' });
// prefer: 학생부에 미카만 있으면 순위의 미타는 미카로
const pref = parseReceipt(sample, rowsOf(['학생 기록', '미카 (Lv. 9, 5성)', '최애 순위', '1등. 미타 (호감도: 10)']), SAMPLE_DICTS, []);
expect(pref.data.favorites[0].names[0].ref).toBe('s4');
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): parse records and rankings`

### Task 1.12: letter

**Files:** Create `src/core/parse/letter.ts` · Modify `src/core/parse/index.ts` · Test `tests/core/parse-letter.test.ts`

**Interfaces:** Consumes 1.10~1.11. Produces letter 해석.

동작: letter 섹션의 행을 공백 하나로 이어 한 문자열로 만든 뒤, `sentences` 패턴마다 첫 일치에서 값을 뽑는다. 못 찾은 값은 `null` + `missing`/`not_found`. 조사 변이(이고/고, 이에요/예요)는 읽기 양식의 패턴이 받는다(코드에 규칙 없음). `private` 값이 든 행은 `privateRowKeys`에 넣는다(문장이 여러 행에 걸치면 걸친 행 모두).

- [ ] **Step 1: 실패하는 테스트**
```ts
const r = parseReceipt(sample, rowsOf(['교무일지', '올해 스케줄을', '1,234번 해주셨어요.', '가장 많이 건네신 선물은 「고전 시집」, 무려']), SAMPLE_DICTS, []);
expect(r.data.letter).toMatchObject({ schedule: 1234, topGift: { printed: '고전 시집', ref: 'g1' } });
const none = parseReceipt(sample, rowsOf(['교무일지', '안녕하세요']), SAMPLE_DICTS, []);
expect(none.data.letter.schedule).toBeNull();
expect(none.review['letter.schedule']).toMatchObject({ status: 'missing', reasons: ['not_found'] });
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): parse letter sentences`

### Task 1.13: 사용자 수정 다시 입히기

**Files:** Create `src/core/parse/corrections.ts` · Modify `src/core/parse/index.ts` · Test `tests/core/corrections.test.ts`

**Interfaces:** Consumes `Correction`. Produces `parseReceipt`의 `corrections` 처리와 `staleCorrections`.

동작: 텍스트 수정(`{ rowKey, text }`)은 해석 전에 그 행의 텍스트를 바꾼다. 값 수정(`{ rowKey, path, set }`)은 해석 뒤, 그 `rowKey`에서 나온 값(records는 레코드의 start나 part 행, list는 그 행, fields는 라벨 행)의 `path` 값을 `set`으로 바꾸고 review 항목을 `{ status: 'ok', reasons: ['edited'], rowKey }`로 통째로 바꾼다(candidates 없앰). 대상 행을 찾지 못한 수정은 버리지 않고 `staleCorrections`로 돌려준다.

- [ ] **Step 1: 실패하는 테스트**
```ts
const rows = rowsOf(['학생 기록', '세리타 (Lv. 9, 5성)'], 'p2');
const fix = { rowKey: 'p2:r1', path: 'students.name', set: { printed: '세리나', ref: 's2' } };
expect(parseReceipt(sample, rows, SAMPLE_DICTS, [fix]).data.students[0].name).toEqual({ printed: '세리나', ref: 's2' });
// reattach: 앞에 사진이 하나 더 들어와 위치가 밀려도 같은 학생에 붙는다
const more = [...rowsOf(['학생 기록', '미카 (Lv. 9, 5성)'], 'p1'), ...rowsOf(['세리타 (Lv. 9, 5성)'], 'p2').map(r => ({ ...r, rowKey: 'p2:r1' }))];
const re = parseReceipt(sample, more, SAMPLE_DICTS, [fix]);
expect(re.data.students[1].name.ref).toBe('s2');
expect(re.review['students[1].name']).toEqual({ status: 'ok', reasons: ['edited'], rowKey: 'p2:r1' });
// stale
const gone = { rowKey: 'p9:r0', path: 'students.name', set: { printed: 'x', ref: null } };
expect(parseReceipt(sample, rows, SAMPLE_DICTS, [gone]).staleCorrections).toEqual([gone]);
// 텍스트 수정으로 unparsed 행을 다시 해석
const t = parseReceipt(sample, rowsOf(['학생 기록', '세리나 Lv 1 3']), SAMPLE_DICTS, [{ rowKey: 'p1:r1', text: '세리나 (Lv. 1, 3성)' }]);
expect(t.unparsed).toEqual([]);
expect(t.data.students[0].name.ref).toBe('s2');
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): reattach user corrections by row`

### Task 1.14: 빠진 줄 감지

**Files:** Create `src/core/checks.ts` · Test `tests/core/checks.test.ts`

**Interfaces:** Produces `findGaps(data, seams): Warning[]`. 파이프라인은 `parseReceipt`의 warnings와 `findGaps` 결과를 합친다.

- [ ] **Step 1: 실패하는 테스트** (`d()`는 `emptyRecordData()` 복사본)
```ts
expect(findGaps(d(), [{ afterPhotoId: 'p1', beforePhotoId: 'p2', overlapRows: 0 }]))
  .toContainEqual({ kind: 'seam_no_overlap', afterPhotoId: 'p1', beforePhotoId: 'p2' });
const c = d(); c.header.studentCount = 3; c.students = [stu('세리카')];
expect(findGaps(c, [])).toContainEqual({ kind: 'count_mismatch', section: 'students', expected: 3, found: 1 });
const s = d(); s.totalAssault = [ta(32), ta(33), ta(35)];
expect(findGaps(s, [])).toContainEqual({ kind: 'season_gap', section: 'totalAssault', after: 33 });
const g = d(); g.gifts = [gift(10), gift(12)];
expect(findGaps(g, [])).toContainEqual({ kind: 'not_descending', section: 'gifts', index: 1 });
expect(findGaps(d(), [])).toEqual([]);
```
(`stu`, `ta`, `gift`는 테스트 안의 작은 생성 함수. 시즌 검사는 `totalAssault`, `grandAssault`, `limitBreak`, 내림차순 검사는 `gifts`.)
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): missing line warnings`

### Task 1.15: 가리기와 문제 신고용 텍스트

**Files:** Create `src/core/export.ts` · Test `tests/core/export.test.ts`

**Interfaces:** Produces `toMaskedLines(rows, privateRowKeys)`, `maskRecord(record)`.

`maskRecord`는 원본을 바꾸지 않고 복사본에서: `header.nickname`·`header.circle`·`letter.nickname`을 `null`로, `corrections`에서 그 세 경로의 값 수정과 모든 텍스트 수정(`{ rowKey, text }`)을 빼고, `section`이 `header`나 `letter`인 `unparsed`의 `text`를 `'***'`로 바꾼다.

- [ ] **Step 1: 실패하는 테스트**
```ts
const rows = rowsOf(['닉네임 실제닉네임', '함께한 날 1,000일']);
expect(toMaskedLines(rows, ['p1:r0'])).toBe('***\n함께한 날 1,000일');
const base = validRecord();
const rec = { ...base,
  data: { ...base.data, header: { ...base.data.header, nickname: '실제닉네임', circle: '서클' } },
  corrections: [{ rowKey: 'p1:r0', path: 'header.nickname', set: '실제닉네임' }, { rowKey: 'p1:r3', text: '실제닉네임 선생님' }],
  unparsed: [{ rowKey: 'p1:r5', section: 'letter', text: '『실제닉네임』선생님과', photoId: 'p1' }] };
const masked = maskRecord(rec);
expect(masked.data.header.nickname).toBeNull();
expect(masked.data.header.circle).toBeNull();
expect(JSON.stringify(masked)).not.toContain('실제닉네임');
expect(rec.data.header.nickname).toBe('실제닉네임');   // 원본은 그대로
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: Commit** `feat(core): masking for export`

---

## 2단계 — 앱 뼈대

### Task 2.1: Capacitor 추가

**Files:** Create `capacitor.config.ts`, `android/`(생성) · Modify `android/variables.gradle`, `android/app/build.gradle`, `android/app/src/main/AndroidManifest.xml`, Create `android/app/src/main/res/xml/data_extraction_rules.xml`, Modify `CLAUDE.md`(실행 방법)

- [ ] **Step 1:** `@capacitor/core`, `@capacitor/android`, `@capacitor/cli`(dev)를 작업 시점 최신 8.x로 설치. `npx cap init "<앱 이름>" "<앱 ID>" --web-dir dist` → `npx cap add android` (앱 이름·ID는 소유자에게 묻는다).
- [ ] **Step 2:** `minSdkVersion = 29`. `defaultConfig.ndk { abiFilters 'arm64-v8a', 'armeabi-v7a' }`. 매니페스트 `android:allowBackup="false"`, `android:dataExtractionRules="@xml/data_extraction_rules"`(cloud-backup·device-transfer 모두 `<exclude domain="root" .../>` 등으로 전부 제외), `fullBackupContent` 제거.
- [ ] **Step 3: 확인** — `npm run build && npx cap sync android && cd android && ./gradlew assembleDebug` 성공. 실기기에서 앱이 뜬다(빈 화면 가능).
- [ ] **Step 4: Commit** `chore(android): add capacitor`

### Task 2.2: Java 플러그인 (스캔·사진 고르기·인식)

**Files:** Create `android/app/src/main/java/<패키지>/ReceiptPlugin.java`, `src/plugin/receipt.ts`, `src/engines/types.ts`, `src/engines/mlkit.ts` · Modify `android/app/build.gradle`, `MainActivity.java` · Test `tests/engines/mlkit.test.ts`

**Interfaces:**
```ts
// src/plugin/receipt.ts — 플러그인 이름 'Receipt'
isScannerAvailable(): Promise<{ available: boolean }>;
scan(opts: { pageLimit: number }): Promise<{ pages: string[] }>;                 // 파일 URI
pickImages(opts: { limit: number }): Promise<{ uris: string[] }>;                // content URI
recognize(opts: { uri: string }): Promise<{ width: number; height: number;
  lines: { text: string; confidence: number | null; points: Point[] }[] }>;
// saveToDownloads, openDocuments는 Task 3.4에서 추가
// src/engines/types.ts
export type OcrInput = { kind: 'canvas'; canvas: HTMLCanvasElement } | { kind: 'uri'; uri: string };
export interface OcrEngine { name: 'paddle' | 'mlkit'; accepts: OcrInput['kind'];
  init(onProgress?: (ratio: number) => void): Promise<void>;
  recognize(input: OcrInput, photoId: string): Promise<{ lines: OcrLine[]; size: { w: number; h: number } }>;
  dispose(): void }
```
Produces `MlkitEngine implements OcrEngine`(accepts `'uri'`), `fromMlkitLines(raw, photoId): OcrLine[]`.

- [ ] **Step 1: 실패하는 테스트**
```ts
expect(fromMlkitLines([{ text: 'A', confidence: null, points: [[0,0],[1,0],[1,1],[0,1]] }], 'p1'))
  .toEqual([{ text: 'A', score: 1, box: [[0,0],[1,0],[1,1],[0,1]], photoId: 'p1' }]);
expect(fromMlkitLines([{ text: 'B', confidence: 0.8, points: [] }], 'p2')[0].score).toBe(0.8);
```
- [ ] **Step 2:** FAIL → **Step 3:** 구현
  - 의존성: `com.google.mlkit:text-recognition-korean`(번들형), `com.google.android.gms:play-services-mlkit-document-scanner`(작업 시점 최신 안정판).
  - `isScannerAvailable`: `GoogleApiAvailability`.
  - `scan`: 갤러리 가져오기 허용, `pageLimit`, 결과 JPEG, 모드는 `FULL`(5.1 실측 뒤 바꿀 수 있게 상수로). 결과는 앱 캐시 파일이므로 받자마자 처리한다.
  - `pickImages`: androidx `PickMultipleVisualMedia`(포토 피커가 없으면 시스템 문서 선택기로 대체됨).
  - `recognize`: `InputImage.fromFilePath` → 한국어 인식기 → 블록 안 줄마다 텍스트, `Line.getConfidence()`, `getCornerPoints()`, 이미지 크기.
- [ ] **Step 4:** PASS. 실기기 확인: 스캔 → 인식, 사진 고르기 → 인식 모두 줄이 나온다.
- [ ] **Step 5: Commit** `feat(android): receipt plugin with scanner, picker and korean recognizer`

---

## 3단계 — 앱 처리 흐름·기록

### Task 3.1: 파이프라인·입력·세션 저장

**Files:** Create `src/pipeline.ts`, `src/store/session.ts`, `src/ui/upload.ts`, `src/engines/index.ts`, `src/platform/templateSource.ts`(이 Task에서는 `loadBundled`만) · Test `tests/pipeline.test.ts`, `tests/store/session.test.ts`

**Interfaces:**
```ts
export interface Photo { photoId: string; kind: 'file' | 'uri'; uri?: string; file?: File }
export function inputOptions(env: { native: boolean; scanner: boolean }): ('scanner' | 'file')[];
export function mapBox(box: Point[], from: { w: number; h: number }, to: { w: number; h: number }): Point[];
export function processPhotos(photos: Photo[], engine: OcrEngine,
  onEach: (photoId: string, lines: OcrLine[], size: { w: number; h: number }) => void): Promise<void>;
export function recompute(s: SessionRecord, t: Template, d: Dicts): ParseResult & { seams: Seam[]; rows: Row[] };
export interface SessionRecord { recordId: string | null; templateId: string | null; order: string[];
  photos: Record<string, { kind: 'file' | 'uri'; uri?: string; size: { w: number; h: number } }>;
  linesByPhoto: Record<string, OcrLine[]>; corrections: Correction[]; updatedAt: string }
// src/platform/templateSource.ts
export interface TemplateBundle { index: TemplateIndex; templates: Template[]; dicts: Dicts }
export function loadBundled(): Promise<TemplateBundle>;   // 앱에 들어 있는 public/templates·public/dict를 fetch로 읽음
export function toSessionRecord(state: unknown): SessionRecord;   // 캔버스·Blob 등은 버린다
export function saveSession(s: SessionRecord): Promise<void>; loadSession(): Promise<SessionRecord | null>; clearSession(): Promise<void>;
export function selectEngine(env: { native: boolean }): 'paddle' | 'mlkit';   // 앱 'mlkit', 웹 'paddle'(7단계)
```
`recompute` = `order` 순서로 `assembleRows`(+`dedupeLines`) → `mergePhotos` → `parseReceipt(t, rows, d, s.corrections, merge.conflicts)` → `findGaps`로 경고를 합친다(사진을 다시 인식하지 않는다).

- [ ] **Step 1: 실패하는 테스트**
```ts
expect(inputOptions({ native: true, scanner: true })).toEqual(['scanner', 'file']);
expect(inputOptions({ native: true, scanner: false })).toEqual(['file']);
expect(inputOptions({ native: false, scanner: false })).toEqual(['file']);
expect(mapBox([[100, 200]], { w: 4000, h: 3000 }, { w: 1000, h: 750 })).toEqual([[25, 50]]);
expect(selectEngine({ native: true })).toBe('mlkit');
const rec = toSessionRecord({ ...state, canvases: [{}], blobs: [new Blob(['x'])] });
expect(JSON.parse(JSON.stringify(rec))).toEqual(rec);
expect(Object.keys(rec).sort()).toEqual(['corrections', 'linesByPhoto', 'order', 'photos', 'recordId', 'templateId', 'updatedAt']);
expect(recompute(sessionWith(['p2', 'p1']), sample, SAMPLE_DICTS).rows[0].photoId).toBe('p2');
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: UI** — 첫 화면에 `inputOptions` 결과대로 「스캔으로 찍기」·「사진 고르기」(앱에서는 플러그인 `pickImages`, 웹에서만 `<input type="file" multiple>`). 사진마다 `photoId = crypto.randomUUID()`. 한 장씩 인식하고 끝날 때마다 세션 저장·화면 갱신. 진행률 n/N. 저장된 세션이 있으면 「이어서 하기 / 새로 시작」. 촬영 안내 문구: "영수증을 위에서부터 차례로, 위아래가 한두 줄씩 겹치게 찍어 주세요." 시작할 때 `loadBundled()`로 양식·사전을 읽고, 첫 사진 인식 뒤 `detectTemplate`으로 `templateId`를 정해 세션에 저장한다. `null`이면 양식 고르기 화면을 띄운다(개발자 모드에서는 index에 없는 `sample`도 고를 수 있게 `templates/sample.json`을 직접 읽는다).
- [ ] **Step 6: 확인** — 실기기에서 사진 3장을 넣고 중간에 앱을 껐다 켜면 이어서 하기로 인식 결과가 복원된다.
- [ ] **Step 7: Commit** `feat: photo pipeline, input options and session store`

### Task 3.2: 사진 순서 화면

**Files:** Create `src/ui/order.ts` · Modify `src/pipeline.ts` · Test `tests/order.test.ts`

**Interfaces:** Produces `movePhoto(order: string[], from: number, to: number): string[]`(순수 함수).

- [ ] **Step 1: 실패하는 테스트**
```ts
expect(movePhoto(['a', 'b', 'c'], 2, 0)).toEqual(['c', 'a', 'b']);
expect(movePhoto(['a', 'b', 'c'], 0, 0)).toEqual(['a', 'b', 'c']);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: UI** — 썸네일 목록(메모리 속 축소본, 복원 세션에서는 사진 번호만)과 위/아래 이동 버튼(끌기는 선택). 이음매 경고(`seam_no_overlap`)가 있는 사진 사이에 "이 사이에 빠진 줄이 있을 수 있어요" 표시. 순서를 바꾸면 `recompute` 후 세션 저장. 사진 추가·삭제도 이 화면에서.
- [ ] **Step 6: Commit** `feat(ui): photo order screen`

### Task 3.3: 확인 화면

**Files:** Create `src/ui/review.ts` · Test `tests/review.test.ts`

**Interfaces:** Consumes `recompute`, `mapBox`, `normalizeValue`, `Correction`. Produces `groupReview(result: ParseResult, t: Template): { section: string; items: { path: string; entry: ReviewEntry }[]; unparsed: Unparsed[] }[]`(섹션 순서는 읽기 양식 순서, `ok` 항목은 빼고, 남은 개수 계산용).

- [ ] **Step 1: 실패하는 테스트**
```ts
const g = groupReview(resultWith({ 'students[1].name': rv('review'), 'header.level': rv('missing'), 'students[0].level': rv('ok') }, [unp('students')]), sample);
expect(g.map(x => x.section)).toEqual(['header', 'students']);
expect(g[1].items.map(i => i.path)).toEqual(['students[1].name']);
expect(g[1].unparsed).toHaveLength(1);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: UI**
  - 맨 위: 경고 목록, 「적용하지 못한 수정」(`staleCorrections`).
  - 섹션별 `review`·`missing` 값: 원본 조각 미리보기(세션에 사진이 있으면 parts 박스를 `mapBox`로 환산해 잘라 표시, 없으면 "원본 없음"), 직접 수정 입력, `dict_ambiguous`·`not_in_dict`는 후보 3개 버튼.
  - 해석 못 한 행: 텍스트 수정 → 텍스트 Correction 추가 → `recompute`.
  - 직접 입력한 값은 그 값의 ValueSpec으로 `normalizeValue(spec, 입력, ctx)`를 돌려, `bad_format`·`out_of_range`면 저장하지 않고 이유를 보이고, 아니면 결과 `value`를 Correction의 `set`으로 쓴다.
  - 수정은 모두 `corrections`에 넣고 세션 저장 → `recompute`.
  - 「저장」 버튼 → Task 3.4의 기록 저장. 「문제 신고용 인식 결과 복사」 → `toMaskedLines`.
- [ ] **Step 6: Commit** `feat(ui): review screen`

### Task 3.4: 기록 저장·목록·백업·복구

**Files:** Create `src/store/records.ts`, `src/ui/records.ts` · Modify `ReceiptPlugin.java`, `src/plugin/receipt.ts` · Test `tests/store/records.test.ts`

**Interfaces:**
```ts
// 플러그인 추가
saveToDownloads(opts: { name: string; text: string }): Promise<{ uri: string }>;   // MediaStore.Downloads, RELATIVE_PATH 'Download/<앱 폴더>/'
openDocuments(opts: { mimeTypes: string[]; multiple: boolean }): Promise<{ files: { name: string; text: string }[] }>; // ACTION_OPEN_DOCUMENT
// src/store/records.ts
export function newRecord(r: ParseResult, templateId: string, corrections: Correction[], now: Date, id: string): ReceiptRecord;
export function putRecord(r: ReceiptRecord): Promise<void>; getRecord(id: string): Promise<ReceiptRecord | null>;
export function listRecords(): Promise<{ recordId: string; readingTemplate: string; printedAt: string | null; savedAt: string }[]>;
export function deleteRecord(id: string): Promise<void>;   // 인식 줄도 함께 지움
export interface RecordLines { order: string[]; linesByPhoto: Record<string, OcrLine[]>; photos: SessionRecord['photos']; corrections: Correction[] }
export function putLines(id: string, l: RecordLines): Promise<void>; getLines(id: string): Promise<RecordLines | null>;
export function backupName(r: ReceiptRecord, masked: boolean): string;   // 'receipt-<recordId>.json' | 'receipt-<recordId>-masked.json'
export function planRestore(files: { name: string; text: string }[], existingIds: string[]):
  { imports: ReceiptRecord[]; conflicts: ReceiptRecord[]; rejected: { name: string; errors: string[] }[] };
```
idb-keyval 키는 `record:<recordId>`, 인식 줄은 `lines:<recordId>`(OCR 텍스트만, 이미지 없음, 백업 파일에는 넣지 않음). 저장할 때 세션에 `recordId`가 있으면 그것을, 없으면 새 UUID를 쓰고 세션에 기록한다. 저장마다 `putRecord`·`putLines` 뒤 `saveToDownloads(backupName(r, false), JSON.stringify(r))`. 같은 기록은 같은 이름으로 덮어쓴다(현재 설치가 만든 파일이면 MediaStore에서 찾아 갱신, 아니면 새로 만든다). **기록 열기:** 인식 줄이 있으면 세션으로 불러와(`recordId` 유지) 확인 화면에서 다시 고칠 수 있다. 백업에서 복원한 기록처럼 인식 줄이 없으면 읽기 전용 보기(데이터, 경고, 가린 사본 저장, 삭제)다.

- [ ] **Step 1: 실패하는 테스트**
```ts
const r = newRecord(result, 'sample', [], new Date('2026-11-10T03:00:00Z'), 'id1');
expect(parseRecordFile(JSON.stringify(r)).ok).toBe(true);
expect(backupName(r, true)).toBe('receipt-id1-masked.json');
// rejects invalid
const p = planRestore([{ name: 'a.json', text: JSON.stringify(r) }, { name: 'b.json', text: '{}' }, { name: 'c.json', text: 'x' }], ['id1']);
expect(p.conflicts.map(x => x.recordId)).toEqual(['id1']);
expect(p.imports).toEqual([]);
expect(p.rejected.map(x => x.name)).toEqual(['b.json', 'c.json']);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: UI** — 첫 화면에 기록 목록(읽기 양식 연도, DATE, 저장 시각)과 열기·삭제·「새로 읽기」·「백업에서 복원」. 복원은 `openDocuments({ mimeTypes: ['application/json'], multiple: true })` → `planRestore` → `conflicts`는 하나씩 "덮어쓸까요?" → 결과 요약(가져옴 n, 건너뜀 n, 거절 n과 이유). 기록 화면에 「가린 사본 저장」(`maskRecord` → `saveToDownloads(backupName(r, true), …)`).
- [ ] **Step 6: 확인** — 실기기: 저장 → 파일 앱의 Download/<앱 폴더>에 파일이 있다. 앱 삭제 → 재설치 → 복원으로 그 파일을 고르면 기록이 돌아온다. 깨진 JSON을 고르면 거절되고 기존 기록은 그대로다.
- [ ] **Step 7: Commit** `feat: record list, downloads backup and restore`

### Task 3.5: Gemini 보조 (BYOK)

**Files:** Create `src/llm/types.ts`, `src/llm/keyStore.ts`, `src/llm/gemini.ts`, `src/ui/settings.ts` · Test `tests/llm/keyStore.test.ts`, `tests/llm/gemini.test.ts`

**Interfaces:** Produces `createKeyStore(storage: Storage)`, `pickModel(models): string | null`, `buildRequest(items: { path: string; spec: ValueSpec; base64: string }[]): object`, `GeminiReader implements LlmReader`(`readFields(items: { path: string; spec: ValueSpec; crop: Blob }[]): Promise<{ path: string; value: string | null }[]>`), `class RateLimitError`.

모델 선택: `generateContent`를 지원하고 이름에 `flash`가 있으며 `lite|preview|exp|image|tts|live|embedding`이 없는 것 중 버전 숫자가 가장 큰 것. 버전 숫자가 없는 별칭은 뺀다. 없으면 `flash-lite` 중 가장 큰 것. 설정의 수동 지정이 우선. 요청은 `LLM_BATCH`개씩, `responseMimeType: 'application/json'` + `responseSchema`(`[{ index: integer, value: string | null }]`) + `thinkingConfig: { thinkingBudget: 0 }`. 400이 thinking 관련이면 `thinkingConfig` 없이 한 번 재시도. 429는 `RateLimitError`. Reader는 읽은 값을 그대로 돌려주고, 사전·형식 검사는 확인 화면이 `normalizeValue`로 한다(채택 `llm_read`, 거절 `llm_rejected`).

- [ ] **Step 1: 실패하는 테스트**
```ts
const mem = createKeyStore(fakeStorage);
mem.set('k1', false); expect(mem.get()).toBe('k1'); expect(fakeStorage.getItem('gemini_api_key')).toBeNull();
mem.set('k2', true); expect(fakeStorage.getItem('gemini_api_key')).toBe('k2');
mem.clear(); expect(mem.get()).toBeNull(); expect(fakeStorage.getItem('gemini_api_key')).toBeNull();
expect(pickModel(models(['models/gemini-2.5-flash', 'models/gemini-2.5-flash-lite', 'models/gemini-9.0-flash-preview']))).toBe('models/gemini-2.5-flash');
expect(pickModel(models(['models/gemini-2.5-flash-lite']))).toBe('models/gemini-2.5-flash-lite');
expect(pickModel(models(['models/gemini-flash-latest', 'models/gemini-2.5-flash']))).toBe('models/gemini-2.5-flash');
expect(imageParts(buildRequest([item('students[0].level', { type: 'number' }), item('header.nickname', { type: 'text', private: true })]))).toHaveLength(1);
expect(fetchCalledWith.headers['x-goog-api-key']).toBe('k1'); expect(fetchCalledWith.url).not.toContain('k1');
await expect(readerWith(status429).readFields([item('students[0].level', { type: 'number' })])).rejects.toBeInstanceOf(RateLimitError);
expect(await readerWith(json([{ index: 0, value: 'zzz' }])).readFields([item('students[0].name', { type: 'name', dict: 'students' })]))
  .toEqual([{ path: 'students[0].name', value: 'zzz' }]);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: UI** — 설정: 키 입력 → 「확인」(모델 목록 조회 성공 여부) → 「이 기기에 기억하기」 → 「키 삭제」 → 모델 수동 지정. 고정 안내: "무료 키로 보낸 이미지는 Google이 서비스 개선에 사용할 수 있습니다. 애매한 칸의 조각 이미지만 보내며, 닉네임·서클 칸은 보내지 않습니다. Gemini API는 만 18세 이상만 사용할 수 있습니다." 확인 화면: 키가 있을 때만 「AI로 다시 읽기」·「애매한 칸 전부 AI로」. `privateRowKeys`에 든 행은 버튼을 숨긴다. 429면 "AI 확인 한도에 걸렸어요. 직접 확인하거나 잠시 뒤 다시 시도하세요."
- [ ] **Step 6: Commit** `feat(llm): gemini byok reader`

---

## 4단계 — 템플릿·기본 데이터·원격 갱신

### Task 4.1: fixture 회귀 테스트

**Files:** Create `tests/core/fixtures.test.ts`, `tests/fixtures/sample/dicts.json`(`SAMPLE_DICTS`와 같은 내용), `tests/fixtures/sample/basic/lines.json`, `tests/fixtures/sample/basic/expected.json`

**Interfaces:** fixture 형식: `lines.json = { order: string[]; linesByPhoto: Record<string, OcrLine[]> }`(익명화), `expected.json = { templateId: string; data: Partial<RecordData>; warnings?: Warning['kind'][] }`.

- [ ] **Step 1:** `tests/fixtures/*/*/`를 모두 돌며 `public/templates/<templateId>.json`과 사전(`index.json`의 `dicts` + 테스트용 `tests/fixtures/<templateId>/dicts.json`이 있으면 그것)을 읽고, `recompute`와 같은 순서(assemble → merge → parse(merge의 conflicts 포함) → findGaps)로 돌린 결과가 `expect(result.data).toMatchObject(expected.data)`, `warnings` 종류가 같은지 검사한다. 케이스마다 `it` 하나.
- [ ] **Step 2:** sample/basic fixture를 손으로 만든다: 사진 2장, 겹침 1행(앵커 포함), 1단계 테스트의 행들을 OcrLine으로. 닉네임은 `***`.
- [ ] **Step 3: 확인** — `npx vitest run tests/core/fixtures.test.ts` → PASS. expected.json 값 하나를 바꾸면 그 케이스만 FAIL.
- [ ] **Step 4: Commit** `test: fixture regression runner`

### Task 4.2: SchaleDB 변환 스크립트

**Files:** Create `scripts/import-schaledb.mjs`, `scripts/lib/schaledb-map.mjs` · Test `tests/scripts/import-schaledb.test.ts`

**Interfaces:** `node scripts/import-schaledb.mjs [--from <로컬 폴더>] [--out private/schaledb-dict]` — 기본은 `https://schaledb.com/data/kr/{students,items,raids,localization}.min.json`을 받는다. 출력은 기본 `private/schaledb-dict/`(`.gitignore`의 `private/`라 커밋되지 않음). Produces `mapStudents(raw): DictEntry[]`, `mapGifts(raw): DictEntry[]`, `mapBosses(raids, loc): DictEntry[]`, `mapLimitBreak(raids, loc): DictEntry[]`(순수 함수, `schaledb-map.mjs`).

규칙: 학생은 Global 출시(`IsReleased[1]`)만, `name`은 SchaleDB 한국어 이름 그대로, 같은 이름이 둘 이상이면 하나로(`id`는 첫 항목의 `student:<PathName>`), `aliases`는 `SearchTags`. 선물은 `Category: 'Favor'`, id `gift:<Id>`. 보스는 보스 × 지형 조합마다 `name = '<보스>(<지형>)'`, 지형 `Street→시가전`, `Outdoor→야전`, `Indoor→실내전`. 제약해제결전은 `'<보스> (<장갑>)'`(장갑 한국어는 localization).

- [ ] **Step 1: 실패하는 테스트** (작은 손 작성 입력, 실제 다운로드 아님)
```ts
expect(mapStudents({ 1: st('하나코(수영복)', 'hanako_swimsuit', [true, true]), 2: st('미출시', 'x', [true, false]) }))
  .toEqual([{ id: 'student:hanako_swimsuit', name: '하나코(수영복)' }]);
expect(mapStudents({ 1: st('호시노(무장)', 'a', [true, true]), 2: st('호시노(무장)', 'b', [true, true]) })).toHaveLength(1);
expect(mapBosses(raids([{ name: '비나', terrains: ['Street', 'Outdoor'] }]), loc).map(e => e.name)).toEqual(['비나(시가전)', '비나(야전)']);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: 확인** — 스크립트를 실행하면 `private/schaledb-dict/`에 파일이 생기고 `git status`에 나타나지 않는다. **소유자가 사용 허락을 확인하면** 그 파일을 `public/dict/`로 옮기고 `index.json`의 `dicts`에 등록해 커밋한다(THIRD_PARTY_NOTICES에 출처 추가). 허락 전에는 이 단계를 하지 않는다.
- [ ] **Step 6: Commit** `feat(scripts): schaledb dictionary import (local output)`

### Task 4.3: 템플릿 제작 도구와 인식 줄 내보내기

**Files:** Create `tools/template-maker.html`, `tools/template-maker.ts`, `src/core/tooling.ts` · Modify `src/ui/settings.ts`(개발자 모드), `src/ui/review.ts`(「인식 줄 내보내기」) · Test `tests/core/tooling.test.ts`

**Interfaces:** Consumes `assembleRows`, `mergePhotos`, `parseReceipt`, `findGaps`, `validateTemplate`(도구에서만). Produces `collectCandidates(result: ParseResult, t: Template): Record<string, string[]>`(템플릿에서 `type: 'name'`인 값을 따라 data를 읽어, review 사유가 `not_in_dict`인 값의 printed를 dict 이름별로 모음. `truncated`는 제외. 중복 제거, 정렬), `anonymizeLines(lines: Record<string, OcrLine[]>, privateRowKeys: string[], rows: Row[]): Record<string, OcrLine[]>`(private 행에 속한 OcrLine의 text를 `***`로).

범위(설계 6.8): 행 패턴은 사람이 읽기 양식 JSON에 직접 쓴다. 도구는 ① 앱이 내보낸 인식 줄 파일들과 읽기 양식 JSON을 불러와 해석 결과를 섹션별로 미리 보고 ② `unparsed` 행과 경고를 강조하고 ③ 사전 후보를 모아 `DictEntry[]` 모양으로 내보내고 ④ 익명화한 `lines.json`과 현재 결과로 `expected.json`(private 값은 `null`)을 내보낸다. 패턴을 만들어 주는 화면은 없다. 개발 서버에서만 연다(`npm run dev` 후 `/tools/template-maker.html`), 배포 빌드에 포함하지 않는다.

앱 쪽: 설정의 개발자 모드를 켜면 확인 화면에 「인식 줄 내보내기」가 보이고, `saveToDownloads('lines-<시각>.json', JSON.stringify({ order, linesByPhoto }))`. 이 파일에는 private 값이 그대로 있으므로 저장소에 넣지 않는다(`private/`에 둔다).

- [ ] **Step 1: 실패하는 테스트**
```ts
// sample 양식: totalAssault.boss(dict 'bosses'), students.name(dict 'students')
const res = parseReceipt(sample, rowsOf(['학생 기록', '세리카 (Lv. 9, 5성)', '총력전 트로피', '• 시즌 1 - 새 보스(야전) 골드', '• 시즌 2 - 새 보스(야전) 골드', '• 시즌 3 - KAITEN FX Mk.0(... 골드']), SAMPLE_DICTS, []);
expect(collectCandidates(res, sample)).toEqual({ bosses: ['새 보스(야전)'] });
const lines = { p1: [L('닉네임 실제닉네임', 10, 10), L('함께한 날 1,000일', 10, 40)] };
const an = anonymizeLines(lines, ['p1:r0'], assembleRows(lines.p1, 'p1'));
expect(an.p1.map(l => l.text)).toEqual(['***', '함께한 날 1,000일']);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: 확인** — 가짜 인식 줄 파일로 도구를 열어 sample 양식 결과가 보이고, 사전 후보·fixture를 내보낼 수 있다.
- [ ] **Step 6: Commit** `feat(tools): template maker and line export`

### Task 4.4: Cloudflare Pages에 템플릿 공개

**Files:** Create `public/_headers` · Modify `src/platform/templateSource.ts`(SITE_URL — 4.5에서 파일 생성 시 함께)

- [ ] **Step 1:** Cloudflare Pages 프로젝트를 GitHub 저장소에 연결한다(소유자 계정에서). 빌드 명령 `npm run check:templates && npm run build`, 출력 `dist`, `NODE_VERSION`은 로컬과 같은 LTS, 프로덕션 브랜치 `main`. 이 단계에서는 템플릿·사전·스키마 파일만 의미가 있다(웹 화면은 7단계).
- [ ] **Step 2:** `public/_headers`(이 단계에서는 CORS만):
```
/templates/*
  Access-Control-Allow-Origin: *
/dict/*
  Access-Control-Allow-Origin: *
/schema/*
  Access-Control-Allow-Origin: *
```
- [ ] **Step 3: 확인** — `curl -sI https://<프로젝트>.pages.dev/templates/index.json`에 `access-control-allow-origin: *`가 있다. 잘못된 템플릿을 넣은 커밋은 Pages 빌드가 실패한다.
- [ ] **Step 4: Commit** `chore: publish templates with cors headers`

### Task 4.5: 원격 템플릿과 업데이트 알림

**Files:** Modify `src/platform/templateSource.ts`(3.1의 `loadBundled` 옆에 추가), `src/ui/upload.ts`(`loadBundled()` 호출을 `loadTemplateBundle({ …, bundled: loadBundled })`로 교체) · Create `src/platform/updateCheck.ts` · Test `tests/platform/templateSource.test.ts`, `tests/platform/updateCheck.test.ts`

**Interfaces:** `TemplateBundle`(3.1). Produces `loadTemplateBundle(deps: { fetchRemote(): Promise<TemplateBundle>; readCache(): Promise<TemplateBundle | null>; writeCache(b: TemplateBundle): Promise<void>; bundled(): Promise<TemplateBundle> }): Promise<{ bundle: TemplateBundle; from: 'remote' | 'cache' | 'bundled' }>`, `filterSupported(ts: Template[]): { usable: Template[]; needsUpdate: boolean }`, `isNewer(latestTag: string, current: string): boolean`(A.B.CDE와 핫픽스 알파벳까지 비교).

앱은 `SITE_URL/templates/index.json`과 거기 적힌 파일들을 5초 제한으로 받아 IndexedDB에 캐시한다. 실패하면 캐시, 캐시도 없으면 APK에 포함된 파일. 업데이트 확인은 앱에서만 `https://api.github.com/repos/<owner>/<repo>/releases/latest`의 `tag_name`을 빌드 때 넣은 앱 버전과 비교한다.

- [ ] **Step 1: 실패하는 테스트**
```ts
expect((await loadTemplateBundle(deps({ remote: B1 }))).from).toBe('remote');   // writeCache도 호출됨
expect((await loadTemplateBundle(deps({ remote: 'error', cache: B0 }))).from).toBe('cache');
expect((await loadTemplateBundle(deps({ remote: 'error', cache: null }))).from).toBe('bundled');
expect(filterSupported([t1, { ...t1, id: 'x', schemaVersion: 2 }])).toEqual({ usable: [t1], needsUpdate: true });
expect(isNewer('v1.2.00', '1.1.99')).toBe(true);
expect(isNewer('v1.1.99', '1.1.99')).toBe(false);
expect(isNewer('v1.10.00', '1.9.90')).toBe(true);
expect(isNewer('v1.2.34a', '1.2.34')).toBe(true);
expect(isNewer('v1.2.35', '1.2.34a')).toBe(true);
expect(isNewer('v1.2.34', '1.2.34a')).toBe(false);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS
- [ ] **Step 5: UI** — `needsUpdate`면 "새 영수증 양식을 쓰려면 앱 업데이트가 필요해요", 새 버전이 있으면 "새 버전 vX.Y.ZZ" 배너(누르면 릴리스 페이지를 외부 브라우저로).
- [ ] **Step 6: Commit** `feat(platform): remote templates and update check`

---

## 5단계 — 앱 공개

**선행 조건:** 원본 해상도 영수증 사진 한 세트(로컬 `private/`에만, 저장소에 넣지 않음). 사진이 없으면 이 단계를 시작하지 않는다.

### Task 5.1: 원본 사진 실측

**Files:** Create `docs/spike-ocr.md` · Modify `src/core/config.ts`, `ReceiptPlugin.java`(스캐너 모드)

- [ ] **Step 1:** 실기기에서 사진 15~20장을 인식하고, 개발자 모드로 인식 줄을 내보낸다.
- [ ] **Step 2:** 측정해서 `docs/spike-ocr.md`에 기록(개인 정보 없이 수치만): 2열 행이 ML Kit에서 한 줄 / 같은 블록 두 줄 / 다른 블록으로 나뉜 비율, 같은 행 좌우 덩어리의 중심 y 차이 ÷ 행 높이 분포(→ `ROW_OVERLAP`), 사진 사이 겹친 행 수와 앵커 유무(→ `MERGE_MIN_SIM`), 혼동 문자표, 스캐너 FULL과 BASE_WITH_FILTER에서 `•`, `*`, `&`, `…`가 살아남는지, 스캐너 출력 JPEG 해상도, 잘림 글자 수 규칙, 섹션 순서·큰 헤더 위치·「확인 필요」 표기(설계 4.2).
- [ ] **Step 3:** 측정 결과로 `config.ts` 임계값과 스캐너 모드를 고치고 `npm test`가 통과하는지 본다. 재설치 뒤 Downloads 사본 처리(덮어쓰기 불가 → 새 파일)도 확인한다.
- [ ] **Step 4: Commit** `docs: ocr spike results and tuned thresholds`

### Task 5.2: 2026 읽기 양식과 실제 fixture

**Files:** Create `public/templates/2026.json`, `tests/fixtures/2026/case1/lines.json`, `expected.json` · Modify `public/templates/index.json`, `public/dict/mainStories.json`·`events.json`·`miniStories.json`·`groupStories.json`·`collabItems.json`(영수증에서 모은 이름)

- [ ] **Step 1:** 템플릿 제작 도구로 2026 영수증의 모든 섹션(설계 4.2: header, students, favorites, totalAssault, grandAssault, counters, tacticalExam, limitBreak, missions, mainStory, events, miniStories, groupStories, collabItems, office, currency, gifts, consumption, letter)을 기록 계약에 맞춰 쓴다. `unparsed`가 0이 될 때까지 패턴을 고친다.
- [ ] **Step 2:** 도구로 익명화 fixture를 내보내 `tests/fixtures/2026/case1/`에 넣는다(닉네임·서클은 `***`/`null`).
- [ ] **Step 3: 확인** — `npm run check:templates && npm test` 통과. fixture의 `warnings`가 실제 사진 상태와 맞다.
- [ ] **Step 4: Commit** `feat(templates): 2026 reading template and fixture`

### Task 5.3: 서명과 GitHub Releases

**Files:** Modify `android/app/build.gradle`(서명 설정을 환경변수에서, `versionName`·`versionCode`를 태그에서) · Create `.github/workflows/release-android.yml`

- [ ] **Step 1:** 키스토어를 한 번만 만든다: `keytool -genkeypair -v -keystore receipt-ocr.jks -keyalg RSA -keysize 4096 -validity 36500 -alias receipt`. 저장소 밖에 두고 두 곳 이상에 백업, 비밀번호는 비밀번호 관리자에. **잃어버리면 기존 사용자가 업데이트를 설치할 수 없다.**
- [ ] **Step 2:** GitHub Secrets: `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`.
- [ ] **Step 3:** 버전 A.B.CDE. `versionName` = 태그에서 `v`를 뗀 값, `versionCode = A × 100000 + B × 1000 + C × 100 + D × 10 + E`(E는 알파벳이 없으면 0, `a` = 1, `b` = 2 …). 태그와 버전은 사용자 확인을 받은 뒤에만 만든다.
- [ ] **Step 4:** 워크플로(태그 `v*` push 시): checkout → Node LTS → JDK(그때 AGP가 요구하는 버전) → `npm ci` → `npm test` → `npm run build` → `npx cap sync android` → 키스토어 복원 → `./gradlew assembleRelease` → `receipt-ocr-<tag>.apk`로 이름 변경 → `gh release create`로 업로드(`permissions: contents: write`). 외부 액션은 공식 액션만, 버전 태그 고정.
- [ ] **Step 5: 확인** — 사용자 확인을 받은 첫 태그(예: `v0.1.00`)에서 릴리스에 APK가 붙는다. 실기기에 설치 후 다음 버전(예: `v0.1.01`)을 덮어 설치해도 데이터가 유지된다.
- [ ] **Step 6: Commit** `ci(android): signed release workflow`

### Task 5.4: CI 마무리와 신고 창구

**Files:** Modify `.github/workflows/ci.yml` · Create `.github/ISSUE_TEMPLATE/bug.md`

- [ ] **Step 1:** `ci.yml` 순서: `npm ci` → `npm run gen:validators && git diff --exit-code src/core/generated` → `npm test` → `npm run check:templates` → `npm run build` → `npm run check:assets`(7.3 전에는 이 줄 생략).
- [ ] **Step 2:** 이슈 템플릿 칸: 영수증 연도, 기기와 앱 버전, 증상, 「문제 신고용 인식 결과」 붙여넣기. 맨 위에 "영수증 사진과 닉네임은 올리지 마세요".
- [ ] **Step 3: 확인** — 일부러 템플릿을 깨뜨린 PR에서 CI가 실패한다.
- [ ] **Step 4: Commit** `ci: full checks and issue template`

### Task 5.5: 라이선스·고지·크레딧

**Files:** Create `LICENSE`, `THIRD_PARTY_NOTICES.md` · Modify `src/ui/`(푸터·정보 화면)

- [ ] **Step 1:** 프로젝트 라이선스는 소유자에게 묻는다(제안: MIT).
- [ ] **Step 2:** THIRD_PARTY_NOTICES: 배포물에 포함되는 패키지의 라이선스 전문(Capacitor, es-hangul, idb-keyval, Ajv 생성 코드 등), ML Kit은 Google 약관 링크, 기본 데이터를 커밋했다면 SchaleDB 출처.
- [ ] **Step 3:** 앱 정보 화면·README: "비공식 팬메이드 도구", "블루 아카이브 · 넥슨" 출처 표기, 제작 크레딧(Jigap-Appem), GitHub Issues 링크, "사진은 기기 밖으로 나가지 않습니다".
- [ ] **Step 4: Commit** `docs: license, notices and credits`

### Task 5.6: README

**Files:** Create `README.md`

- [ ] **Step 1:** 이 앱이 하는 일, 설치(GitHub Releases APK, 출처를 알 수 없는 앱 허용), 촬영 방법(위아래 한두 줄 겹치게), 개인정보 원칙(사진은 기기 밖으로 안 나감, AI는 선택, 백업 파일에는 닉네임이 들어 있음), 백업·복원 방법, 개발 명령(`npm run dev / test / check:templates / gen:validators`), 템플릿 제작 도구 사용 순서, `docs/ANNUAL.md` 링크.
- [ ] **Step 2: Commit** `docs: readme`

### Task 5.7: 연례 점검 문서

**Files:** Create `docs/ANNUAL.md`

- [ ] **Step 1:** 아래 체크리스트를 그대로 넣는다.
  1. 새 영수증을 앱으로 인식해 인식 줄을 내보내고, 템플릿 제작 도구로 `public/templates/<id>.json`과 `tests/fixtures/<id>/case1/`을 만든다.
  2. 기록 계약에 없는 내용이 있으면 `src/core/record.ts`·`record.schema.json`을 고치고 `schemaVersion`을 올린다(`npm run gen:validators`).
  3. `node scripts/import-schaledb.mjs`로 기본 데이터를 새로 받고(사용 허락 상태 재확인), 스토리 제목은 도구가 모은 후보로 `public/dict/`에 추가한다.
  4. `public/templates/index.json`에 등록한다.
  5. `npm run check:templates && npm test`가 지난 연도 fixture까지 전부 통과하는지 본다.
  6. `main`에 push → Cloudflare 자동 배포 → 앱에서 앱 업데이트 없이 새 양식이 보이는지 확인한다.
  7. Capacitor, ML Kit, Gradle 의존성과 targetSdk를 올리고, 사용자 확인 후 태그를 붙여 새 APK를 릴리스한다.
  8. 본인 Gemini 키로 AI 재확인이 되는지, 모델 자동 선택이 현재 모델을 고르는지 확인한다.
  9. 안드로이드 개발자 인증(2027년 전 세계 시행 예정) 상태를 확인하고 필요하면 패키지와 서명 키를 등록한다.
  10. 키스토어 백업 두 곳이 살아 있는지 확인한다.
  11. `npm audit`을 보고, 의존성 업데이트는 테스트가 통과할 때만 반영한다.
- [ ] **Step 2: Commit** `docs: annual checklist`

---

## 6단계 — 디지털 영수증 (자리만)

설계 8장. 이 계획에는 상세 Task를 쓰지 않는다. 시작 조건: 5단계 완료, 소유자가 글꼴을 정함, 공식 이미지 조각(사용자 정보 없는 부분)을 준비함. 시작할 때 `docs/superpowers/plans/<날짜>-digital-receipt.md`로 별도 계획을 쓴다. 그때 정할 것: 출력 양식을 데이터 파일로 할지 연도별 코드로 할지. 정해 둔 방향: HTML/CSS 배치 → 줄 경계에서 자른 조각을 차례로 캔버스에 그려 PNG/JPEG 여러 장(조각 높이 기본 8,192px, 폭은 사용자 선택) → 같은 조각으로 PDF, 이름은 `printed`, 교무일지는 문장 양식 + 조사·줄바꿈 규칙, 닉네임·서클 가리기, 저장은 MediaStore(Pictures, Download). 첫 Task에서 `.githooks/pre-commit`의 이미지 허용 경로에 `public/receipt-art/`를 추가한다.

---

## 7단계 — 웹

### Task 7.1: PaddleOCR 엔진

**Files:** Create `src/engines/paddle.ts`, `public/models/v1/`, `public/ort/v1/` · Modify `docs/spike-ocr.md`, `src/engines/index.ts` · Test `tests/engines/paddle.test.ts`

**Interfaces:** Produces `PaddleEngine implements OcrEngine`(accepts `'canvas'`), `toOcrLines(items: { text: string; score: number; poly: Point[] }[], photoId: string): OcrLine[]`.

- [ ] **Step 1: 실패하는 테스트**
```ts
expect(toOcrLines([{ text: 'A', score: 0.9, poly: [[0,0],[10,0],[10,5],[0,5]] }], 'p2'))
  .toEqual([{ text: 'A', score: 0.9, box: [[0,0],[10,0],[10,5],[0,5]], photoId: 'p2' }]);
```
- [ ] **Step 2:** FAIL → **Step 3:** 구현 — `@paddleocr/paddleocr-js`, 한국어 인식 `korean_PP-OCRv5_mobile_rec`, 탐지 `PP-OCRv5_mobile_det`. `lang` 옵션으로 안 되면 ONNX 모델을 SDK가 요구하는 tar 형식(`inference.onnx` + `inference.yml`)으로 묶어 `public/models/v1/`에서 지정. ONNX Runtime WASM과 OpenCV.js는 `public/ort/v1/`, `wasmPaths`를 그곳으로. `worker: true`, `backend: 'auto'`, `numThreads: Math.min(4, navigator.hardwareConcurrency || 2)`.
- [ ] **Step 4:** PASS. `docs/spike-ocr.md`에 모델 파일 크기(25MiB 미만), 첫 로딩 시간, 사진 한 장 처리 시간(PC, 샤오신 2024), 원근 보정 필요 여부. 앱 WebView에서도 엔진을 PaddleOCR로 강제해 단일 스레드로라도 끝까지 도는지 확인. Network 탭에서 다른 도메인 요청 0건.
- [ ] **Step 5: Commit** `feat(engines): paddle ocr engine`

### Task 7.2: 이미지 준비

**Files:** Create `src/image/prepare.ts` · Test `tests/image/prepare.test.ts`

**Interfaces:** Produces `computeTargetSize(w, h, maxPixels): { w; h }`(전체 화소가 maxPixels 이하가 되게 비율 유지, 내림), `planTiles(height, tileH, overlap): { y; h }[]`, `loadAndShrink(file: File): Promise<HTMLCanvasElement>`, `toTiles(canvas): { canvas; offsetY }[]`.

- [ ] **Step 1: 실패하는 테스트**
```ts
expect(computeTargetSize(8000, 6000, 3_000_000)).toEqual({ w: 2000, h: 1500 });
expect(computeTargetSize(1000, 800, 3_000_000)).toEqual({ w: 1000, h: 800 });
expect(computeTargetSize(1080, 8000, 3_000_000)).toEqual({ w: 636, h: 4714 });
expect(planTiles(4000, 1600, 120)).toEqual([{ y: 0, h: 1600 }, { y: 1480, h: 1600 }, { y: 2960, h: 1040 }]);
expect(planTiles(1000, 1600, 120)).toEqual([{ y: 0, h: 1000 }]);
```
- [ ] **Step 2~4:** FAIL → 구현 → PASS. 원본 크기 캔버스는 만들지 않는다(`createImageBitmap` 후 바로 축소 캔버스에 그림). 타일 결과는 `offsetY`를 더해 원래 좌표로 되돌린 뒤 `dedupeLines`. 다 쓴 캔버스는 `width = 0`, ImageBitmap은 `close()`.
- [ ] **Step 5: Commit** `feat(image): shrink by pixel count and tile`

### Task 7.3: 헤더와 용량 검사

**Files:** Modify `public/_headers`, `vite.config.ts` · Create `scripts/check-assets.mjs`

- [ ] **Step 1:** `_headers`에 추가(4.4의 CORS 줄은 유지):
```
/*
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
  Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; worker-src 'self' blob:; connect-src 'self' https://generativelanguage.googleapis.com; img-src 'self' blob: data:; style-src 'self' 'unsafe-inline'
  Referrer-Policy: no-referrer
/models/*
  Cache-Control: public, max-age=31536000, immutable
/ort/*
  Cache-Control: public, max-age=31536000, immutable
```
- [ ] **Step 2:** `vite.config.ts`의 `server.headers`·`preview.headers`에 같은 COOP·COEP·CSP. 모델·WASM은 덮어쓰지 말고 `v2/`처럼 새 경로로 올린다.
- [ ] **Step 3:** `check-assets.mjs`: `dist/`에 25MiB 이상 파일이 있으면 경로를 출력하고 종료 코드 1. `npm run check:assets`, `ci.yml`의 build 뒤.
- [ ] **Step 4: 확인** — `npm run build && npm run check:assets` → 0. `npx vite preview`에서 `crossOriginIsolated === true`.
- [ ] **Step 5: Commit** `chore: security headers and asset size check`

### Task 7.4: 웹 화면과 공개 확인

**Files:** Modify `src/main.ts`, `src/ui/*`(웹 분기), `src/store/records.ts`(웹 백업)

- [ ] **Step 1:** 웹의 백업 방식을 정한다(이 단계의 판단: 파일 내려받기/올리기가 기본 후보). 정한 방식으로 `saveToDownloads`·`openDocuments` 자리를 웹 구현으로 채운다.
- [ ] **Step 2: 확인(실제 주소)** — 콘솔 `crossOriginIsolated === true`. 응답 헤더에 CSP. Network 탭에 다른 도메인 요청이 없다(AI 재확인 때의 `generativelanguage.googleapis.com`만 예외). 키 없이 사진 입력부터 기록 저장까지 된다. 안드로이드 크롬에서도 같은 흐름.
- [ ] **Step 3: Commit** `feat(web): web flow and backup`

---

## 시작 전에 채울 것

- 원본 해상도 영수증 사진 한 세트(구간별, 위아래 겹치게). 5단계 시작 조건. 저장소에는 올리지 않는다.
- SchaleDB 데이터 사용 허락 확인(소유자). 확인 전에는 변환 결과를 커밋하지 않는다.
- 앱 이름, 앱 ID(예: `io.github.<아이디>.<이름>`), GitHub 저장소 이름(owner/repo), pages.dev 주소. GitHub 원격 저장소는 아직 만들지 않았다.
- 프로젝트 라이선스(제안: MIT).
- 이용자 중 미성년자 비율. Gemini 약관은 18세 미만 대상 서비스에서의 사용을 막고 있어, AI 기능을 어떻게 노출할지에 영향을 준다.
- (6단계 전) 글꼴, 공식 이미지 조각.
- 참고: 넥슨코리아 「게임IP사용가이드」(https://m.nexon.com/terms/743)는 비영리 앱 제작 시 고객센터 개별 문의를 권한다(설계 11장).
