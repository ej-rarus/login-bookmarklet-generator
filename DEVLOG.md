# 로그인 북마크렛 생성기 개발기 🔐

> 테스트 계정으로 반복 로그인하기 귀찮아서 만든 자동 로그인 도구

## 🎯 프로젝트 배경

웹 개발을 하다 보면 테스트 환경에서 여러 계정으로 로그인/로그아웃을 반복해야 할 때가 많습니다. 매번 아이디와 비밀번호를 입력하고 로그인 버튼을 누르는 작업이 생각보다 귀찮더라고요.

"북마크 하나로 자동 로그인되면 얼마나 편할까?" 라는 생각에서 시작된 프로젝트입니다.

### 목표
- 코딩 지식이 없는 사람도 쉽게 사용할 수 있어야 함
- HTML만 업로드하면 자동으로 북마크렛 생성
- 여러 계정을 관리하기 쉬워야 함

## 🛠 기술 스택

- **순수 HTML/CSS/JavaScript** - 별도의 프레임워크 없이 가볍게
- **브라우저 API** - FileReader, DOMParser, iframe 활용
- **SheetJS (xlsx)** - 엑셀 파일 파싱용

서버가 필요 없는 **100% 클라이언트 사이드** 애플리케이션으로 설계했습니다.

## ✨ 주요 기능

### 1. 자동 로그인 폼 감지
HTML 파일을 업로드하면 로그인 폼 요소를 자동으로 찾아냅니다.

```javascript
function detectLoginFormElements(doc) {
    // 비밀번호 필드 먼저 찾기 (가장 확실한 요소)
    const passwordInput = doc.querySelector('input[type="password"]');

    // ID 필드 찾기 - name 속성 우선 검색
    const candidates = [
        'input[name="username"]',
        'input[name="user"]',
        'input[type="email"]',
        // ... 더 많은 패턴
    ];

    // 로그인 버튼 찾기
    const submitButton = doc.querySelector('button[type="submit"]');

    // ...
}
```

### 2. 시각적 요소 선택 기능

자동 감지가 실패할 때를 대비한 **핵심 기능**입니다. 사용자가 직접 눈으로 보고 클릭해서 요소를 선택할 수 있습니다.

#### 개발 과정의 핵심 문제들

**문제 1: iframe 하이라이트가 안 보임**
- 원인: iframe 내부에 CSS가 주입되지 않음
- 해결: iframe 로드 완료 후 동적으로 스타일 주입

```javascript
function injectStylesIntoIframe(iframe) {
    const iframeDoc = iframe.contentDocument;
    const style = iframeDoc.createElement('style');
    style.textContent = `
        .element-highlight {
            outline: 3px solid #4caf50 !important;
            outline-offset: 2px;
            background: rgba(76, 175, 80, 0.1) !important;
        }
    `;
    iframeDoc.head.appendChild(style);
}
```

**문제 2: iframe이 프리즈되어 클릭이 안 됨**
- 원인: 오버레이가 iframe 위를 덮어서 클릭 이벤트 차단
- 해결: `pointer-events: none`으로 오버레이는 클릭을 통과시키고, 가이드 박스만 클릭 가능하게 설정

```css
.selection-overlay {
    pointer-events: none; /* iframe 클릭 허용 */
}

.selection-guide {
    pointer-events: auto; /* 가이드 박스는 클릭 가능 */
}
```

이 문제를 해결하는 데 가장 많은 시간이 걸렸는데, 해결하고 나니 정말 뿌듯했습니다! 🎉

**문제 3: 이벤트 리스너가 불안정함**
- 원인: 개별 요소에 이벤트를 붙이는 방식
- 해결: 이벤트 위임(Event Delegation) 방식으로 변경

```javascript
// Before: 개별 요소마다 이벤트 추가 (불안정)
allElements.forEach(element => {
    element.addEventListener('click', handleClick);
});

// After: 부모에 하나의 리스너만 (안정적)
iframeDoc.body.addEventListener('click', function(e) {
    if (e.target.matches('input, button, a')) {
        handleElementSelection(e.target);
    }
}, true);
```

