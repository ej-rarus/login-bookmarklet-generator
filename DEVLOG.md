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

이렇게 단계적으로 개선해나가는 과정이 더 효과적이었습니다.

## 🎯 개선하고 싶은 점

### 향후 계획
- [ ] 북마크 내보내기/가져오기 기능
- [ ] 브라우저 확장 프로그램 버전
- [ ] 더 다양한 로그인 폼 패턴 지원
- [ ] 다국어 지원 (영어, 일본어 등)
- [ ] 북마크렛 테스트 기능 (페이지 내에서 미리 테스트)

## 📊 성과

### 주요 지표
- 순수 JavaScript로 구현 (프레임워크 없음)
- 파일 크기: 약 40KB (가볍고 빠름)
- 서버 비용: $0 (정적 호스팅만 필요)
- 지원 브라우저: Chrome, Firefox, Edge, Safari

### 사용자 경험
- HTML 파일 준비: 5초 (Ctrl+S)
- 북마크렛 생성: 10초
- 로그인 실행: 1초 (클릭 한 번)

**기존 방식 대비 90% 이상 시간 절약** 🚀

## 🎉 마무리

처음엔 "간단한 도구"로 시작했지만, 실제로 만들어보니 예상보다 고려해야 할 것들이 많았습니다. 특히 **코딩을 모르는 사용자도 쉽게 사용할 수 있게** 만드는 과정에서 많은 것을 배웠습니다.

가장 뿌듯했던 순간은 iframe 프리즈 문제를 해결했을 때였습니다. `pointer-events`라는 간단한 CSS 속성 하나로 해결되는 문제였지만, 그걸 찾기까지의 과정이 정말 값진 경험이었습니다.

이 프로젝트를 통해 **"사용자가 쉽게 사용할 수 있는 도구"**를 만드는 것이 **"기술적으로 완벽한 도구"**를 만드는 것보다 훨씬 중요하다는 걸 깨달았습니다.

---

## 🔗 링크

- [GitHub Repository](#)
- [Live Demo](#)
- [Tutorial](tutorial.html)

## 📝 기술 블로그 시리즈

1. [로그인 북마크렛 생성기 개발기] ← 현재 글
2. iframe 다루기: pointer-events와 이벤트 위임 (예정)
3. 순수 JavaScript로 파일 업로드 구현하기 (예정)

---

**Made with ❤️ by [EJ Lee](https://leeeunjae.com)**

> 이 프로젝트가 도움이 되셨나요? ⭐️ Star를 눌러주세요!
