<%_*
// OpenAI API 키 설정
const OPENAI_API_KEY="이곳에 OpenAI API 키를 입력하세요"_%>

<%_*
// 프롬프트
const index_prompt = `You are an expert in generating an appropriate index properties that will be used in Obsidian Note. Your mission is to generate one or two indexes suitable for given content.
Your generated output must be comma-separated values.

Here is the example output:

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

####

[Note] 
- Write your Final Answer in Korean. 
- DO NOT narrate, just write the output without any markdown formatting of wrapping.
- Generated indexes must be related to the content. Otherwise, you will be penalized.
- Generated indexes must be one of the <Index List>
`
_%>
 
<%_*
// frontmatter의 index 속성을 업데이트하는 함수
// @param file: 대상 파일 객체
// @param newIndex: 새로운 인덱스 문자열 (쉼표로 구분된 값)
const processIndex = async (file, newIndex) => {
  await tp.app.fileManager.processFrontMatter(file, (frontmatter) => {
    // 쉼표로 구분된 인덱스를 배열로 변환하고 각 항목의 앞뒤 공백 제거
    const index = newIndex.split(',').map(index => index.trim());
    // frontmatter의 index 속성 업데이트
    frontmatter.index = index.map(index => `${index}`);
  });
};
_%>

<%_*
// OpenAI API를 호출하여 인덱스를 생성하는 함수
async function generateIndex(content) {
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
                    { "role": "system", "content": index_prompt },
                    { "role": "user", "content": `Here is the content of the note:\n${content}` }
                ]
            })
        });
        
        return response.json.choices[0].message.content;
    } catch (error) {
        console.error('인덱스 생성 중 오류 발생:', error);
        return '인덱스 생성 실패';
    }
}

// 현재 노트의 내용을 가져옵니다
const fileContent = tp.file.content;

// 인덱스를 생성하고 파일에 적용
const index = await generateIndex(fileContent);
const file = tp.config.target_file;
await processIndex(file, index);
_%>