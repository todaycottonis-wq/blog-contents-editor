# Layered Studio (구 코튼카드) — 인스타 카드뉴스 에디터

블로그 / 질문답변 / 사전 화면처럼 보이는 카드(4:5, 1080×1350)를 만들어 PNG로 내려받는 웹 도구.
소유자: 코튼(ㅇㅅㅇ), 한국어 사용. 응답·UI 문구는 한국어.

## 현재 상태 (2026-10-30)
- **단일 파일 `index.html`** (HTML+CSS+JS, 빌드 없음, 프레임워크 없음). 전체 스크립트는 하나의 IIFE 안이라 전역에서 함수가 안 보임.
- 배포: GitHub `todaycottonis-wq/blog-contents-editor` → Vercel 자동 배포. (Claude 웹 세션의 Vercel 연동은 쓰기 403이라 배포는 사용자가 push로.)
- 화면(해시 라우터 `route()`): `#/` 랜딩(`v-home`) · `#/workspace`(`v-ws`) · `#/templates`(`v-tpl`) · `#/editor`(`v-editor`). 알 수 없는 해시는 워크스페이스로.
- 흐름: 랜딩 "바로 시작하기"/로그인/회원가입/유형별 바로가기(`data-login`) → 로그인 창(Supabase: Google OAuth + 이메일 링크) → 로그인 성공 시 `#/workspace`. 이미 로그인 상태면 바로 워크스페이스(유형 버튼은 해당 유형 빈 에디터).
- "+ 새로 만들기"(`data-new`) = 아무 형식도 아닌 **빈 캔버스**(`blankDoc`, `doc.blank`) → 캔버스 안 프레임 선택(`.fp`) → `fillFrame`이 빈 입력 칸을 채움. 예시 글은 템플릿(`makeTemplate`)에서만 나옴.
- 디자인: 흰 배경 · 검정 버튼 · Pretendard · 라이트. 포인트 연두 `#E0FFBA`는 "✦ 글로 한번에 만들기" 버튼 등 일부에만. 이름은 **Layered Studio**.

## 코드 구조 메모
- 문서 모델: `doc={title,platform('blog'|'qna'|'dict'),blank?,header:{blogName,fontSize,height,gap},dict,frame,global,ts,pages:[{id,valign,blocks}]}`, `assets={id:dataURL}`, 스타일은 `rs(b)` = `{...doc.ts[type], ...b.style}`. 블록은 `blockEl(b,mode)`가 `.blk.b-<type>` 안 `.tx`에 렌더.
- 렌더러 하나: `renderPage(page, mode)` (mode: edit/thumb/export). 썸네일은 `miniCard(d,assets,pageIdx,width)`가 전역 `doc`/`assets`를 잠깐 바꿔 그림.
- 내보내기: 화면 밖 1080×1350 홀더 → `htmlToImage.toBlob` → 단일 PNG / 여러 장 JSZip / "페이지 선택해서 내보내기". 글꼴은 Google Fonts를 `&text=`로 받아 인라인. **실제 브라우저 PNG 품질은 아직 한 번도 검증 못 함(샌드박스 네트워크 차단).**
- 저장: IndexedDB `bse_drafts`/`d`(`DB` 헬퍼), 레코드 `{id,title,platform,pages,updated,doc,assets}`. `draftSave(silent)`, `edited` 플래그. localStorage: `bse_draft_id`, `bse_migrated`. **작업은 브라우저에만 저장됨(계정과 무관).**
- CSS 레이어(뒤가 앞을 덮음): v2 토스톤 → v3 "어도비 다크"를 라이트로 치환한 블록 → `/* ===== 랜딩 (라이트) ===== */`(`lp-*`, `#v-home` 범위). 옛 랜딩 CSS(`.hero`,`.sec` 등)는 죽은 코드.
- 클래스 충돌 주의: `.lg .wide .ph .cta .note .page .btn .brand`는 옛 CSS에 이미 정의됨. 새 클래스는 접두어(`lp-`, `lgb`, `lmg`) 사용.
- 질문답변형 블록: `qtitle,qmeta,qbody,qtags,qdate,qacts,qanswerer,qreact,qnote`. 조회수·채택률·좋아요는 랜덤 생성(`rViews` 등) — 고정 요소.
- 랜딩 미리보기는 `assets/*_544x432.mp4`를 base64로 `<video><source>`에 내장(mp4+webm). 영상을 바꾸면 다시 인코딩해 교체.
- 로그인 헬퍼: `openLogin()`, `initAuth()`(`user`, `onAuthStateChange`), `paintAuth()`. 클라우드 저장 코드(`cloudSave` 등)는 비활성 상태로 남아 있음.

