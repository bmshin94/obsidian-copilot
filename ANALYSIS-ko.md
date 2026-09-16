# Obsidian Copilot 분석 정리

이 저장소가 무엇인지, 어떻게 쓰는지, 어떤 기회가 있는지 정리한 문서입니다.

## 저장소 정보

| 항목 | 내용 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/obsidian-copilot |
| **원본 (upstream)** | https://github.com/logancyang/obsidian-copilot |
| **플러그인 설치 페이지** | https://obsidian.md/plugins?id=copilot |
| **공식 사이트 / 요금제** | https://www.obsidiancopilot.com/en/pricing |
| **Miyo (의미 검색 앱)** | https://www.miyo.md/ |
| 버전 | 4.0.8 |
| 라이선스 | AGPL-3.0 |
| 원작자 | Logan Yang |

---

## 1. 이게 뭔가

메모 앱 **Obsidian** 안에서 AI 에이전트를 돌리는 **플러그인**입니다.

옵시디언은 메모를 내 컴퓨터에 파일로 저장하는 노트 앱인데, 메모가 쌓이면 찾기 어려워집니다.
이 플러그인을 설치하면 옵시디언 옆에 채팅창이 생기고, **거기 있는 AI가 내 메모 전체를 읽을 수 있게** 됩니다.

- "지난달 회의 내용 요약해줘"
- "이 주제 관련 메모 다 찾아줘"
- "이 자료들 정리해서 새 노트로 만들어줘"

챗GPT는 내 메모를 몰라서 매번 복붙해야 하지만, 이건 메모장 안에 들어와 있어서 복붙이 필요 없습니다.

### 실적

- 2024년 옵시디언 공식 **Best LLM Integration** 수상
- 옵시디언 AI 플러그인 **다운로드 1위**

---

## 2. 폴더 구조

```
src/            TypeScript 소스 1,419개 (React + Tailwind)
 ├ agentMode/     에이전트 백엔드 (ACP 프로토콜, Claude Agent SDK)
 │   ├ acp/         외부 CLI 에이전트 프로세스 실행·통신
 │   ├ backends/    claude / codex / opencode
 │   ├ skills/      스킬 발견·동기화·마이그레이션
 │   └ ui/          도구 호출 시각화, 권한 승인 UI
 ├ miyo/          로컬 의미검색 서버 클라이언트
 ├ tools/         AI가 쓰는 도구 (노트 읽기, 검색, 태그, 캔버스, 유튜브)
 ├ LLMProviders/  모델 공급자별 어댑터
 └ projects, settings, components, search ...
docs/           사용자 매뉴얼
designdocs/     개발자용 설계 문서 7종
dev/gallery/    컴포넌트 갤러리 (UI 상태 시각 검증)
AGENTS.md       AI 코딩 에이전트용 규칙서 (13KB)
.claude/        Claude Code 설정 + 전용 에이전트
```

---

## 3. 설치 및 사용법

### 일반 사용 (소스 불필요)

1. 옵시디언 → **설정 → 커뮤니티 플러그인 → 탐색** → "Copilot" 검색 → 설치 → 활성화
2. **설정 → Copilot → Basic → Agents**
3. 셋 중 선택
   - **opencode 다운로드** (권장, Copilot이 알아서 관리)
   - **Claude Code 자동 감지** (기존 로그인 재사용)
   - **Codex 연결** (`@agentclientprotocol/codex-acp` 어댑터)
4. 리본의 Agent 아이콘 클릭 또는 명령 팔레트 → `Open Copilot Agent Chat Window`

**요구사항**: 옵시디언 1.11.4 이상 (`manifest.json`의 `minAppVersion`)

**모바일 주의**: Agent 모드는 로컬 프로세스를 띄워야 해서 **데스크톱 전용**입니다.
모바일에서는 Quick Chat, Commands, Quick Ask만 사용 가능합니다.

### 소스 빌드

```bash
npm install
npm run build     # TypeScript 검사 + main.js 생성
```

생성된 `main.js`, `manifest.json`, `styles.css`를 볼트의 `.obsidian/plugins/copilot/`에 복사.

> `AGENTS.md`에 **"`npm run dev` 절대 실행 금지"**로 명시돼 있습니다. 빌드는 `npm run build`만 사용하세요.

---

## 4. 플러그인인가, 스킬인가, MCP인가

**옵시디언 플러그인입니다.** 그 안에 스킬과 MCP가 함께 들어 있습니다.

```
[옵시디언]
  └─ Copilot 플러그인            ← 이 저장소. 본체
       └─ ACP 프로토콜로 외부 에이전트 실행
            └─ opencode / Claude Code / Codex   ← 실제 AI 두뇌
                 ├─ Skills       ← Copilot이 관리해서 주입
                 └─ MCP 서버들   ← 에이전트 쪽에 설정된 것
```

