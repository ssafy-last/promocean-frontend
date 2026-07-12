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

먼저 인증 스토어에 로그인 여부와 별개인 하이드레이션 상태를 두었습니다. `partialize`를 사용해 로그인에 필요한 최소 정보만 브라우저 저장소에 유지하고, `onRehydrateStorage`에서 복원 완료 시점을 표시했습니다.

```ts
export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      isLoggedIn: false,
      user: null,
      token: null,
      hasHydrated: false,
      // ...
    }),
    {
      name: 'auth-storage',
      partialize: (state) => ({
        isLoggedIn: state.isLoggedIn,
        user: state.user,
      }),
      onRehydrateStorage: () => (state) => {
        return () => {
          state!.hasHydrated = true;
        };
      },
    },
  ),
);
```

가드에서는 클라이언트가 마운트되기 전에는 리다이렉트 로직 자체를 실행하지 않습니다. 마운트 이후에도 로그인 정보가 없다면 로그인 화면으로 이동시키고, 인증 확인 중에는 자식 UI를 렌더링하지 않아 보호 화면이 순간적으로 노출되는 것을 막았습니다.

```tsx
const { user, isLoggedIn } = useAuthStore();
const [isHydrated, setIsHydrated] = useState(false);

useEffect(() => {
  setIsHydrated(true);
}, []);

useEffect(() => {
  if (!isHydrated) return;

  if (!isLoggedIn || !user) {
    router.push(redirectTo);
  }
}, [isHydrated, isLoggedIn, user, router, redirectTo]);

if (!isHydrated || !isLoggedIn || !user) {
  return null;
}
```

가챠 페이지처럼 persist 복원 상태를 직접 구독하는 화면은 `hasHydrated`가 `false`인 동안 판정을 중단합니다.

```tsx
const { user, isLoggedIn, hasHydrated } = useAuthStore();

useEffect(() => {
  if (!hasHydrated) return;

  if (!isLoggedIn || !user) {
    router.push('/auth/login?tab=login');
  }
}, [hasHydrated, isLoggedIn, user, router]);
```

**관련 코드**

- [`src/store/authStore.ts`](src/store/authStore.ts): persist 대상과 복원 완료 상태 관리
- [`src/components/auth/AuthGuard.tsx`](src/components/auth/AuthGuard.tsx): 클라이언트 마운트 전 인증 판정 및 렌더링 보류
- [`src/app/gacha/page.tsx`](src/app/gacha/page.tsx): `hasHydrated` 이후 사용자 검증

### 2. 확장자를 위장한 비허용 파일 업로드

**문제**
파일명 확장자나 브라우저가 전달하는 MIME 타입만 검사하면 GIF·MP4 등의 파일명을 `.jpg` 또는 `.png`로 바꿔 업로드 검사를 우회할 수 있습니다. 프론트엔드 검증만으로 보안을 완전히 보장할 수는 없지만, 잘못된 파일이 업로드 단계까지 진행되면 사용자 경험이 나빠지고 서버의 불필요한 처리도 늘어납니다.

**해결**
파일의 앞부분을 `ArrayBuffer`로 읽고 JPEG, PNG, WebP의 매직 바이트와 대조하는 간단한 파일 시그니처 검증을 추가했습니다. 허용되지 않은 시그니처는 즉시 거절하고 입력값을 초기화하며, 확장자와 감지된 실제 형식이 다르면 사용자에게 경고해 계속 진행할지 선택하도록 했습니다. 또한 10MB 크기 제한을 함께 확인했습니다. 이 검사는 빠른 UX 피드백을 담당하며, 최종 보안은 백엔드에서도 MIME·시그니처·디코딩 가능 여부 등을 재검증하도록 협업해 다중 방어 구조로 강화했습니다.

검증 유틸리티는 파일 전체를 업로드하거나 읽지 않고, 판별에 필요한 앞 12바이트만 읽습니다. 이후 각 형식의 고유한 매직 바이트와 실제 바이트 배열을 비교합니다.

