# Promocean

> **생성형 AI 기반의 프롬프트 공유·관리 플랫폼**

Promocean은 생성형 AI 프롬프트와 그 결과물을 작성하고 공유하며, 개인 또는 팀 단위로 체계적으로 관리할 수 있는 플랫폼입니다. 커뮤니티, 개인·팀 스페이스, 프롬프트 대회, 가챠 및 실시간 알림을 하나의 서비스에서 제공합니다.

## 주요 기능

- 프롬프트 및 생성 결과물 작성·공유
- 카테고리, 태그, 정렬 조건을 활용한 커뮤니티 검색
- 개인 스페이스와 팀 스페이스 기반의 폴더·아카이브 관리
- 프롬프트 대회 개최, 참가작 등록 및 상세 조회
- 마일리지를 이용한 이모티콘 가챠
- SSE(Server-Sent Events) 기반 실시간 알림
- Lexical 기반 리치 텍스트 에디터
- Storybook을 활용한 UI 컴포넌트 문서화 및 미리보기

## 기술 스택

| 영역 | 기술 |
| --- | --- |
| Language | TypeScript |
| Framework | Next.js 15, React 19 |
| State Management | Zustand, Zustand Persist |
| Styling | Tailwind CSS |
| UI Documentation | Storybook |
| Editor | Lexical |

## 담당 역할

| 이름 | 담당 영역 |
| --- | --- |
| 정태승 | 메인 페이지, 개인 스페이스·팀 스페이스 페이지, 글쓰기 페이지, 가챠 추첨 페이지, 실시간 알림(SSE) 연동 구현 |
| 이재환 | 메인 페이지, 커뮤니티 페이지, 글 상세 페이지, 대회 페이지, 대회 상세·등록 페이지, 사이드바 구현 |

## 프로젝트 구조

```text
src/
├─ app/          # Next.js App Router 기반 페이지 및 레이아웃
├─ api/          # 도메인별 API 요청 모듈
├─ components/   # 공통 UI 컴포넌트와 Storybook stories
├─ hooks/        # 게시글, 이미지, 알림 관련 커스텀 훅
├─ store/        # Zustand 전역 상태
└─ utils/        # 파일 검증, 업로드, 게시글 처리 유틸리티
```

## 이슈와 해결

### 1. Zustand Persist 복원 전에 실행되는 네비게이션 가드

**문제**
Next.js는 서버에서 만든 초기 화면과 클라이언트의 상태를 결합하는 하이드레이션 과정을 거칩니다. 이때 로컬 스토리지의 로그인 상태가 Zustand Persist에서 복원되기 전에 가드가 먼저 실행되면, 실제 로그인 사용자도 비로그인 상태로 판단되거나 보호 페이지가 그대로 통과하는 등 잘못된 흐름이 발생할 수 있었습니다.

**해결**
인증 스토어에 `hasHydrated` 상태와 `onRehydrateStorage` 콜백을 두어 persist 복원 완료 시점을 구분했습니다. 보호 페이지는 하이드레이션이 끝나기 전에는 인증 판정과 화면 렌더링을 보류하고, 복원 이후의 `isLoggedIn`과 `user`를 기준으로 이동 여부를 결정하도록 구성했습니다. 가챠와 마이페이지처럼 인증이 필요한 화면에서도 같은 원칙을 적용해 초기 상태를 로그인 상태로 오인하지 않도록 했습니다.

**관련 코드**

- [`src/store/authStore.ts`](src/store/authStore.ts): persist 대상과 복원 완료 상태 관리
- [`src/components/auth/AuthGuard.tsx`](src/components/auth/AuthGuard.tsx): 클라이언트 마운트 전 인증 판정 및 렌더링 보류
- [`src/app/gacha/page.tsx`](src/app/gacha/page.tsx): `hasHydrated` 이후 사용자 검증

### 2. 확장자를 위장한 비허용 파일 업로드

**문제**
파일명 확장자나 브라우저가 전달하는 MIME 타입만 검사하면 GIF·MP4 등의 파일명을 `.jpg` 또는 `.png`로 바꿔 업로드 검사를 우회할 수 있습니다. 프론트엔드 검증만으로 보안을 완전히 보장할 수는 없지만, 잘못된 파일이 업로드 단계까지 진행되면 사용자 경험이 나빠지고 서버의 불필요한 처리도 늘어납니다.