| 구분 | 위치 |
|---|---|
| **플러그인** | 본체. `manifest.json` + `src/main.ts` → esbuild → `main.js` |
| **스킬** | 기능으로 포함. `src/agentMode/skills/`의 `SkillManager`가 관리 |
| **MCP** | 간접 지원. Copilot 자체는 MCP 서버가 아님 |

MCP 부분 근거 — `src/agentMode/session/toolName.ts`:

```typescript
/^mcp__(.+?)__(.+)$/   // mcp__<서버명>__<도구명> 파싱
```

밑에 깔린 Claude Code 등에 MCP 서버를 설정해두면, 그 도구 실행 시
**Copilot UI가 어느 서버의 무슨 도구인지 표시**해주는 수준입니다.
MCP 연결 자체는 에이전트 담당, Copilot은 통역·표시 담당.

---

## 5. API 토큰이 필요한가

**아니요. 4가지 길 중 2개는 추가 토큰이 필요 없습니다.**

| 방법 | 토큰 | 비용 |
|---|---|---|
| ① 기존 Claude Code / Codex 로그인 재사용 | 불필요 | 이미 내는 구독료로 끝 |
| ② 로컬 모델 (Ollama) | 불필요 | **0원**, 완전 오프라인 |
| ③ BYOK (내 API 키) | 필요 | 종량제 |
| ④ Copilot 유료 플랜 | 라이선스 키 | 월 구독 |

BYOK 지원 공급자 (`package.json` 기준): OpenAI, Anthropic, Google Gemini,
Groq, DeepSeek, xAI, Ollama, 그 외 OpenAI 호환 엔드포인트.

### 보안 포인트

> Keys are stored in this device's Obsidian Keychain, **not in the vault's `data.json`**.

API 키를 볼트 파일이 아닌 **기기 키체인**에 저장합니다.
볼트를 iCloud나 Git으로 동기화해도 **키가 새지 않습니다.**

### 주의

opencode의 **무료 Zen 모델**은 제공자가 프롬프트를 로깅하거나 학습에 쓸 수 있다고
README에 명시돼 있습니다. 민감한 노트에는 사용하지 마세요.

---

## 6. 왜 유명한가

1. **시장 적중** — 옵시디언 사용자층(개발자·연구자·작가)은 "내 데이터는 내 컴퓨터에" 성향.
   여기에 AI를 붙이되 로컬 우선 원칙을 지킨 것이 정확히 맞아떨어짐
2. **공식 인증** — 2024 Best LLM Integration 수상, 다운로드 1위
3. **타이밍** — V4에서 코딩 에이전트를 노트 앱에 꽂은 초기 사례
4. **오픈 코어** — AGPL로 전부 공개 + 호스팅 모델은 유료
5. **코드 품질** — 소스 1,419개 파일에 테스트가 거의 1:1,
   `eslint-plugin-boundaries`로 레이어 간 import 강제,
   설계 문서 7종, `AGENTS.md` 13KB. 다른 개발자들이 참고하러 옴

---

## 7. 로컬 에이전트 구축에 도움이 되는가

**매우 유용합니다.** 어려운 건 모델 호출이 아니라 프로세스 관리·권한·UI인데, 그게 다 풀려 있습니다.

| 문제 | 참고할 코드 |
|---|---|
| 외부 CLI 에이전트를 자식 프로세스로 실행·통신 | `src/agentMode/acp/AcpBackendProcess.ts`, `AcpProcessManager.ts` |
| 에이전트 ↔ 앱 메시지 변환 | `acp/wireTranslate.ts` |
| "파일 수정해도 될까요?" 승인 UI | `agentMode/ui/permissionPrompter.ts` |
| 도구 호출 내역 시각화 | `ui/agentTrail.ts`, `activityGroups.ts`, `toolSummaries.ts` |
| 스킬 발견·중복처리·마이그레이션 | `agentMode/skills/` 전체 |
| 로컬 검색 서버 연동 (헬스체크, 디스커버리) | `src/miyo/` |
| 에이전트 대화 재생·디버깅 | `acp/replayTranscript.ts`, `debugTap.ts` |

### 설계 문서

```
designdocs/AGENT_HOME_ARCHITECTURE.md                에이전트 작업공간 구조
designdocs/AGENT_INSTRUCTIONS_AND_PROMPT_CACHING.md  프롬프트 캐싱 (비용 절감)
designdocs/MULTI_AGENT_FANOUT_ARCHITECTURE.md        멀티 에이전트 병렬 실행
designdocs/AGENT_TRAIL_GROUPING.md                   도구 호출 로그 그룹핑
```