```ts
const IMAGE_SIGNATURES = {
  jpg: [
    [0xff, 0xd8, 0xff, 0xe0],
    [0xff, 0xd8, 0xff, 0xe1],
    [0xff, 0xd8, 0xff, 0xe2],
  ],
  png: [[0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]],
};

async function checkFileSignature(
  file: File,
  signatures: number[][],
): Promise<boolean> {
  return new Promise((resolve) => {
    const reader = new FileReader();

    reader.onloadend = () => {
      if (!reader.result) return resolve(false);

      const bytes = new Uint8Array(reader.result as ArrayBuffer);
      const isValid = signatures.some((signature) =>
        signature.every((byte, index) => bytes[index] === byte),
      );

      resolve(isValid);
    };

    reader.onerror = () => resolve(false);
    reader.readAsArrayBuffer(file.slice(0, 12));
  });
}
```

WebP는 `RIFF` 헤더만 확인하면 다른 RIFF 컨테이너를 잘못 허용할 수 있으므로, 8~11번째 바이트의 `WEBP` 문자열까지 함께 검사했습니다.

```ts
const isRIFF =
  arr[0] === 0x52 && arr[1] === 0x49 &&
  arr[2] === 0x46 && arr[3] === 0x46;

const isWEBP =
  arr[8] === 0x57 && arr[9] === 0x45 &&
  arr[10] === 0x42 && arr[11] === 0x50;

resolve(isRIFF && isWEBP);
```

컴포넌트에서는 검증 실패 시 파일 입력을 초기화해 그대로 제출되지 않게 했습니다. 시그니처는 허용 형식이지만 확장자와 실제 형식이 다르면 사용자에게 불일치를 알린 뒤 명시적인 확인을 받습니다.

```tsx
const validation = await validateImageFile(file);

if (!validation.isValid) {
  alert(validation.error || '유효하지 않은 파일입니다.');
  e.target.value = '';
  return;
}

const fileExtension = getFileExtension(file.name);
const detectedType = validation.detectedType;

if (
  detectedType &&
  fileExtension !== detectedType &&
  !(fileExtension === 'jpeg' && detectedType === 'jpg')
) {
  const confirmUpload = confirm(
    `파일 확장자(${fileExtension})와 실제 파일 형식(${detectedType})이 일치하지 않습니다.`,
  );

  if (!confirmUpload) {
    e.target.value = '';
    return;
  }
}
```

**관련 코드**

- [`src/utils/fileValidation.ts`](src/utils/fileValidation.ts): JPEG·PNG·WebP 시그니처 및 파일 크기 검증
- [`src/components/button/ImageChoiceButton.tsx`](src/components/button/ImageChoiceButton.tsx): 실패 안내, 입력 초기화, 확장자 불일치 경고 UX
- [`src/api/upload.ts`](src/api/upload.ts): 검증을 통과한 파일의 업로드 요청

### 3. 실시간 검색 요청으로 인한 서버 부하

**문제**
Elasticsearch 기반 실시간 검색 API를 입력 변경 이벤트에 바로 연결하면, 사용자가 검색어를 완성하기 전의 무의미한 입력마다 요청이 발생합니다. 빠르게 타이핑할수록 취소되거나 곧바로 쓸모없어지는 요청이 누적되어 서버와 네트워크에 불필요한 부담을 줄 수 있었습니다.

**해결**
검색어가 변경될 때 1초 타이머를 시작하고, 그 안에 다음 입력이 들어오면 기존 타이머를 `clearTimeout`으로 취소하는 디바운싱을 적용했습니다. 입력이 잠시 멈췄을 때만 자동완성 API를 호출하며, 빈 검색어나 태그 검색이 아닌 경우에는 요청하지 않습니다. 자동완성 항목을 선택해 검색어가 바뀐 경우에도 `skipApiCall`로 중복 호출을 막았습니다.

`keyword`가 바뀔 때마다 effect의 정리 함수가 이전 타이머를 취소합니다. 마지막 입력 이후 1초 동안 추가 입력이 없을 때만 API 호출부가 실행되므로, `onChange` 횟수와 실제 요청 횟수를 분리할 수 있었습니다.

