<%*
// 1. Google Gemini API 키 설정
const GEMINI_API_KEY = "AIzaSyAnP9D9bQIyVidsqWwnvyC1rt46xwJuKqQ";

// 2. 요약 생성을 위한 프롬프트 정의
const summary_prompt = `You are an expert at creating dense, informative summaries of technical and business content. Your task is to analyze the given Obsidian markdown note and create a highly structured summary.

### Summary Generation Guidelines:
- First bullet: 문서의 핵심 주제와 목적을 한 문장으로 압축
- Second bullet: 주요 기술적 내용과 의사결정 사항
- Third bullet: 구현 방법론이나 해결 방안
- Fourth bullet: 비즈니스 임팩트나 실행 계획
- Last bullet: 후속 조치나 주요 고려사항

Writing Style:
- Use technical and business terminology precisely
- End sentences with 명사/~임/~함 style
- Start each bullet with '-'
- ONLY include the final bullet-point summary in Korean
`;
_%>

<%*
// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// Gemini API를 호출하여 요약을 생성하는 함수
async function generateSummary(content) {
  const model = "gemini-1.5-flash"; // 속도와 효율성을 위해 Flash 모델 사용
  const url = `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${GEMINI_API_KEY}`;

  try {
    const response = await tp.obsidian.requestUrl({
      method: "POST",
      url: url,
      contentType: "application/json",
      body: JSON.stringify({
        contents: [{
          parts: [{
            text: `${summary_prompt}\n\nHere is the content of the note:\n${content}`
          }]
        }]
      })
    });
    
    // Gemini API 응답에서 텍스트 추출
    return response.json.candidates[0].content.parts[0].text.trim();
  } catch (error) {
    console.error('요약 생성 중 오류 발생:', error);
    return null;
  }
}

// frontmatter와 내용을 분리하는 함수
function separateFrontmatterAndContent(content) {
  const match = content.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)/);
  if (!match) {
    return { frontmatter: '', content: content };
  }
  return {
    frontmatter: match[1],
    content: match[2]
  };
}

// 파일 내용을 업데이트하는 함수
async function updateFileContent(file, summary) {
  try {
    const currentContent = await tp.file.content;
    const { frontmatter, content } = separateFrontmatterAndContent(currentContent);
    
    // 요약 callout 생성 (각 줄 앞에 '>' 추가)
    const summaryCallout = `> [!Summary]\n${summary.split('\n').map(line => `> ${line}`).join('\n')}\n\n`;
    
    let newContent;
    if (frontmatter) {
      newContent = `---\n${frontmatter}\n---\n\n${summaryCallout}${content.trimStart()}`;
    } else {
      newContent = `${summaryCallout}${content.trimStart()}`;
    }
    
    await app.vault.modify(file, newContent);
    return true;
  } catch (error) {
    console.error('파일 업데이트 중 오류 발생:', error);
    throw error;
  }
}

// 메인 실행 로직
const file = tp.config.target_file;

try {
  const summary = await generateSummary(fileContent);
  if (!summary) {
    throw new Error('요약 생성 실패');
  }
  
  await updateFileContent(file, summary);
  new Notice("노트 요약이 생성되어 상단에 삽입되었습니다.");
} catch (error) {
  console.error('메인 실행 중 오류 발생:', error);
  new Notice("요약 생성 중 오류가 발생했습니다.");
}
_%>