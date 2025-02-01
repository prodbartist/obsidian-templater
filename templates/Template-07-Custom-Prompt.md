<%_*
// OpenAI API 키 설정
const OPENAI_API_KEY="OPENAI_API_KEY 키를 입력하세요"

const MODEL_NAME = await tp.system.suggester(
    ["gpt-4o", "gpt-4o-mini", "o3-mini"], 
    ["gpt-4o", "gpt-4o-mini", "o3-mini"], 
    true, 
    "모델을 선택하세요.");

const user_output_option = await tp.system.suggester(
    ["① Callout", "② Markdown"], 
    ["① Callout", "② Markdown"], 
    true, 
    "출력 옵션을 선택하세요.");

const USE_CALLOUT = user_output_option === "① Callout";
_%>

<%_*
// 요약 생성을 위한 프롬프트 정의
const custom_prompt = await tp.system.prompt("프롬프트를 입력하세요");
_%>

<%_*
// 현재 노트의 내용을 가져옵니다
const fileContent = await tp.file.content;

// OpenAI API를 호출하여 요약을 생성하는 함수
async function generateResponse(content) {
  try {
    // OpenAI API 호출
    const response = await tp.obsidian.requestUrl({
      method: "POST",
      url: "https://api.openai.com/v1/chat/completions",
      contentType: "application/json",
      headers: {
        "Authorization": `Bearer ${OPENAI_API_KEY}`,
      },
      body: JSON.stringify({
        model: MODEL_NAME,
        messages: [
          { "role": "system", "content": custom_prompt },
          { "role": "user", "content": `Here is the content of the note:\n${content}` }
        ]
      })
    });
    
    const responseData = JSON.parse(response.text);
    return responseData.choices[0].message.content;
  } catch (error) {
    console.error('API 호출 중 오류 발생:', error);
    return null;
  }
}

// frontmatter와 내용을 분리하는 함수
function separateFrontmatterAndContent(content) {
  const match = content.match(/^---\n(.*?)\n---\n([\s\S]*)/s);
  if (!match) {
    return { frontmatter: '', content: content };
  }
  return {
    frontmatter: match[1],
    content: match[2]
  };
}

// 파일 내용을 업데이트하는 함수
async function updateFileContent(file, response) {
  try {
    // 현재 파일 내용 가져오기
    const currentContent = await tp.file.content;
    const { frontmatter, content } = separateFrontmatterAndContent(currentContent);
    
    let newContent;
    if (USE_CALLOUT) {
      // 요약 callout 생성 (각 줄 앞에 '>' 추가)
      const summaryCallout = `> [!${MODEL_NAME}]\n${response.split('\n').map(line => `> ${line}`).join('\n')}\n\n`;
      
      if (frontmatter) {
        newContent = `---\n${frontmatter}\n---\n\n${summaryCallout}${content.trimStart()}`;
      } else {
        newContent = `${summaryCallout}${content.trimStart()}`;
      }
    } else {
      // 마크다운 응답 생성
      const markdownResponse = `\n----\n## ✅ ${MODEL_NAME} Response\n\n${response}\n\n----\n`;
      
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

try {
  // API 호출
  const response = await generateResponse(fileContent);
  if (!response) {
    throw new Error('API 호출 실패');
  }
  
  // 파일 업데이트
  await updateFileContent(file, response);
} catch (error) {
  console.error('메인 실행 중 오류 발생:', error);
}
_%>