```tsx
const timeout = useRef<NodeJS.Timeout | null>(null);
const skipApiCall = useRef(false);

useEffect(() => {
  if (selected !== '태그' || !keyword.trim()) {
    setShowSuggestions(false);
    setTagSuggestions([]);
    return;
  }

  if (skipApiCall.current) {
    skipApiCall.current = false;
    return;
  }

  timeout.current = setTimeout(async () => {
    try {
      const res = await TagAPI.getTagAutoCompleteList({
        keyword,
      });
      setTagSuggestions(res.data);
      setShowSuggestions(true);
    } catch (error) {
      console.error('Failed to fetch tag suggestions:', error);
      setTagSuggestions([]);
    }
  }, 1000);

  return () => {
    if (timeout.current) {
      clearTimeout(timeout.current);
    }
  };
}, [keyword, selected]);
```

자동완성 결과를 클릭하면 그 선택으로 인해 `keyword`가 다시 변경됩니다. 이 변경은 사용자의 추가 검색 입력이 아니므로 `skipApiCall` 플래그를 한 번 소비해 같은 검색어로 API가 재호출되는 것도 방지했습니다.

```tsx
const handleSuggestionClick = (tagName: string) => {
  skipApiCall.current = true;
  setKeyword(tagName);
  setShowSuggestions(false);
  setSelectedSuggestionIndex(-1);
};
```

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

각 story는 대상 컴포넌트와 문서 설명을 `Meta`에 연결하고 `autodocs` 태그를 지정했습니다. 덕분에 컴포넌트 구현과 사용 예시가 함께 문서화됩니다.

```tsx
import type { Meta, StoryObj } from '@storybook/react';
import HeroSection from '@components/section/HeroSection';

const meta: Meta<typeof HeroSection> = {
  title: 'Components/Section/HeroSection',
  component: HeroSection,
  parameters: {
    layout: 'fullscreen',
    docs: {
      description: {
        component: '메인 페이지의 히어로 섹션 컴포넌트입니다.',
      },
    },
  },
  tags: ['autodocs'],
};

export default meta;
type Story = StoryObj<typeof HeroSection>;

export const Default: Story = {};
```

페이지를 직접 조작하지 않아도 같은 컴포넌트를 모바일·태블릿 너비나 다른 배경에서 비교할 수 있도록 decorator로 렌더링 조건을 분리했습니다.

```tsx
export const MobileView: Story = {
  decorators: [
    (Story) => (
      <div className="max-w-md mx-auto">
        <Story />
      </div>
    ),
  ],
};

export const TabletView: Story = {
  decorators: [
    (Story) => (
      <div className="max-w-3xl mx-auto">
        <Story />
      </div>
    ),
  ],
};
```

Storybook 실행과 정적 결과물 생성도 프로젝트 스크립트로 통일했습니다.

```json
{
  "scripts": {
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  }
}
```

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

팀 스페이스 진입 시에는 URL을 신뢰 가능한 기준으로 두고, 현재 Zustand 상태가 없거나 다른 스페이스를 가리키는지 먼저 검사합니다. 불일치하면 서버에서 접근 가능한 스페이스 목록을 다시 받아 URL의 ID와 일치하는 항목으로 상태를 복구합니다.

```tsx
const params = useParams();
const spaceIdFromUrl = Number(params.spaceId);
const currentSpace = spaceStore.currentSpace;

if (!currentSpace || currentSpace.spaceId !== spaceIdFromUrl) {
  const teamSpaceList = await SpaceAPI.getTeamSpaceList();
  const spaces = teamSpaceList?.spaces || [];

  spaceStore.setAllTeamSpaces(spaces);

  const targetSpace = spaces.find(
    (space) => space.spaceId === spaceIdFromUrl,
  );

  if (!targetSpace) {
    router.push('/team-space');
    return;
  }

  spaceStore.setCurrentSpace(targetSpace);
}
```

