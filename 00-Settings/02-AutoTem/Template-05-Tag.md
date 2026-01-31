<%*
// 1. Google Gemini API 키 설정
const GEMINI_API_KEY = "AIzaSyAnP9D9bQIyVidsqWwnvyC1rt46xwJuKqQ";

// 2. 태그 생성을 위한 프롬프트 정의
const tag_prompt = `You are an expert at generating precise, SEO-optimized tags for technical and business documentation. Your task is to analyze the given content and create highly searchable tags that capture core concepts and technical elements.

### Tag Generation Rules:
- 1-10 tags only
- Each tag starts with #
- Technical terms in English, Context in Korean
- NO spaces within tags
- Korean preferred except for technical terms

[Output Requirements]
- ONLY comma-separated tags
- NO explanations or alternatives
- Single line output
- NO markdown formatting
- Example format: #RAG, #문서처리, #파이프라인, #LangChain
`;
_%>

<%*
// frontmatter의 tags 속성을 업데이트하는 함수
const processTags = async (file, newTags) => {
  await tp.app.fileManager.processFrontMatter(file, (frontmatter) => {
    // 쉼표로 구분된 태그를 배열로 변환하고 각 항목의 앞뒤 공백 및 불필요한 마크다운 기호 제거
    const tagsArray = newTags.split(',').map(tag => tag.trim().replace(/`/g, ''));
    // frontmatter의 tags 속성 업데이트
    frontmatter.tags = tagsArray;
  });
};
_%>

<%*
// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// Gemini API를 호출하여 태그를 생성하는 함수
async function generateTags(content) {
    // 모델 설정 (gemini-1.5-flash)
    const model = "gemini-1.5-flash";
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${GEMINI_API_KEY}`;

    try {
        const response = await tp.obsidian.requestUrl({
            method: "POST",
            url: url,
            contentType: "application/json",
            body: JSON.stringify({
                contents: [{
                    parts: [{
                        text: `${tag_prompt}\n\nHere is the content of the note:\n${content}`
                    }]
                }]
            })
        });
        
        // Gemini API 응답에서 텍스트 추출
        return response.json.candidates[0].content.parts[0].text.trim();
    } catch (error) {
        console.error('태그 생성 중 오류 발생:', error);
        return '태그 생성 실패';
    }
}

// 실행부: 태그 생성 및 파일 적용
const tags = await generateTags(fileContent);

if (tags && tags !== '태그 생성 실패') {
    const file = tp.config.target_file;
    await processTags(file, tags);
    new Notice("태그가 생성되어 적용되었습니다.");
} else {
    new Notice("태그 생성에 실패했습니다.");
}
_%>