---

## 8. React나 PHP로 만들 수 있는가

### React → 이미 React입니다

```json
"react": "^18.2.0",  "react-dom": "^18.2.0",
"@radix-ui/react-*": ...      // UI 컴포넌트
"tailwindcss": "^3.4.15",     // 스타일
"jotai": "^2.10.3",           // 상태관리
"lexical": "^0.34.0"          // 텍스트 에디터
```

`src/components/`, `src/agentMode/ui/`가 전부 `.tsx`.
React를 아시면 UI 쪽은 바로 기여 가능합니다. 단 **TypeScript**이고,
스타일은 Tailwind 사용이 `AGENTS.md`에 강제돼 있습니다.

### PHP → 불가능

- 옵시디언은 **Electron 앱**(Chromium + Node.js)
- 플러그인은 Electron 안에서 도는 **JavaScript**여야 로드됨
- esbuild가 `src/main.ts` → `main.js`(CommonJS)로 번들해서 옵시디언에 전달
- PHP는 서버 언어라 들어갈 자리가 없음

### 단, PHP를 백엔드로 붙이는 건 가능

```
[옵시디언 Copilot]  ──HTTP──▶  [내 PHP 서버]  ──▶  [내 DB]
```

- PHP로 API 서버를 만들고 플러그인이 HTTP 호출
- **PHP로 MCP 서버 구현** → 에이전트에 연결 (MCP는 프로토콜이라 언어를 안 가림)

---

## 9. 수익화 아이디어

### 제약: AGPL-3.0

| 행위 | 가능? |
|---|---|
| 포크해서 혼자 쓰기 | 자유 |
| 수정해서 배포 | 수정 소스 **전부 공개** 의무 |
| 수정해서 웹 서비스로 제공 | 이것도 소스 공개 의무 |
| 소스 비공개로 유료 판매 | **위반** |

**단, AGPL은 "코드를 수정할 때"만 적용됩니다.**
코드를 안 건드리고 위에 얹는 콘텐츠·데이터·별도 프로그램은 완전히 자유입니다.

### 핵심 발견: 스킬은 표준 규격

`docs/agent-mode-and-tools.md` 157행:

> Skills live under `<Copilot folder>/skills/`. Copilot links them into
> `.opencode/skills/`, `.claude/skills/`, and `.agents/skills/`

스킬은 `SKILL.md` 파일 하나입니다 (`src/agentMode/skills/skillFormat.ts`):

```markdown
---
name: tax-invoice-review
description: 세금계산서를 검토하고 오류를 찾아냅니다
license: Commercial
allowedTools: Read Grep Bash
metadata:
  copilot-enabled-agents: claude opencode codex
---

# 세금계산서 검토 스킬
...
```

**한 번 만들면 Obsidian Copilot / Claude Code / Codex / opencode / Claude Desktop에서 모두 동작합니다.**
시장이 옵시디언 하나에 갇히지 않습니다.

또한 README 161행 — **커스텀 스킬은 무료 플랜에서도 동작**하므로
고객 풀이 "Copilot 유료 결제자"로 좁혀지지 않습니다.

### 아이디어별 정리

| 아이디어 | 기술 난이도 | 초기 투입 | 수익 천장 | AGPL 리스크 | 종합 |
|---|---|---|---|---|---|
| **MCP 서버 제작** | 중 | 2~4주 | **매우 높음** | 없음 | ★★★ |
| **도메인 스킬 팩** | 하 | 1~2주 | 중 | 없음 | ★★★ |
| 볼트 템플릿 + 커맨드 | 하 | 1주 | 낮음 | 없음 | ★★ |
| 교육 콘텐츠 | 하 | 지속 | 중 | 없음 | ★★ |
| 세팅 대행 / 컨설팅 | 중 | 즉시 | 낮음(시간제) | 없음 | ★★ |
| 니치 포크 | 상 | 수개월 | 낮음 | **높음** | ★ |

#### ① MCP 서버 제작 (기대값 최상)

독립 프로그램이라 AGPL 무관, 라이선스 자유.
Claude Desktop·Cursor·Claude Code 등 MCP를 쓰는 모든 도구가 시장.

한국 시장 특화 (경쟁 거의 없음):

```
공공데이터포털 MCP     국가 통계·데이터 조회
DART 전자공시 MCP      기업 재무제표 분석
법제처 국가법령 MCP    법령·판례 검색
한국은행 ECOS MCP      경제통계
더존/영림원 MCP        국내 ERP 연동 (B2B)
네이버/카카오 MCP      지도·검색·쇼핑 API
```

수익 모델: 오픈소스 + 호스팅 유료 / B2B 커스텀 개발 / API 중계.