### 3. 북마크렛 코드 생성

선택된 요소 정보를 바탕으로 북마크렛을 생성합니다.

```javascript
function createBookmarkletCode(account) {
    const code = `
(function() {
    const idInput = document.querySelector('${idSelector}');
    const pwInput = document.querySelector('${passwordSelector}');

    if (idInput) idInput.value = '${account.id}';
    if (pwInput) pwInput.value = '${account.password}';

    const btn = document.querySelector('${buttonSelector}');
    if (btn) btn.click();
})();
    `.trim();

    return `javascript:${encodeURIComponent(code)}`;
}
```

북마크 URL에 JavaScript 코드를 저장하는 방식입니다. `javascript:` 프로토콜을 사용하면 북마크 클릭 시 해당 코드가 실행됩니다.

### 4. 계정 관리

**엑셀 업로드 방식**
```javascript
function handleExcelFile(file) {
    const reader = new FileReader();
    reader.onload = (e) => {
        const data = new Uint8Array(e.target.result);
        const workbook = XLSX.read(data, { type: 'array' });
        const firstSheet = workbook.Sheets[workbook.SheetNames[0]];
        const jsonData = XLSX.utils.sheet_to_json(firstSheet);

        // ID, Password 컬럼 자동 감지 (대소문자 무시)
        accountList = jsonData.map(row => {
            const idKey = Object.keys(row).find(key =>
                key.toLowerCase().includes('id') ||
                key.toLowerCase().includes('user')
            );
            // ...
        });
    };
}
```

**수동 입력 방식**
- 동적으로 행 추가/삭제 가능
- 실시간으로 계정 개수 표시

### 5. 북마크 등록 방법

생성된 북마크렛을 브라우저에 등록하는 **세 가지 방법**을 제공합니다.

#### 방법 1: 드래그 앤 드롭 (가장 쉬움!)

HTML5 Drag and Drop API를 활용한 직관적인 방법입니다.

```html
<a href="${bookmarkletCode}" class="bookmarklet-link" draggable="true">
    📌 ${account.name}
</a>
```

사용자가 생성된 링크를 브라우저 북마크바로 드래그하기만 하면 자동으로 북마크가 등록됩니다. 별도의 복사/붙여넣기 과정이 필요 없어 **가장 빠르고 편리한 방법**입니다.

#### 방법 2: HTML 파일 내보내기 (대량 등록)

여러 계정을 한 번에 등록할 때 유용한 방법입니다. **Netscape Bookmark File Format**을 사용해 브라우저가 인식할 수 있는 북마크 파일을 생성합니다.

```javascript
function exportBookmarks() {
    let bookmarksHTML = `<!DOCTYPE NETSCAPE-Bookmark-file-1>
<META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=UTF-8">
<TITLE>Bookmarks</TITLE>
<H1>Bookmarks</H1>
<DL><p>
    <DT><H3>로그인 북마크렛</H3>
    <DL><p>`;

    allAccounts.forEach(account => {
        const bookmarkletCode = createBookmarkletCode(account);
        const escapedName = account.name
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;');
        bookmarksHTML += `        <DT><A HREF="${bookmarkletCode}">${escapedName}</A>\n`;
    });

    bookmarksHTML += `    </DL><p>\n</DL><p>`;

    // Blob으로 파일 생성 및 다운로드
    const blob = new Blob([bookmarksHTML], { type: 'text/html' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'login_bookmarklets.html';
    a.click();
}
```

**Netscape Bookmark Format**은 모든 주요 브라우저(Chrome, Firefox, Edge, Safari)에서 지원하는 표준 형식입니다. 생성된 HTML 파일을 브라우저의 "북마크 가져오기" 기능으로 불러오면 모든 북마크가 한 번에 등록됩니다.

#### 방법 3: 코드 복사 (수동)

각 북마크렛의 코드를 복사해서 수동으로 북마크를 만드는 전통적인 방법입니다.

