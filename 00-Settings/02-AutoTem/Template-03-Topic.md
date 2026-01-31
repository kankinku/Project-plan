<%*
// 1. Google Gemini API 키 설정
const GEMINI_API_KEY = "AIzaSyAnP9D9bQIyVidsqWwnvyC1rt46xwJuKqQ";

// 2. 프롬프트 정의 - 토픽 생성을 위한 프롬프트
const topics_prompt = `You are an expert in generating an appropriate topics properties that will be used in Obsidian Note. Your mission is to generate one or more topics suitable for given content.
Your generated output must be comma-separated values.

### Example Output:
[[📚 214 Document Parser]], [[📖 200 AI & Data]], [[📚 601 Enterprise Outsourcing Projects]]

### Topics List:
(이하 생략 - 사용자님이 제공하신 토픽 리스트가 이 자리에 들어갑니다)

[Note] Write your Final Answer in Korean. DO NOT narrate, just write the output without any markdown formatting.
`;
_%>

<%*
// frontmatter의 topics 속성을 업데이트하는 함수
const processTopics = async (file, newTopics) => {
  await tp.app.fileManager.processFrontMatter(file, (frontmatter) => {
    // 쉼표로 구분된 토픽을 배열로 변환하고 각 항목의 앞뒤 공백 및 마크다운 기호 제거 시도
    const topicsArray = newTopics.split(',').map(topic => topic.trim().replace(/`/g, ''));
    frontmatter.topics = topicsArray;
  });
};
_%>

<%*
// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// Gemini API를 호출하여 토픽을 생성하는 함수
async function generateTopics(content) {
    // 모델 설정 (gemini-1.5-flash 추천)
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
                        text: `${topics_prompt}\n\nHere is the content of the note:\n${content}`
                    }]
                }]
            })
        });
        
        // Gemini API 응답에서 텍스트 추출
        return response.json.candidates[0].content.parts[0].text.trim();
    } catch (error) {
        console.error('토픽 생성 중 오류 발생:', error);
        return '토픽 생성 실패';
    }
}

// 실행부: 토픽 생성 및 파일 적용
const topics = await generateTopics(fileContent);

if (topics && topics !== '토픽 생성 실패') {
    const file = tp.config.target_file;
    await processTopics(file, topics);
    new Notice("토픽이 자동으로 분류되어 적용되었습니다.");
} else {
    new Notice("토픽 생성에 실패했습니다.");
}
_%>