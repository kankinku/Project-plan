<%*
// 1. Google Gemini API 키 설정
const GEMINI_API_KEY = "AIzaSyAnP9D9bQIyVidsqWwnvyC1rt46xwJuKqQ";

// 2. 모델 선택 (Gemini 라인업으로 변경)
const MODEL_NAME = await tp.system.suggester(
    ["Gemini 1.5 Pro (정교한 분석)", "Gemini 1.5 Flash (빠른 속도)", "Gemini 2.0 Flash Exp (최신 모델)"], 
    ["gemini-1.5-pro", "gemini-1.5-flash", "gemini-2.0-flash-exp"], 
    true, 
    "사용할 Gemini 모델을 선택하세요.");

// 3. 출력 옵션 선택
const user_output_option = await tp.system.suggester(
    ["① Callout (박스 형태)", "② Markdown (텍스트 형태)"], 
    ["① Callout", "② Markdown"], 
    true, 
    "출력 방식을 선택하세요.");

const USE_CALLOUT = user_output_option === "① Callout";

// 4. 커스텀 프롬프트 입력받기
const custom_prompt = await tp.system.prompt("Gemini에게 내릴 명령을 입력하세요 (예: 요약해줘, 번역해줘, 모순점 찾아줘)");
_%>

<%*
// 현재 노트의 내용을 가져옵니다
const fileContent = await tp.file.content;

// Gemini API를 호출하여 응답을 생성하는 함수
async function generateResponse(content) {
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${MODEL_NAME}:generateContent?key=${GEMINI_API_KEY}`;
    
    try {
        const response = await tp.obsidian.requestUrl({
            method: "POST",
            url: url,
            contentType: "application/json",
            body: JSON.stringify({
                contents: [{
                    parts: [{
                        text: `System Instruction: ${custom_prompt}\n\nNote Content:\n${content}`
                    }]
                }]
            })
        });
        
        // Gemini API 응답에서 텍스트 추출
        return response.json.candidates[0].content.parts[0].text.trim();
    } catch (error) {
        console.error('Gemini API 호출 중 오류 발생:', error);
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
async function updateFileContent(file, responseText) {
    try {
        const currentContent = await tp.file.content;
        const { frontmatter, content } = separateFrontmatterAndContent(currentContent);
        
        let newContent;
        if (USE_CALLOUT) {
            // Callout 생성
            const summaryCallout = `> [!${MODEL_NAME.toUpperCase()}]\n${responseText.split('\n').map(line => `> ${line}`).join('\n')}\n\n`;
            
            if (frontmatter) {
                newContent = `---\n${frontmatter}\n---\n\n${summaryCallout}${content.trimStart()}`;
            } else {
                newContent = `${summaryCallout}${content.trimStart()}`;
            }
        } else {
            // 마크다운 응답 생성
            const markdownResponse = `\n----\n## ✅ Gemini (${MODEL_NAME}) Response\n\n${responseText}\n\n----\n`;
            
            if (frontmatter) {
                newContent = `---\n${frontmatter}\n---\n\n${markdownResponse}${content.trimStart()}`;
            } else {
                newContent = `${markdownResponse}${content.trimStart()}`;
            }
        }
        
        // 파일 업데이트
        await app.vault.modify(file, newContent);
        return true;
    } catch (error) {
        console.error('파일 업데이트 중 오류 발생:', error);
        throw error;
    }
}

// 메인 실행 로직
const file = tp.config.target_file;

if (MODEL_NAME && custom_prompt) {
    try {
        new Notice("Gemini가 요청을 처리 중입니다...");
        const responseText = await generateResponse(fileContent);
        
        if (!responseText) {
            throw new Error('API 응답이 비어있습니다.');
        }
        
        await updateFileContent(file, responseText);
        new Notice("성공적으로 적용되었습니다!");
    } catch (error) {
        console.error('실행 중 오류 발생:', error);
        new Notice("오류가 발생했습니다. 콘솔을 확인하세요.");
    }
}
_%>