<%_*
// OpenAI API 키 설정
const OPENAI_API_KEY="이곳에 OpenAI API 키를 입력하세요"_%>

<%_*
// 제목 생성을 위한 프롬프트 정의
const title_prompt = `You are an expert at creating precise, information-dense titles for technical and business documents. Your task is to analyze the given Obsidian markdown note and generate a highly specific title that captures the essence of the content.

### Analysis Framework:

1. Document Classification:
   - Content Type (구현/연구/분석/제안/설계/운영)
   - Technical Depth (개념/응용/심화/실무)
   - Scope (PoC/프로토타입/프로덕션/연구)
   - Target Audience (개발자/아키텍트/비즈니스/운영)

2. Core Components Analysis:
   - Primary Technology/Framework
   - Implementation Approach
   - Business Context
   - Problem Domain
   - Solution Architecture

3. Title Structure Rules:
   Format: Domain Technology Implementation Context
   Example: LLM GPT4 PDF 문서처리 파이프라인 RAG 시스템 구축

### Title Generation Guidelines:

Technical Elements:
- Keep technical terms in English (RAG, LLM, API etc)
- Include version numbers if relevant
- Specify frameworks/tools used
- Include architectural patterns

Formatting Requirements:
- Use spaces as natural word delimiters
- NO special characters (\/:*?"<>|)
- Keep total length under 60 characters
- NO brackets or special formatting
- Natural reading flow priority

Language Style:
- End with 구현/설계/분석/연구/검토
- Technical terms in English, context in Korean
- Be specific about implementation details
- Include quantitative metrics when relevant

[Output Requirements]
- ONLY output the final title
- NO explanations or alternatives
- NO markdown formatting
- Must follow the exact format specified
- NEVER use following characters: \/:*?"<>|[]
- Use natural spaces between words

Remember: 
- Title should serve as a precise index for future reference and search while maintaining natural readability.
- DO NOT wrap the title with double quotes. Just output the title.
- DO NOT include any special characters such as "\/:*?"<>|[]".
`
_%>

<%_*
// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// OpenAI API를 호출하여 제목을 생성합니다
async function generateTitle(content) {
    try {
        const response = await tp.obsidian.requestUrl({
            method: "POST",
            url: "https://api.openai.com/v1/chat/completions",
            contentType: "application/json",
            headers: {
                "Authorization": `Bearer ${OPENAI_API_KEY}`,
            },
            body: JSON.stringify({
                model: "gpt-4o",
                messages: [
                    { "role": "system", "content": title_prompt },
                    { "role": "user", "content": `Here is the content of the note:\n${content}` }
                ]
            })
        });
        
        return response.json.choices[0].message.content;
    } catch (error) {
        console.error('제목 생성 중 오류 발생:', error);
        return '제목 생성 실패';
    }
}

// 제목을 생성하고 파일 이름을 변경합니다
const title = await generateTitle(fileContent);
const sanitizedTitle = title.replace(/[\\/:*?"<>|\[\]]/g, '');
await tp.file.rename(sanitizedTitle);
_%>