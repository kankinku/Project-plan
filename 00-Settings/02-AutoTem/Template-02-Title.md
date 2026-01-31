<%*
// 1. Google Gemini API 키 설정
const GEMINI_API_KEY = "AIzaSyAnP9D9bQIyVidsqWwnvyC1rt46xwJuKqQ"; // 사용자님의 키가 입력되었습니다.

// 2. 제목 생성을 위한 프롬프트 정의
const title_prompt = `You are an expert at creating precise, information-dense titles for technical and business documents. Your task is to analyze the given Obsidian markdown note and generate a highly specific title that captures the essence of the content.

### Analysis Framework:
1. Document Classification (구현/연구/분석/제안/설계/운영)
2. Core Components (Technology, Approach, Domain)
3. Title Structure: Domain Technology Implementation Context
   (Example: LLM GPT4 PDF 문서처리 파이프라인 RAG 시스템 구축)

[Output Requirements]
- ONLY output the final title (No markdown, No quotes, No special characters)
- NEVER use: \\/:*?"<>|[]
- Natural spaces between words, Max 60 characters.
- Language: Technical terms in English, Context in Korean.
`;

// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// Gemini API를 호출하여 제목을 생성합니다
async function generateTitle(content) {
    // 사용할 모델 설정 (gemini-1.5-flash가 빠르고 무료 할당량이 넉넉합니다)
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
                        text: `${title_prompt}\n\nHere is the content of the note:\n${content}`
                    }]
                }]
            })
        });
        
        // Gemini API의 응답 구조에서 텍스트 추출
        return response.json.candidates[0].content.parts[0].text.trim();
    } catch (error) {
        console.error('Gemini 제목 생성 중 오류 발생:', error);
        return '제목 생성 실패';
    }
}

// 실행부
const title = await generateTitle(fileContent);

// 파일명으로 사용할 수 없는 특수문자 최종 제거
const sanitizedTitle = title.replace(/[\\/:*?"<>|\[\]\r\n]/g, '').trim();

if (sanitizedTitle && sanitizedTitle !== '제목 생성 실패') {
    await tp.file.rename(sanitizedTitle);
    new Notice(`파일명이 변경되었습니다: ${sanitizedTitle}`);
} else {
    new Notice("제목 생성에 실패하여 파일명을 변경하지 못했습니다.");
}
_%>