## Supabase
- 프로젝트 `nhcnbfouudvgdaxnedwy` (`https://nhcnbfouudvgdaxnedwy.supabase.co`). 공개(anon) 키는 `index.html`에 있음 — 클라이언트 공개용이라 OK, **service_role 키는 절대 넣지 말 것**.
- 표: `public.events`(익명 사용 통계 `track()`), `public.projects`(미사용). RLS 켜짐, 보안 점검 이슈 없음. 저장소 버킷 `user-assets`.
- 대시보드에서 사용자가 해야 할 설정: Auth → URL Configuration(Site URL = Vercel 주소, Redirect URLs에 주소와 `주소/**`), Email 제공자 켜기(운영 전 Custom SMTP), Google 제공자(Client ID/Secret; 리디렉션 URI `https://nhcnbfouudvgdaxnedwy.supabase.co/auth/v1/callback`).

## 테스트 요령 (샌드박스/CI)
- Playwright로 `file://.../index.html`을 열고 `http*` 요청은 abort. 전역 접근이 필요하면 테스트용 복사본에서 IIFE 끝(`})();` 직전)에 `window.__x={makeTemplate,renderPage,...}`를 주입.
- 로그인은 `window.supabase`를 `add_init_script`로 가짜 주입해 모달/이동만 검증.
- 영상·이미지 캡처: 카드 요소만 element screenshot, 글꼴은 Pretendard를 로컬에서 로드(npm `pretendard`).

## 다음 작업: Buffer API로 인스타 자동 발행 (방법 A)
목표: 에디터에서 만든 캐러셀을 "인스타그램에 발행" 버튼으로 Buffer를 통해 올린다.
1. **먼저 검증(스파이크)**: https://developers.buffer.com/ 문서에서 ① 신규 개발자 API 키 발급 조건/요금제 ② GraphQL 스키마에서 **인스타 다중 이미지(캐러셀) 게시** 지원 여부 ③ 이미지 입력 방식(공개 URL?) ④ 레이트 리밋. 사용자는 Buffer 설정 > API(https://publish.buffer.com/settings/api)에서 키 발급 가능한지 확인해야 함. 캐러셀이 안 되면 방법 B(인스타 Graph API 직접)로 전환 논의.
2. **이미지 준비**: 각 페이지를 JPEG(또는 Buffer가 받는 형식)로 렌더 → Supabase Storage 공개 버킷에 업로드 → 공개 URL 목록 생성. (인스타는 첫 장 비율로 크롭됨, 4:5 유지 여부 실측 필요.)
3. **서버 코드**: Supabase Edge Function(또는 Vercel 함수)에서 Buffer 호출. **Buffer 토큰은 서버 환경변수/시크릿에만 보관, 클라이언트·index.html·저장소에 절대 넣지 말 것.** 호출자는 Supabase 로그인 JWT로 검증.
4. **UI**: 다운로드 메뉴 옆 "인스타그램에 발행" → 캡션 입력, 예약 시간(선택), 채널 선택, 결과/에러 표시.
5. 사용자 계정별로 Buffer 토큰을 받으려면 OAuth가 필요한지 확인(본인 한 계정용이면 단일 시크릿으로 충분).
- 파일 구조가 커지면 `index.html` 단일 파일 → `index.html` + `api/`(함수) 정도로 분리. 분리하기 전에 사용자와 상의.

## 기타 미해결
- PNG 내보내기 실브라우저 검증, 모바일 에디터(현재 데스크톱 전용), 템플릿 6개뿐, 빈 프레임 초안이 워크스페이스에 쌓임(자동 정리 제안만 함), 개인정보 처리방침 링크 없음, 클라우드 저장(계정별 동기화) 미구현.
- 랜딩의 요금제 카드/Sign up 버튼은 Figma 템플릿 문구 그대로이며 `data-soon`(준비 중 토스트).

## 작업 규칙
- 사용자는 GitHub에서 `index.html`을 직접 덮어써 배포함. 큰 변경은 백업 후 진행, 변경 요약은 한국어로 짧게.
- 로고·상표 모방 금지(네이버 등 실제 서비스 이름/로고 쓰지 않음). UI 느낌만 참고.