폴더 페이지도 같은 방식으로 URL의 `folderId`와 `currentFolder`를 비교합니다. 스토어가 비었거나 다른 폴더가 저장돼 있다면 폴더 목록을 다시 조회하고, 존재하지 않는 폴더 ID라면 해당 팀 스페이스의 상위 화면으로 복귀시킵니다.

```tsx
const isAllPromptsView = params.folderId === 'all-prompts';
const folderIdFromUrl = isAllPromptsView
  ? 0
  : Number(params.folderId);

if (
  !folderStore.currentFolder ||
  folderStore.currentFolder.folderId !== folderIdFromUrl
) {
  const res = await SpaceAPI.getSpaceArchiveFoldersData(spaceIdFromUrl);
  const folders = res?.folders || [];
  folderStore.setAllFolderList(folders);

  const targetFolder = folders.find(
    (folder) => folder.folderId === folderIdFromUrl,
  );

  if (!targetFolder) {
    router.push(`/team-space/${spaceIdFromUrl}`);
    return;
  }

  targetFolder.color = `#${targetFolder.color}`;
  folderStore.setCurrentFolder(targetFolder);
}
```

글 수정 화면에서는 URL 문자열을 숫자로 변환한 결과까지 검증합니다. 잘못된 ID로는 API를 호출하지 않고 이전 화면으로 돌려보내며, 팀 스페이스 ID는 전역 상태보다 URL 파라미터를 우선 사용해 새로고침 후에도 데이터를 다시 불러올 수 있게 했습니다.

```ts
if (postType === 'team-space' && spaceIdParam) {
  spaceId = parseInt(spaceIdParam, 10);

  if (isNaN(spaceId)) {
    alert('잘못된 팀 스페이스 ID입니다.');
    router.back();
    return;
  }
} else if (spaceStore.currentSpace?.spaceId) {
  spaceId = spaceStore.currentSpace.spaceId;
} else {
  alert('스페이스 정보를 찾을 수 없습니다.');
  router.back();
  return;
}

const articleId = parseInt(articleIdParam, 10);
if (isNaN(articleId)) {
  alert('잘못된 게시글 ID입니다.');
  router.back();
  return;
}

await loadArchiveArticleData(spaceId, articleId, {
  setSelectedPromptType,
  setTitle,
  setTags,
  setDescriptionState,
  setUsedPrompt,
  setExamplePrompt,
  setAnswerPrompt,
  setUploadedImageUrl,
  setUploadedImageKey,
});
```

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

자주 사용하는 서비스 색상을 Tailwind 테마에 의미 기반 이름으로 등록했습니다. 컴포넌트가 `#0094ff` 같은 원시 색상값을 직접 알 필요 없이 `primary`, `background`, `text`처럼 용도에 맞는 이름을 사용하도록 구성했습니다.

```ts
import type { Config } from "tailwindcss";
import typography from "@tailwindcss/typography";

const config: Config = {
  content: [
    "./src/pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/components/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/app/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        primary: "#0094ff",
        secondary: "#a6fbfc",
        background: "#fdfdfc",
        text: "#343434",
        "dark-mode": "#213477",
      },
    },
  },
  plugins: [typography],
} satisfies Config;

export default config;
```

등록한 색상은 일반 Tailwind 유틸리티와 동일하게 조합할 수 있습니다. 예를 들어 검색어 강조에는 `text-primary`, 버튼에는 `bg-primary`를 사용하고, 투명도나 hover 상태도 별도의 CSS 선언 없이 함께 표현했습니다.

```tsx
<span className="font-bold text-primary">
  {part}
</span>

<button className="bg-primary text-white hover:bg-primary/90 rounded-lg">
  시작하기
</button>
```

이를 통해 색상 정책이 변경되더라도 설정의 토큰값만 수정하면 해당 클래스를 사용하는 화면에 일괄 반영되며, 여러 개발자가 동시에 작업할 때도 같은 색상 체계를 유지할 수 있었습니다.

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