**해결**
파일의 앞부분을 `ArrayBuffer`로 읽고 JPEG, PNG, WebP의 매직 바이트와 대조하는 간단한 파일 시그니처 검증을 추가했습니다. 허용되지 않은 시그니처는 즉시 거절하고 입력값을 초기화하며, 확장자와 감지된 실제 형식이 다르면 사용자에게 경고해 계속 진행할지 선택하도록 했습니다. 또한 10MB 크기 제한을 함께 확인했습니다. 이 검사는 빠른 UX 피드백을 담당하며, 최종 보안은 백엔드에서도 MIME·시그니처·디코딩 가능 여부 등을 재검증하도록 협업해 다중 방어 구조로 강화했습니다.

**관련 코드**

- [`src/utils/fileValidation.ts`](src/utils/fileValidation.ts): JPEG·PNG·WebP 시그니처 및 파일 크기 검증
- [`src/components/button/ImageChoiceButton.tsx`](src/components/button/ImageChoiceButton.tsx): 실패 안내, 입력 초기화, 확장자 불일치 경고 UX
- [`src/api/upload.ts`](src/api/upload.ts): 검증을 통과한 파일의 업로드 요청

### 3. 실시간 검색 요청으로 인한 서버 부하

**문제**
Elasticsearch 기반 실시간 검색 API를 입력 변경 이벤트에 바로 연결하면, 사용자가 검색어를 완성하기 전의 무의미한 입력마다 요청이 발생합니다. 빠르게 타이핑할수록 취소되거나 곧바로 쓸모없어지는 요청이 누적되어 서버와 네트워크에 불필요한 부담을 줄 수 있었습니다.

**해결**
검색어가 변경될 때 1초 타이머를 시작하고, 그 안에 다음 입력이 들어오면 기존 타이머를 `clearTimeout`으로 취소하는 디바운싱을 적용했습니다. 입력이 잠시 멈췄을 때만 자동완성 API를 호출하며, 빈 검색어나 태그 검색이 아닌 경우에는 요청하지 않습니다. 자동완성 항목을 선택해 검색어가 바뀐 경우에도 `skipApiCall`로 중복 호출을 막았습니다.

쓰로틀은 입력 중에도 일정 주기로 요청을 계속 보내므로 “사용자가 입력을 마친 시점에 검색한다”는 의도를 정확히 표현하기 어렵다고 판단했습니다. 검색 결과의 즉각적인 연속 갱신보다 최종 검색어의 정확성과 요청 절감이 중요한 이 기능에는 디바운스가 더 적합했습니다.

**관련 코드**

- [`src/components/filter/CombinedSearchFilter.tsx`](src/components/filter/CombinedSearchFilter.tsx): 공통 검색 필터의 디바운싱과 중복 요청 방지
- [`src/components/filter/MySpaceMyPostFilter.tsx`](src/components/filter/MySpaceMyPostFilter.tsx): 개인 게시글 검색 자동완성
- [`src/components/filter/SpaceArchiveFilter.tsx`](src/components/filter/SpaceArchiveFilter.tsx): 아카이브 검색 자동완성
- [`src/api/tag.ts`](src/api/tag.ts): 태그 자동완성 API 요청

### 4. UI 컴포넌트의 디자인 공유와 검토 비용

**문제**
개발자마다 UI 컴포넌트의 디자인과 상태를 확인하려면 실제 페이지를 실행하고 해당 화면까지 이동해야 했습니다. 재사용 컴포넌트의 기본·로딩·빈 데이터·반응형 상태를 한눈에 비교하기 어렵고, 구현자 외의 팀원이 변경 영향을 파악하는 데에도 시간이 필요했습니다.

**해결**
Storybook을 도입해 컴포넌트를 애플리케이션과 분리된 환경에서 확인하고 문서화했습니다. 버튼, 폼, 필터, 레이아웃, 아이템, 리스트, 섹션, 모달 등 계층별 stories를 작성하고, `autodocs`와 viewport별 시나리오를 활용해 상태와 반응형 디자인을 빠르게 검토할 수 있도록 했습니다.

반복적인 stories 초안 작성은 Claude에 작업 규칙과 컴포넌트 인터페이스를 제공해 이관했습니다. 이후 개발자가 실제 렌더링, 상호작용, 디자인 일관성을 검수하는 방식으로 역할을 나눠 단순 문서화 시간을 줄이고 핵심 기능 개발에 집중했습니다.

**관련 코드**