```javascript
function copyBookmarkletCode(code, index) {
    navigator.clipboard.writeText(code).then(() => {
        showCopyFeedback(index);
    });
}
```

Clipboard API를 사용해 원클릭 복사를 구현했습니다.

### 6. UI 개선: 목록형 디자인

초기 버전의 카드 기반 UI에서 **테이블 기반 목록형 UI**로 전면 개편했습니다.

#### 변경 배경
- 카드 UI는 계정이 많을 때 세로로 길어져서 스크롤이 많이 필요했음
- 정보 밀도가 낮아 한 화면에 적은 수의 북마크만 표시됨
- 드래그 앤 드롭 링크가 시각적으로 덜 돋보였음

#### 새로운 테이블 UI

```javascript
function generateBookmarklets() {
    resultsDiv.innerHTML = `
        <table class="bookmarklet-table">
            <thead>
                <tr>
                    <th>계정 정보</th>
                    <th>북마크 드래그</th>
                    <th>동작</th>
                </tr>
            </thead>
            <tbody id="bookmarkletTableBody"></tbody>
        </table>
    `;

    allAccounts.forEach((account, index) => {
        const row = tbody.insertRow();
        row.innerHTML = `
            <td>
                <div class="account-name">${account.name}</div>
                <div class="account-id">ID: ${account.id}</div>
            </td>
            <td>
                <a href="${bookmarkletCode}" class="bookmarklet-link" draggable="true">
                    📌 ${account.name}
                </a>
            </td>
            <td>
                <button class="small-copy-btn" onclick="copyBookmarkletCode(...)">
                    📋 코드 복사
                </button>
            </td>
        `;
    });
}
```

#### 개선 효과
- **공간 효율성 300% 향상**: 같은 화면 크기에 3배 이상의 북마크 표시
- **명확한 정보 계층**: 계정 정보 → 드래그 링크 → 액션 버튼 순으로 시각적 흐름 개선
- **드래그 타겟 명확화**: 📌 아이콘으로 드래그 가능한 요소를 직관적으로 표시

## 🎨 UX 개선 과정

### 사용자 피드백 반영

**피드백 1: "HTML 파일 준비가 복잡해요"**
- 기존: 페이지 소스 보기 → 복사 → 메모장에 붙여넣기 → 저장
- 개선: 로그인 페이지에서 `Ctrl+S` 누르기만 하면 끝!

튜토리얼에도 이 내용을 강조했습니다.

**피드백 2: "코드를 모르는데 어떻게 요소를 선택하나요?"**

이 피드백이 **시각적 선택 기능**을 만든 계기였습니다.
- 코드 대신 눈으로 보고 클릭
- 초록색 하이라이트로 시각적 피드백
- 잘못 선택하면 "↶ 이전 단계" 버튼으로 되돌리기

**피드백 3: "FAQ 섹션이 디자인이 달라서 어색해요"**
- 문제: h3 태그로 질문을 표시해서 다른 섹션과 스타일이 달랐음
- 해결: FAQ도 step-card로 감싸서 디자인 통일

```html
<!-- Before -->
<h3>Q1. 북마크렛이 작동하지 않아요</h3>
<p>답변...</p>

<!-- After -->
<div class="step-card">
    <h4>Q1. 북마크렛이 작동하지 않아요</h4>
    <p>답변...</p>
</div>
```

**피드백 4: "북마크를 어떻게 등록하나요?"**
- 문제: 북마크렛 코드를 복사한 후 수동으로 북마크를 만드는 과정이 복잡했음
- 해결: 세 가지 등록 방법 제공
  1. 드래그 앤 드롭 (가장 직관적)
  2. HTML 파일 내보내기 (대량 등록)
  3. 코드 복사 (전통적 방법)

```html
<div class="method-section">
    <div class="method-number">1</div>
    <div class="method-content">
        <strong>🖱️ 드래그 앤 드롭 (가장 쉬움!)</strong>
        <p>생성된 표에서 <code>📌 계정 이름</code> 링크를 브라우저 북마크바로 드래그</p>
    </div>
</div>
```

