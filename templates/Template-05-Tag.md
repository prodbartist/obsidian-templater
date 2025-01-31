<%_*
// OpenAI API 키 설정
const OPENAI_API_KEY="이곳에 OpenAI API 키를 입력하세요"_%>

<%_*
// 태그 생성을 위한 프롬프트 정의
const tag_prompt = `You are an expert at generating precise, SEO-optimized tags for technical and business documentation. Your task is to analyze the given content and create highly searchable tags that capture core concepts and technical elements.

### Analysis Framework:

1. Technical Classification:
  - Primary Technologies (프레임워크/라이브러리/도구)
  - Technical Concepts (알고리즘/아키텍처/패턴)
  - Implementation Details (구현방식/메서드/기술스택)
  - Domain Specifics (분야/산업/적용영역)

2. Contextual Analysis:
  - Business Value (ROI/성과/효율)
  - Use Cases (활용사례/적용방안)
  - Problem Domains (해결과제/니즈)
  - Target Users (사용자/클라이언트)

3. Tag Selection Criteria:
  Must Meet ALL Conditions:
  - Explicitly appears in content
  - Highly specific (no general terms)
  - Technical precision
  - Search optimization value
  - Clear relation to core topic

### Tag Generation Rules:

Technical Requirements:
- Keep technical terms in English (#RAG, #LLM etc)
- Include version numbers when specific
- Use established technical terminology
- Maintain technical accuracy

Tag Structure:
- 1-10 tags only
- NO spaces within tags
- Korean preferred except for technical terms
- Compound words allowed for precision
- Follow established tech community conventions

Exclusion Rules:
- NO general/vague terms
- NO marketing buzzwords
- NO compound tags mixing Korean/English
- NO tags not present in content
- NO subjective descriptors

Quality Checks:
- Verify each tag appears in content
- Ensure searchability
- Check technical accuracy
- Validate specificity
- Confirm relevance

[Output Requirements]
- ONLY comma-separated tags
- Each tag starts with #
- NO explanations or alternatives
- Single line output
- NO markdown formatting

Example format:
#RAG, #문서처리, #파이프라인, #LangChain, #성능최적화

Remember: 
- Tags must be precise, present in content, and highly searchable while maintaining technical accuracy.
- Generate maximum 10 tags.
`
_%>

<%_*
// frontmatter의 tags 속성을 업데이트하는 함수
// @param file: 대상 파일 객체
// @param newTags: 새로운 태그 문자열 (쉼표로 구분된 값)
const processTags = async (file, newTags) => {
  await tp.app.fileManager.processFrontMatter(file, (frontmatter) => {
    // 쉼표로 구분된 태그를 배열로 변환하고 각 항목의 앞뒤 공백 제거
    const tags = newTags.split(',').map(tag => tag.trim());
    // frontmatter의 tags 속성 업데이트
    frontmatter.tags = tags.map(tag => `${tag}`);
  });
};
_%>

<%_*
// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// OpenAI API를 호출하여 태그를 생성하는 함수
async function generateTags(content) {
    try {
        const response = await tp.obsidian.requestUrl({
            method: "POST",
            url: "https://api.openai.com/v1/chat/completions",
            contentType: "application/json",
            headers: {
                "Authorization": `Bearer ${OPENAI_API_KEY}`,
            },
            body: JSON.stringify({
                model: "gpt-4",
                messages: [
                    { "role": "system", "content": tag_prompt },
                    { "role": "user", "content": `Here is the content of the note:\n${content}` }
                ]
            })
        });
        
        return response.json.choices[0].message.content;
    } catch (error) {
        console.error('태그 생성 중 오류 발생:', error);
        return '태그 생성 실패';
    }
}

// 태그를 생성하고 파일에 적용
const tags = await generateTags(fileContent);
const file = tp.config.target_file;
await processTags(file, tags);
_%>