- [`src/components/section/stories/HeroSection.stories.tsx`](src/components/section/stories/HeroSection.stories.tsx): 기본·모바일·태블릿·다크 배경 등 화면 시나리오
- [`src/components/form/stories/SignInForm.stories.tsx`](src/components/form/stories/SignInForm.stories.tsx): 폼 컴포넌트 문서화 예시
- [`src/components/layout/stories/Sidebar.stories.tsx`](src/components/layout/stories/Sidebar.stories.tsx): 공통 레이아웃 미리보기 예시
- [`package.json`](package.json): Storybook 실행 및 정적 빌드 스크립트

### 5. 예상하지 못한 URL 직접 진입과 전역 상태 불일치

**문제**
목록에서 카드를 선택해 상세 화면으로 이동하는 정상 흐름만 고려하면 Zustand에 현재 스페이스와 폴더가 이미 저장되어 있다고 가정하기 쉽습니다. 하지만 URL 직접 입력, 새로고침, 북마크, 뒤로 가기와 같이 다양한 경로로 진입하면 메모리 상태가 비어 있거나 URL과 다른 대상을 가리킬 수 있었습니다. 그 결과 잘못된 정보를 보여 주거나 API 요청에 필요한 ID를 얻지 못하는 문제가 발생했습니다.

**해결**
URL의 `spaceId`·`folderId`를 화면 상태의 기준으로 삼고, Zustand의 `currentSpace`·`currentFolder`와 일치하는지 먼저 검사했습니다. 상태가 없거나 ID가 다르면 팀 스페이스 및 폴더 목록을 다시 조회해 URL에 해당하는 항목으로 스토어를 복구합니다. 존재하지 않거나 접근 권한이 없는 ID라면 안전한 상위 경로로 이동시킵니다. 글쓰기·수정 화면에서도 URL 파라미터를 파싱하고 ID 유효성을 검사한 뒤 필요한 데이터를 다시 가져오도록 해 특정 이동 순서에 대한 의존성을 줄였습니다.

**관련 코드**

- [`src/app/team-space/[spaceId]/page.tsx`](src/app/team-space/%5BspaceId%5D/page.tsx): URL과 스페이스 상태 대조 및 목록 재조회
- [`src/app/team-space/[spaceId]/[folderId]/page.tsx`](src/app/team-space/%5BspaceId%5D/%5BfolderId%5D/page.tsx): 폴더 상태 복구와 잘못된 접근 처리
- [`src/app/team-space/[spaceId]/[folderId]/[articleId]/page.tsx`](src/app/team-space/%5BspaceId%5D/%5BfolderId%5D/%5BarticleId%5D/page.tsx): 직접 진입 시 스페이스 확인 후 상세 데이터 조회
- [`src/hooks/usePostDataLoader.ts`](src/hooks/usePostDataLoader.ts): 글 종류와 URL ID 검증 및 수정 데이터 재조회

이 경험을 통해 프론트엔드는 기대한 사용자 흐름만 구현하는 것이 아니라, 상태 유실과 비정상적인 진입 순서를 기본 전제로 삼는 방어적인 설계가 중요하다는 점을 배웠습니다.

### 6. 짧은 개발 기간의 디자인 일관성 확보

**문제**
짧은 기간 안에 여러 페이지를 병렬로 개발하면서 색상값과 스타일을 컴포넌트마다 직접 작성하면 구현 속도가 느려지고, 동일한 의미의 색상이 조금씩 달라지는 문제가 생길 수 있었습니다.

**해결**
Tailwind CSS의 유틸리티 클래스로 스타일 작성과 검토 시간을 줄였습니다. `tailwind.config.ts`의 `theme.extend.colors`에 `primary`, `secondary`, `background`, `text`, `dark-mode`를 디자인 토큰처럼 정의해 여러 화면에서 같은 의미와 값으로 재사용했습니다. Typography 플러그인도 적용해 게시글처럼 서식이 있는 콘텐츠의 기본 표현을 일관되게 관리했습니다.

**관련 코드**

- [`tailwind.config.ts`](tailwind.config.ts): 공통 색상 토큰과 Typography 플러그인 설정
- [`src/app/globals.css`](src/app/globals.css): 전역 스타일 및 Tailwind 적용

## 실행 방법

```bash
npm install
npm run dev
```

개발 서버는 기본적으로 `http://localhost:3000`에서 실행됩니다.

Storybook은 다음 명령으로 실행할 수 있습니다.

```bash
npm run storybook
```

Storybook 개발 서버는 기본적으로 `http://localhost:6006`에서 실행됩니다.