**피드백 5: "카드 UI가 공간을 많이 차지해요"**
- 문제: 계정이 10개만 되어도 스크롤이 너무 길어짐
- 해결: 테이블 기반 목록형 UI로 재설계
  - 정보 밀도 3배 향상
  - 한눈에 더 많은 북마크 확인 가능
  - 드래그 타겟이 더 명확해짐

### CSS 최적화

UI 개편과 함께 불필요한 스타일을 대폭 정리했습니다.

**제거한 스타일**:
- `.copy-btn` (통합 복사 버튼)
- `.drag-handle` (개별 드래그 핸들)
- `.result-item` (카드 레이아웃)
- `.bookmarklet-card` (카드 컴포넌트)

**새로 추가한 스타일**:
```css
.bookmarklet-table {
    width: 100%;
    border-collapse: collapse;
    background: white;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.method-section {
    display: flex;
    gap: 15px;
    padding: 20px;
    background: #f8f9fa;
    border-radius: 8px;
}

.method-number {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    font-weight: bold;
    /* ... */
}
```

재사용 가능한 컴포넌트 중심으로 CSS를 재구성해 유지보수성을 크게 개선했습니다.

## 📱 반응형 디자인

데스크톱과 모바일 모두에서 사용 가능하도록 설계했습니다.

```css
@media (max-width: 1024px) {
    .main-wrapper {
        flex-direction: column;
    }

    .sidebar {
        width: 100%;
        position: static;
    }

    .sidebar-menu {
        display: flex;
        flex-wrap: wrap;
        gap: 10px;
    }
}
```

## 🔒 보안 고려사항

### 클라이언트 사이드 처리
- 모든 데이터 처리는 브라우저 내부에서만 이루어짐
- 서버로 파일이나 계정 정보가 전송되지 않음
- 네트워크 요청 없이 완전히 오프라인에서도 작동 가능

### 주의사항 표시
```markdown
⚠️ 주의사항
북마크렛에는 계정 정보가 평문으로 저장됩니다.
실제 프로덕션 계정이 아닌 테스트 계정에만 사용하세요!
```

튜토리얼에 경고 박스로 명확히 안내했습니다.

## 💡 배운 점

### 1. 사용자 중심 설계의 중요성
처음엔 "자동 감지만 잘 되면 되겠지"라고 생각했는데, 실제로는 다양한 로그인 폼 구조 때문에 감지가 실패하는 경우가 많았습니다. **시각적 선택 기능**을 추가하면서 "기술이 완벽하지 않더라도, 사용자가 쉽게 대안을 선택할 수 있어야 한다"는 걸 배웠습니다.

### 2. iframe의 까다로움
iframe은 보안상의 이유로 많은 제약이 있습니다:
- 같은 origin이어야 접근 가능
- sandbox 속성 설정 필요
- 스타일은 내부에 주입해야 함
- 이벤트는 위임 방식이 안정적

이런 제약들을 하나씩 해결하면서 iframe에 대한 이해도가 많이 높아졌습니다.

### 3. 점진적 개선
완벽한 결과물을 한 번에 만들려고 하지 않고:
1. 기본 기능 구현 (자동 감지)
2. 문제 발견 (감지 실패)
3. 대안 추가 (시각적 선택)
4. UX 개선 (Ctrl+S, 이전 단계 버튼)
5. 디자인 통일 (FAQ 스타일)
6. 북마크 등록 간소화 (드래그 앤 드롭)
7. UI 재설계 (테이블 형식)
8. CSS 최적화

이렇게 단계적으로 개선해나가는 과정이 더 효과적이었습니다.

### 4. 브라우저 표준 API 활용
프레임워크 없이 순수 JavaScript만으로 구현하면서 다양한 브라우저 API를 활용했습니다:
- **HTML5 Drag and Drop API**: 드래그 앤 드롭 구현
- **Blob API**: 클라이언트 사이드에서 파일 생성 및 다운로드
- **Clipboard API**: 원클릭 복사 기능
- **URL.createObjectURL()**: 메모리 효율적인 파일 다운로드

