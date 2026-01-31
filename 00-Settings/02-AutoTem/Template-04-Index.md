<%*
// 1. Google Gemini API 키 설정
const GEMINI_API_KEY = "AIzaSyAnP9D9bQIyVidsqWwnvyC1rt46xwJuKqQ";

// 2. 프롬프트 정의 - 인덱스 생성을 위한 프롬프트
const index_prompt = `You are an expert in generating an appropriate index properties that will be used in Obsidian Note. Your mission is to generate one or two indexes suitable for given content.
Your generated output must be comma-separated values.

### Example Output:
[[🏷️ 강의]], [[🏷️ 패스트캠퍼스-주주총회]]

Here's a list of possible substitutions for index. You must use one of these indexes listed below.

<Index List>
- [[🏷️ 스터디]] : Self studying contents. Mostly development self memo will be this index.
- [[🏷️ 강의]] : Contents used in lectures
- [[🏷️ 외주 프로젝트]] : Enterprise Outsourcing Projects
- [[🏷️ 컨설팅]] : Consulting related contents
- [[🏷️ 사이드 프로젝트]] : Contents related to Side Projects
- [[🏷️ PM]] : Project Management
- [[🏷️ YouTube테디노트]] : Contents related to YouTube 테디노트. 테디노트 YouTube channel treats about AI / RAG / Agent related contents.
- [[🏷️ 패스트캠퍼스-주주총회]] : Fastcampus is a platform for learning AI / RAG / Agent, etc. 주주총회 is a monthly Live seminar session for students.
- [[🏷️ 커리큘럼]] : Curriculum related contents
- [[🏷️ 컨퍼런스]] : Conference related contents
- [[🏷️ 회사운영]] : Company operation related contents
- [[🏷️ 데일리 노트]] : Daily note related contents
</Index List>

[Note] 
- Write your Final Answer in Korean. 
- DO NOT narrate, just write the output without any markdown formatting.
- Generated indexes must be one of the <Index List>.
`;
_%>

<%*
// frontmatter의 index 속성을 업데이트하는 함수
const processIndex = async (file, newIndex) => {
  await tp.app.fileManager.processFrontMatter(file, (frontmatter) => {
    // 쉼표로 구분된 인덱스를 배열로 변환하고 각 항목의 앞뒤 공백 및 마크다운 기호 제거
    const indexArray = newIndex.split(',').map(index => index.trim().replace(/`/g, ''));
    frontmatter.index = indexArray;
  });
};
_%>

<%*
// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// Gemini API를 호출하여 인덱스를 생성하는 함수
async function generateIndex(content) {
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
                        text: `${index_prompt}\n\nHere is the content of the note:\n${content}`
                    }]
                }]
            })
        });
        
        // Gemini API 응답에서 텍스트 추출
        return response.json.candidates[0].content.parts[0].text.trim();
    } catch (error) {
        console.error('인덱스 생성 중 오류 발생:', error);
        return '인덱스 생성 실패';
    }
}

// 실행부: 인덱스 생성 및 파일 적용
const index = await generateIndex(fileContent);

if (index && index !== '인덱스 생성 실패') {
    const file = tp.config.target_file;
    await processIndex(file, index);
    new Notice("인덱스가 자동으로 분류되어 적용되었습니다.");
} else {
    new Notice("인덱스 생성에 실패했습니다.");
}
_%>