#### ② 도메인 특화 스킬 팩

AI가 못 하는 건 지능이 아니라 "우리 업계는 이렇게 한다"는 절차 지식.

| 타겟 | 구성 | 가격대 |
|---|---|---|
| 세무·회계 | 부가세 신고, 경비 처리 판단, 세금계산서 검토 | 5~15만원 |
| 노무·인사 | 근로계약서 검토, 취업규칙 진단, 4대보험 | 5~15만원 |
| 연구자 | 논문 구조, 선행연구 정리, 리비전 레터 | 3~8만원 |
| 개발자 | 코드리뷰 체크리스트, 장애보고서, RFC | 3~8만원 |
| 스타트업 대표 | IR덱, 투자자 업데이트, 채용공고 | 10~30만원 |

리스크: 파일이라 복제가 쉬움 → **업데이트 구독형**으로 방어.
주의: 본인 전문 분야가 아니면 금방 티가 남. 세무·법률은 면책 문구 필수.

#### ③ 볼트 템플릿 + 커맨드 세트

커맨드 변수(`docs/custom-commands.md`)를 조합해 상품화:

| 변수 | 의미 |
|---|---|
| `{}` | 선택 텍스트 (없으면 현재 노트) |
| `{activeNote}` | 현재 노트 |
| `{[[노트 제목]]}` | 특정 노트 |
| `{폴더/경로}` | 폴더 내 노트 전체 |
| `{#태그1, #태그2}` | 태그 달린 노트들 |

#### ④ 교육 콘텐츠

국내에 제대로 된 자료가 거의 없음. 단, 이것만으로 생활비는 어려움.
**다른 상품의 마케팅 깔때기**로 보는 것이 현실적.

#### ⑤ 세팅 대행 / 컨설팅

윈도우 설치 스크립트(`docs/install-claude-agent-mode-windows.ps1`)가 따로 있을 만큼
설정이 까다로움. 특히 심볼릭 링크 때문에 윈도우는 개발자 모드가 필요.
→ 비개발자 대행 수요 존재. 단 시간을 파는 구조라 확장 한계.

#### ⑥ 니치 포크 (비추천)

소스를 공개하면 합법이지만 — 본가 개발 속도를 못 따라가고,
상표 문제가 있고, 커뮤니티 스토어 중복 등록이 어렵고, 유지보수가 불가능.

### 첫 30일 실행 플랜 (스킬 팩 기준)

- **1주차 (검증)** — 본인 업무의 반복 절차 3개를 `SKILL.md`로 작성, 2주간 직접 사용하며 시간 절감 측정
- **2주차 (공개)** — 잘 되는 것 2~3개를 GitHub에 무료 공개, 커뮤니티 공유. **반응 없으면 수요 없음 → 방향 전환**
- **3주차 (확장)** — 피드백 반영해 10개 규모 팩 구성 + 문서 + 예시 볼트
- **4주차 (판매)** — Gumroad / 크몽 등록. 무료 스킬은 계속 열어두고 유료 팩으로 유도

---

## 10. 작업 기록: CLAUDE.md 복구

### 문제

```
120000 f58fe1c...  CLAUDE.md   ← 파일 모드 120000 = 심볼릭 링크
```

원래 `CLAUDE.md`는 **`AGENTS.md`를 가리키는 심볼릭 링크**였습니다.
커밋 `74007ea`("Add files via upload")에서 GitHub 웹으로 페르소나 파일을 업로드할 때
**심볼릭 링크라는 파일 속성은 그대로 두고 내용만 교체**되어,
존재하지 않는 경로를 가리키는 깨진 링크가 되었습니다.

그 결과 Claude Code가 이 파일을 읽지 못해 **페르소나 설정이 전혀 적용되지 않았습니다.**

### 조치

일반 파일(`100644`)로 다시 작성하고, 구조를 이렇게 잡았습니다.

- 1~4절: 업로드하신 페르소나 원문 그대로 보존
- 5절 추가: `@AGENTS.md` 참조

5절을 넣은 이유는, 원래 심볼릭 링크가 `AGENTS.md`를 가리키고 있었기 때문입니다.
페르소나로만 덮으면 **프로젝트 코딩 규칙 13KB가 통째로 사라집니다.**
현재 구성은 **말투는 페르소나, 코딩 규칙은 `AGENTS.md`** 입니다.

### 검증 방법

```bash
git ls-files -s CLAUDE.md
# 100644 → 정상 (일반 파일)
# 120000 → 아직 심볼릭 링크
```

> 참고: `CLAUDE.md`는 세션 시작 시점에 한 번 읽히므로,
> 변경 후에는 **새 세션을 열어야** 적용됩니다.
