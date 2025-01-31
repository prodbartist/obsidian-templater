<%_*
// OpenAI API 키 설정
const OPENAI_API_KEY="이곳에 OpenAI API 키를 입력하세요"_%>

<%_*
// 요약 생성을 위한 프롬프트 정의
const summary_prompt = `You are an expert at creating dense, informative summaries of technical and business content. Your task is to analyze the given Obsidian markdown note and create a highly structured summary.

### Analysis Steps:
1. Content Classification:
   - Document Type (기술문서/연구/회의록/분석보고서/제안서 등)
   - Primary Domain (AI/개발/비즈니스/운영 등)
   - Content Level (개념/구현/분석/전략 등)

2. Core Information Extraction:
   - Main Theme & Purpose
   - Key Technical Concepts
   - Critical Findings/Decisions
   - Implementation Details
   - Business Impact/Value

3. Entity Recognition:
   - Technical Terms (기술용어/프레임워크/도구)
   - Organizations (회사명/팀/부서)
   - Time References (날짜/기간/마일스톤)
   - People (담당자/이해관계자)
   - Resources (예산/인력/시스템)

### Summary Generation Guidelines:
Format:
- First bullet: 문서의 핵심 주제와 목적을 한 문장으로 압축
- Second bullet: 주요 기술적 내용과 의사결정 사항
- Third bullet: 구현 방법론이나 해결 방안
- Fourth bullet: 비즈니스 임팩트나 실행 계획
- Last bullet: 후속 조치나 주요 고려사항

Writing Style:
- Use technical and business terminology precisely
- End sentences with 명사/~임/~함 style
- Keep each bullet under 100 characters
- Include quantitative metrics when available
- Maintain formal business writing tone

Formatting Rules:
- Start each bullet with '-'
- No markdown or special characters
- No nested bullets
- No explanatory notes

[Output Requirements]
- ONLY include the final bullet-point summary in Korean
- NO introduction, explanation, or metadata
- NO reference to the analysis process
- Focus on density and precision of information

Remember: Your summary should serve as a comprehensive yet concise reference for both technical and business stakeholders.
`
_%>

<%_*
// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// OpenAI API를 호출하여 요약을 생성하는 함수
async function generateSummary(content) {
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
        model: "gpt-4o",
        messages: [
          { "role": "system", "content": summary_prompt },
          { "role": "user", "content": `Here is the content of the note:\n${content}` }
        ]
      })
    });
    
    return response.json.choices[0].message.content;
  } catch (error) {
    console.error('요약 생성 중 오류 발생:', error);
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
async function updateFileContent(file, summary) {
  try {
    // 현재 파일 내용 가져오기
    const currentContent = await tp.file.content;
    const { frontmatter, content } = separateFrontmatterAndContent(currentContent);
    
    // 요약 callout 생성 (각 줄 앞에 '>' 추가)
    const summaryCallout = `> [!Summary]\n${summary.split('\n').map(line => `> ${line}`).join('\n')}\n\n`;
    
    // frontmatter 유무에 따라 새로운 컨텐츠 조합
    let newContent;
    if (frontmatter) {
      newContent = `---\n${frontmatter}\n---\n\n${summaryCallout}${content.trimStart()}`;
    } else {
      newContent = `${summaryCallout}${content.trimStart()}`;
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
  // 요약 생성
  const summary = await generateSummary(fileContent);
  if (!summary) {
    throw new Error('요약 생성 실패');
  }
  
  // 파일 업데이트
  await updateFileContent(file, summary);
} catch (error) {
  console.error('메인 실행 중 오류 발생:', error);
}
_%>