특히 **Netscape Bookmark Format**을 발견한 것이 큰 도움이 되었습니다. 1990년대부터 사용된 오래된 표준이지만, 2025년 현재까지도 모든 주요 브라우저에서 지원하는 것을 보고 **웹 표준의 힘**을 다시 한번 느꼈습니다.

### 5. 정보 밀도와 사용성의 균형
카드 UI에서 테이블 UI로 변경하면서 배운 점:
- **더 많은 정보 ≠ 복잡함**: 정보를 잘 정리하면 오히려 이해하기 쉬워짐
- **공간 효율성의 중요성**: 스크롤을 줄이면 사용자 경험이 크게 개선됨
- **시각적 계층 구조**: 테이블의 열 구조가 정보의 우선순위를 명확히 전달함

## 🎯 개선하고 싶은 점

### 완료된 기능 ✅
- [x] **북마크 내보내기/가져오기 기능**
  - HTML 파일 내보내기 (Netscape Bookmark Format)
  - 드래그 앤 드롭으로 즉시 등록
  - 코드 복사 기능
- [x] **UI 개선**
  - 목록형 테이블 디자인
  - 정보 밀도 300% 향상
  - 반응형 레이아웃

### 향후 계획
- [ ] 브라우저 확장 프로그램 버전
  - 현재: 북마크렛 방식 (제한적)
  - 목표: Chrome/Firefox 확장 프로그램으로 더 강력한 기능 제공
- [ ] 더 다양한 로그인 폼 패턴 지원
  - reCAPTCHA 우회 방법 안내
  - 2단계 인증 고려
  - SPA(Single Page Application) 로그인 폼 지원
- [ ] 다국어 지원 (영어, 일본어 등)
  - 현재: 한국어만 지원
  - 목표: 글로벌 개발자들도 사용할 수 있게
- [ ] 북마크렛 테스트 기능
  - 페이지 내에서 미리 북마크렛 동작 테스트
  - 오류 디버깅 도구 제공
- [ ] 북마크 그룹 관리
  - 프로젝트별/환경별로 북마크 그룹화
  - 그룹 단위 내보내기/가져오기

## 📊 성과

### 주요 지표
- **순수 JavaScript로 구현** (프레임워크 없음, 의존성 최소화)
- **파일 크기**: 약 50KB (압축되지 않은 상태에서도 가볍고 빠름)
- **서버 비용**: $0 (정적 호스팅만 필요, GitHub Pages 무료 배포 가능)
- **지원 브라우저**: Chrome, Firefox, Edge, Safari (모든 주요 브라우저)
- **오프라인 작동**: 네트워크 연결 없이도 100% 사용 가능

### 기능 통계
- **계정 관리 방식**: 2가지 (엑셀 업로드, 수동 입력)
- **북마크 등록 방법**: 3가지 (드래그 앤 드롭, HTML 내보내기, 코드 복사)
- **폼 요소 감지**: 자동 감지 + 시각적 수동 선택
- **지원 파일 형식**: HTML, XLSX (Excel)

### 사용자 경험 개선
**기존 방식** (수동 로그인):
- HTML 파일 준비: 30초 (소스 보기 → 복사 → 붙여넣기 → 저장)
- 북마크 만들기: 60초 (북마크 추가 → 이름 입력 → URL 붙여넣기)
- 로그인 실행: 10초 (ID 입력 → 비밀번호 입력 → 버튼 클릭)

**개선된 방식** (북마크렛):
- HTML 파일 준비: 5초 (Ctrl+S)
- 북마크 만들기: 3초 (드래그 앤 드롭)
- 로그인 실행: 1초 (북마크 클릭 한 번)

**전체 프로세스 시간 절약**: 100초 → 9초 = **91% 시간 절약** 🚀

### UI 개선 효과
- **카드 UI** (초기): 10개 계정 = 약 1800px 세로 높이
- **테이블 UI** (개선): 10개 계정 = 약 600px 세로 높이
- **스크롤 감소**: 67% ↓
- **정보 밀도**: 300% ↑

