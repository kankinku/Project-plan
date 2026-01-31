이 프로젝트는 옵시디언(Obsidian)의 **Templater** 플러그인을 활용하여 **Google Gemini API**와 통신하고, 노트의 제목 생성, 태그 분류, 주제 요약 등 지식 관리 시스템을 자동화하는 스크립트 모음입니다.

## 📌 주요 기능

이 툴킷은 총 5가지의 핵심 자동화 템플릿을 포함하고 있습니다.

1. **자동 제목 생성 (`Auto-Title`)**: 노트 본문을 분석하여 기술 문서 표준(Domain-Tech-Implementation)에 맞는 최적화된 파일명으로 변경합니다.
    
2. **지능형 토픽 분류 (`Auto-Topic`)**: 사전에 정의된 트리 구조의 주제 리스트 중 노트와 가장 관련 있는 항목을 골라 `topics` 속성에 주입합니다.
    
3. **인덱스 자동 태깅 (`Auto-Index`)**: 노트의 성격(스터디, 강의, 프로젝트 등)을 분석하여 `index` 속성을 업데이트합니다.
    
4. **SEO 최적화 태그 생성 (`Auto-Tag`)**: 본문 내 키워드를 추출하여 1~10개의 마크다운 태그(#Tag)를 생성하고 `tags` 속성에 삽입합니다.
    
5. **인터랙티브 AI 에이전트 (`AI-Summarizer`)**: 모델 선택(Pro/Flash) 및 출력 방식(Callout/Markdown)을 선택하고, 커스텀 프롬프트를 통해 요약이나 액션 아이템을 추출합니다.
    

---

## 🛠️ 사전 준비 (Setup)

### 1. Gemini API 키 발급

- **[Google AI Studio](https://aistudio.google.com/)**에 접속하여 API 키를 발급받습니다.
    
- **Gemini 1.5 Flash** 모델 사용 시 무료 티어(일일 1,500회 호출 등) 내에서 비용 발생 없이 사용 가능합니다.
    

### 2. 옵시디언 플러그인 설정

- **Templater** 커뮤니티 플러그인을 설치합니다.
    
- **설정(Settings) -> Templater**:
    
    - `Template folder location` 지정 (스크립트가 저장된 폴더).
        
    - `Enable User System Action Shell`을 **ON**으로 설정 (외부 통신 및 스크립트 실행 허용).
        

---

## 📂 포함된 템플릿 목록

|파일명|기능 설명|주요 활용 모델|
|---|---|---|
|`Gemini-Auto-Title.md`|본문 분석 기반 파일명 변경|Gemini 1.5 Flash|
|`Gemini-Auto-Topic.md`|사전 정의된 카테고리 분류 (Topics)|Gemini 1.5 Flash|
|`Gemini-Auto-Index.md`|문서 성격 기반 인덱싱 (Index)|Gemini 1.5 Flash|
|`Gemini-Auto-Tag.md`|SEO 키워드 기반 태그 생성 (Tags)|Gemini 1.5 Flash|
|`Gemini-Interactive-AI.md`|모델/옵션 선택형 커스텀 응답 생성|Gemini 1.5 Pro / Flash|

Sheets로 내보내기

---

## 📝 사용 방법

1. 분석하거나 정리하고 싶은 노트를 엽니다.
    
2. `Alt + E` (Templater 실행 단축키)를 누릅니다.
    
3. 원하는 기능(예: `Gemini-Auto-Tag`)을 선택합니다.
    
4. 잠시 기다리면 알림(`Notice`)과 함께 노트 상단의 **Properties(YAML)**나 본문에 결과가 반영됩니다.
    

---

## ⚠️ 주의사항 및 문제 해결 (Troubleshooting)

- **API 호출 에러**: API 키가 정확한지, 인터넷 연결이 안정적인지 확인하세요.
    
- **파일명 변경 실패**: 윈도우 파일 시스템에서 허용되지 않는 특수문자(`\ / : * ? " < > |`)는 스크립트 내에서 자동으로 제거(Sanitize)되도록 설계되었습니다.
    
- **YAML 구조**: 본문 최상단에 `---`로 시작하는 Frontmatter 영역이 있어야 속성 업데이트가 원활하게 이루어집니다.
    

---

## 🛠️ 개발 및 커스터마이징

- **프롬프트 수정**: 각 파일 상단의 `const prompt = ...` 영역을 수정하여 AI의 페르소나나 출력 형식을 바꿀 수 있습니다.
    
- **모델 변경**: 속도를 중시하면 `gemini-1.5-flash`, 정밀한 분석을 원하면 `gemini-1.5-pro`로 스크립트 내 모델 명칭을 변경하세요.