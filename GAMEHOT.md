# GAMEHOT

게임 업계의 퍼스트파티 공지, 뉴스, 커뮤니티 동향을 모아 한국어로 선별·요약하는 대시보드다. [KKKKhazix/AIHOT](https://github.com/KKKKhazix/AIHOT)의 포크이며 MIT 라이선스를 따른다. AIHOT의 이름과 Logo는 쓰지 않는다.

에이전트는 이 문서를 읽은 뒤 upstream 규칙인 `AGENTS.md`와 `docs/customize.md`를 읽는다.

## 목표

- 퍼스트파티 공지: 퍼블리셔·개발사의 공식 공지, 패치노트, 점검, 이벤트, 신작 발표
- 게임 뉴스: 국내외 게임 매체 보도
- 커뮤니티 동향: 커뮤니티에서 화제가 된 이슈를 열기(heat) 신호로 반영
- 한국어: UI, 제목, 요약, 리포트를 모두 한국어로 제공

## 요구와 프레임워크 장치

| GAMEHOT 요구 | AIHOT 장치 | 위치 |
|---|---|---|
| 퍼스트파티 공지 | 신호원 `tier: T1`(공식 1차), `T1_5`(공식 계정), `owner_entity_id` | `industry/sources.json`, `industry/taxonomy.ts`의 `ENTITIES` |
| 게임 뉴스 | 신호원 `tier: T2` | `industry/sources.json` |
| 커뮤니티 동향 | `participation_mode: hot_signal`, 이벤트 열기 계산 | `industry/sources.json`, `site/site.ts`의 `COMMUNITY_FEEDS` |
| 신작·대형 업데이트 집계 | `RELEASE` | `industry/taxonomy.ts` |
| 선별 기준 | 평가 prompt, 신호원 등급별 문턱 | `industry/prompts/selection-score.md`, `industry/selection.ts` |
| 한국어 | `SITE.locale`, prompt 출력 언어, UI 문자열 | `site/`, `industry/prompts/`, `apps/web/` |

`COMMUNITY_FEEDS`는 dev.to와 Hacker News의 계정 규칙만 지원한다(`packages/backend/src/events/hot.ts`). Reddit 같은 커뮤니티를 계정 단위 열기로 세려면 코드를 고쳐야 한다.

## 현재 상태 (2026-10-08)

완료:

- `sgu7788/GAMEHOT` 포크, `upstream` remote 등록, `npm ci`
- `site/site.ts` 식별값: `name` GAMEHOT, `subject` 게임, `locale` ko-KR, `mcpPrefix` gamehot, `crawlerName` GAMEHOTBot/1.0, `homeTitle`

남은 상태:

- `site/site.ts`의 나머지 문구, UI, prompt는 중국어다.
- `industry/`는 upstream의 AI 업계 예시 그대로다. 예시 신호원 18개는 모두 RSS다.
- `.env`가 없다.

로컬 검증 (sangeun4, Arch 계열 host):

| 검사 | 결과 |
|---|---|
| `npm run typecheck` | 통과 |
| `npm run build -w @aihot/web` | 통과 |
| `npm run test:standalone` | 160/161 통과. `outbound-protocol` 1건은 수정 전 트리에서도 실패한다 (sandbox 네트워크 제한 추정) |
| `node --test apps/web/tests/*.test.ts` | 32/44 통과. 실패 12건은 WebKit 실행 불가로 생긴다. Arch host에 Ubuntu 전용 라이브러리(`libicu74`, `libflite1`)가 없다 |
| `npm test` (DB 필요) | 미실행. 현재 계정이 `docker` 그룹에 없다 |

전체 검사는 GitHub Actions의 `Check` workflow가 실행한다. ubuntu runner에서 DB 테스트, 브라우저 테스트, docker compose smoke check를 돌린다.

## 처음 실행하기 전에

- 첫 `docker compose up`은 `industry/sources.json`의 신호원을 DB에 넣는다. 나중에 파일에서 지운 신호원은 DB에 남는다. 게임 신호원으로 바꾼 뒤에 처음 실행한다. 이미 실행했다면 `docker compose down -v`로 DB volume을 지운다.
- `.env`를 만든다: `node scripts/init-env.ts --llm-key <key>`. 기본 설정은 DeepSeek다. 다른 provider는 `.env`의 `LLM_BASE_URL`, `LLM_MODEL`, `LLM_EXTRA_JSON`을 고친다.
- 개발 중에는 `.env`의 `COLLECT_ENABLED`, `MODEL_CALLS_ENABLED`를 `false`로 둔다. 둘을 켜면 외부 수집과 유료 모델 호출이 시작된다.
- 현재 계정은 `docker` 그룹에 없다. `sudo usermod -aG docker $USER`를 실행하고 다시 로그인한다.

## 작업 로드맵 (제안 순서)

1. 한국 시간 기준
   - 발행 시각, 날짜 경계, cron이 `Asia/Shanghai`(UTC+8)로 고정되어 있다.
   - `packages/contracts/src/time.ts`의 `beijing*` 함수, `apps/worker/src/schedules.ts`, `packages/backend/src/`의 일부를 KST(UTC+9)로 바꾼다.
2. 한국어화
   - `site/site.ts` 문구, `site/pages/`, `site/public/`, `site/changelog.json`
   - `apps/web/app` UI 문자열(중국어 포함 줄 약 1,300개), `packages/backend/src`(약 500줄, 리포트·RSS·MCP 문구)
   - `withSubject`, `subjectAfter`는 한글 subject 뒤에 공백을 넣지 않는다. 한국어 띄어쓰기에 맞게 고친다.
   - OG 이미지 폰트 `assets/og-fonts/noto-sans-sc-*`를 한글 폰트로 바꾼다(`packages/backend/src/media/og.ts`). 리포트 nameplate도 다시 만든다(`scripts/nameplates.ts`).
3. 업계 정의 (`industry/`)
   - `taxonomy.ts`: 카테고리, 태그, `ENTITIES`(퍼블리셔·개발사), `RELEASE`
   - `topics.json`: 회사·장르·플랫폼 주제
   - `sources.json`: 공식 공지, 매체, 커뮤니티 신호원
   - taxonomy를 바꾸면 `tests/`의 AI 업계 예시를 게임 예시로 바꾼다.
4. Prompt (`industry/prompts/`)
   - 출력 언어를 한국어로 바꾼다. 10개 파일이 중국어 출력을 지정한다.
   - `rules-domain.md`에 게임 용어의 번역·원문 유지 규칙을 쓴다.
   - `selection-score.md`의 중요·노이즈 예시를 게임 기준으로 바꾼다. 내용 유형, 5개 차원 가중치, 노이즈 억제, 안전 경계 구조는 유지한다.
5. 문턱 보정: 직접 라벨링한 100~200건으로 `scripts/eval-selection.ts`를 실행한다(`docs/selection.md`).
6. 브랜드와 약관: `site/brand/`, `site/pages/terms.md`, `site/pages/privacy.md`
7. 배포: 배포 위치를 정한 뒤 `docs/deploy.md`를 따른다.

## 사용자가 정할 항목

upstream `AGENTS.md`는 아래 항목을 에이전트가 정하지 않고 사용자에게 묻도록 한다.

- 추적 범위: 국내·글로벌, 플랫폼(PC·콘솔·모바일), 주요 게임과 회사
- 신호원 목록과 등급(`T1`, `T1_5`, `T2`, `hot_signal`)
- 중요한 소식과 노이즈의 기준
- 카테고리 체계
- 약관과 개인정보 처리방침 내용

이 포크에서 추가로 정할 항목:

- LLM provider와 model
- 데일리·위클리·먼슬리 리포트 발행 시각
- 공개 범위(사내 전용 또는 공개)와 배포 위치
- `site/site.ts`의 `github` 링크 노출 여부

## Upstream 동기화

```bash
git fetch upstream
git merge upstream/main
```

- 이 포크는 `site/`, `industry/`, `GAMEHOT.md`를 소유한다. 충돌이 나면 포크 쪽 내용을 기준으로 upstream 변경을 반영한다.
- `AGENTS.md` 첫머리와 `CLAUDE.md`에 GAMEHOT 안내를 넣었다. upstream이 같은 줄을 바꾸면 충돌한다.
- 시간대와 UI 한국어화는 `packages/`, `apps/`를 고친다. 이 변경은 upstream 병합 때 충돌 범위가 넓다. 가능하면 설정값이나 `modules/`로 분리한다.
