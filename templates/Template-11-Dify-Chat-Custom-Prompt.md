<%_*
// 환경 변수 설정
const HOST_URL = "https://api.dify.ai";
const DIFY_API_KEY = "DIFY_API_KEY 키를 입력하세요";
const DIFY_TYPE = "chat";
const DIFY_APP_NAME = "심플 챗봇";
const USER_INPUTS = {instruction: await tp.system.prompt("챗봇 지시사항을 작성해 주세요")};
const VERIFY_SSL = HOST_URL.startsWith('https');

// 출력 옵션 선택
const user_output_option = await tp.system.suggester(
    ["① Callout", "② Markdown"], 
    ["① Callout", "② Markdown"], 
    true, 
    "출력 옵션을 선택하세요.");

const USE_CALLOUT = user_output_option === "① Callout";

function createApiUrl() {
  switch(DIFY_TYPE) {
    case "workflow": return `${HOST_URL}/v1/workflows/run`;
    case "agent":
    case "chat": return `${HOST_URL}/v1/chat-messages`;
    case "completion": return `${HOST_URL}/v1/completion-messages`;
    default: throw new Error(`Invalid Dify type: ${DIFY_TYPE}`);
  }
}

async function generateWithDify(content) {
  try {
    const data = {
      query: content,
      inputs: USER_INPUTS,
      response_mode: "streaming",
      user: "Teddy",
      conversation_id: "",
    };

    console.log(JSON.stringify(data));
    console.log(createApiUrl());

    const response = await tp.obsidian.requestUrl({
      method: "POST",
      url: createApiUrl(),
      contentType: "application/json",
      headers: {
        "Authorization": `Bearer ${DIFY_API_KEY}`,
      },
      body: JSON.stringify(data)
    });

    // 스트리밍 응답 처리
    const lines = response.text.split('\n');
    let result = '';

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        try {
          const data = JSON.parse(line.replace('data: ', ''));
          if (data.event === 'text_chunk') {
            result += data.data.text;
          } else if (['agent_message', 'message', 'completion'].includes(data.event)) {
            if (data.answer) {
              result += data.answer;
            } else if (data.data?.text) {
              result += data.data.text;
            }
          } else if (data.event === 'workflow_finished') {
            result += data.data.outputs.output;
          }
        } catch (e) {
          console.error('스트리밍 데이터 파싱 오류:', e);
        }
      }
    }

    return result || null;

  } catch (error) {
    console.error('Dify 호출 중 오류 발생:', error);
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

async function main() {
  const file = tp.config.target_file;
  const fileContent = await tp.file.content;
  
  try {
    const response = await generateWithDify(fileContent);
    if (!response) throw new Error('Dify 호출 실패');
    console.log('Dify 호출이 성공적으로 완료되었습니다.');

    // frontmatter와 내용 분리
    const { frontmatter, content } = separateFrontmatterAndContent(fileContent);
    
    let newContent;
    if (USE_CALLOUT) {
      // Callout 형식으로 출력
      const difyCallout = `> [!${DIFY_APP_NAME}]\n${response.split('\n').map(line => `> ${line}`).join('\n')}\n\n`;
      
      if (frontmatter) {
        newContent = `---\n${frontmatter}\n---\n\n${difyCallout}${content.trimStart()}`;
      } else {
        newContent = `${difyCallout}${content.trimStart()}`;
      }
    } else {
      // Markdown 형식으로 출력
      const markdownResponse = `\n----\n## ✅ ${DIFY_APP_NAME}\n\n${response}\n\n----\n`;
      
      if (frontmatter) {
        newContent = `---\n${frontmatter}\n---\n\n${markdownResponse}${content.trimStart()}`;
      } else {
        newContent = `${markdownResponse}${content.trimStart()}`;
      }
    }

    await app.vault.modify(file, newContent);
    return true;
  } catch (error) {
    console.error('메인 실행 중 오류 발생:', error);
    return false;
  }
}

main();
_%>