## 🎉 마무리

처음엔 "간단한 도구"로 시작했지만, 실제로 만들어보니 예상보다 고려해야 할 것들이 많았습니다. 특히 **코딩을 모르는 사용자도 쉽게 사용할 수 있게** 만드는 과정에서 많은 것을 배웠습니다.

### 개발 과정에서 가장 기억에 남는 순간들

**1. iframe 프리즈 문제 해결**
`pointer-events: none`이라는 간단한 CSS 속성 하나로 해결되는 문제였지만, 그걸 찾기까지의 과정이 정말 값진 경험이었습니다. 몇 시간 동안 JavaScript 이벤트 처리 방법을 바꿔보고, z-index를 조정하고, 심지어 iframe을 다시 구현하려고까지 했는데, 결국 답은 CSS에 있었습니다.

**2. 드래그 앤 드롭 구현의 즐거움**
"북마크를 어떻게 쉽게 등록할까?" 고민하다가 드래그 앤 드롭을 떠올렸을 때의 짜릿함! HTML5 API만으로 이렇게 직관적인 UX를 만들 수 있다는 게 신기했습니다. 복잡한 프레임워크 없이도 충분히 모던한 인터페이스를 만들 수 있다는 자신감을 얻었습니다.

**3. Netscape Bookmark Format 발견**
"브라우저에 어떻게 자동으로 북마크를 추가하지?" 검색하다가 1990년대 포맷을 발견하고 놀랐습니다. 30년 전 표준이 2025년에도 모든 브라우저에서 작동한다는 사실이 **웹 표준의 지속성**을 보여주는 완벽한 예시였습니다.

**4. UI 재설계 결정**
카드 UI가 "예쁘긴 한데 실용적이지 않다"는 피드백을 받고 테이블로 전환했을 때, 처음엔 "디자인이 너무 평범해지는 거 아닌가?" 걱정했습니다. 하지만 완성하고 보니 **기능이 곧 디자인**이라는 걸 깨달았습니다. 사용자가 원하는 건 화려한 UI가 아니라 빠르고 명확한 UI였습니다.

### 핵심 교훈

이 프로젝트를 통해 **"사용자가 쉽게 사용할 수 있는 도구"**를 만드는 것이 **"기술적으로 완벽한 도구"**를 만드는 것보다 훨씬 중요하다는 걸 깨달았습니다.

- 자동 감지가 100% 작동하지 않아도 → 시각적 선택으로 대안 제공
- 코드를 복사/붙여넣기하기 번거로워도 → 드래그 앤 드롭으로 간소화
- 화려한 카드 디자인이 공간을 많이 차지해도 → 실용적인 테이블로 전환

**완벽한 기술보다 사용자 친화적인 경험이 더 중요합니다.**

---

## 🔗 링크

- [GitHub Repository](https://github.com/ej-rarus/login-bookmarklet-generator)
- [Tutorial](tutorial.html)

## 📝 기술 블로그 시리즈

이 프로젝트에서 배운 내용을 시리즈로 정리할 예정입니다:

1. **로그인 북마크렛 생성기 개발기** ← 현재 글
2. **iframe 다루기: pointer-events와 이벤트 위임**
   - iframe 내부 스타일 동적 주입
   - pointer-events로 클릭 이벤트 제어
   - 이벤트 위임 패턴의 장점
3. **순수 JavaScript로 파일 업로드 구현하기**
   - FileReader API 활용
   - Blob과 URL.createObjectURL()
   - Clipboard API로 복사 기능 구현
4. **드래그 앤 드롭과 브라우저 북마크 포맷**
   - HTML5 Drag and Drop API
   - Netscape Bookmark Format 파싱
   - 크로스 브라우저 호환성

---

**Made with ❤️ by [EJ Lee](https://leeeunjae.com)**

> 이 프로젝트가 도움이 되셨나요? ⭐️ Star를 눌러주세요!
