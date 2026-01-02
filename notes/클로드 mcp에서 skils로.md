---
tags: [claude, mcp, automation, workflow, ai-tools, protocol]
---

## 한 줄 요약

MCP는 "무엇(what)"을 제공하는 범용 프로토콜이고, Claude Skills는 "어떻게(how)"를 제공하는 절차적 지식으로, 상호 보완적으로 사용된다.

## 핵심 정리

- **MCP**: AI와 외부 도구를 연결하는 표준 프로토콜 (2024년 11월 출시)
- **Claude Skills**: Markdown 기반으로 워크플로우를 정의하는 방식 (2025년 10월 출시)
- **핵심 차이**: 도구/데이터 접근(MCP) vs 지침/방법론(Skills)
- **토큰 효율성**: MCP는 수만 개 토큰 소비, Skills는 메타데이터 스캔 시 ~100개 토큰만 사용
- **관계**: 경쟁이 아닌 상호 보완적 도구

## 상세 내용

### MCP (Model Context Protocol) 란?

#### 개념

AI 모델이 외부 데이터 소스와 도구를 표준화된 방식으로 사용할 수 있도록 하는 오픈소스 프로토콜입니다. 'AI계의 USB-C'로 비유되며, 클라이언트-서버 아키텍처로 구성됩니다.

#### 구성 요소

- **MCP Hosts**: AI 도구나 IDE 프로그램 (예: Claude Desktop, Cursor)
- **MCP Clients**: 특정 Server와 1:1 연결을 설정하는 호스트의 구성 요소
- **MCP Servers**: 특정 기능을 제공하도록 설계된 프로그램 단위

#### 만드는 방법

1. **서버 구축**: Python이나 TypeScript로 MCP 서버 구현
2. **프로토콜 준수**: Resources, Prompts, Tools 등의 MCP 사양 준수
3. **배포**: Python 패키지처럼 배포하거나 중앙에서 호스팅
4. **설정**: Claude Desktop이나 Cursor의 설정 파일에 MCP 서버 등록

**예시 구조:**
```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["-m", "my_mcp_server"]
    }
  }
}
```

**요구사항**: Claude Pro 계정 필요

### Claude Skills 란?

#### 개념

Claude에게 특정 작업을 더 효율적이고 일관되게 수행할 수 있도록 커스텀 능력을 부여하는 시스템입니다. 지침(instructions), 스크립트, 리소스가 포함된 폴더 형태로 제공됩니다.

#### 작동 방식

- **동적 로딩**: 필요할 때만 로드됨
- **메타데이터 스캔**: 약 100개 토큰만 사용
- **활성화 시**: 전체 스킬 콘텐츠 5k 토큰 미만으로 로드
- **구성**: YAML 메타데이터 + Markdown 지침 + 선택적 실행 스크립트

#### 만드는 방법

**1. Claude.ai에서 대화형으로 만들기**
```
사용자: "이미지 편집 Skill 만들어줘"
→ Claude가 대화를 통해 요구사항 질문
→ 자동으로 Skill 생성
```

**2. API를 통한 구현**
```python
client.beta.skills.create(
    display_title="DCF 분석",
    files=["instructions.md", "script.py"]
)
```

**3. 수동 생성**
```
my-skill/
├── skill.yaml          # 메타데이터
├── instructions.md     # 지침
└── script.py          # 실행 스크립트 (선택)
```

**4. GitHub 저장소 활용**
- anthropic/skills 저장소에서 공식 예제 확인
- 드래그 앤 드롭으로 Claude 워크스페이스에 설치

### 핵심 차이점 비교

#### 철학적 차이

| 측면        | MCP                           | Claude Skills              |
| :-------- | :---------------------------- | :------------------------- |
| **목적**    | "무엇(what)"을 제공 - 도구와 데이터 접근    | "어떻게(how)"를 제공 - 지침과 방법론  |
| **관점**    | 범용 연결기                        | 절차적 지식                     |
| **출시 시기** | 2024년 11월                     | 2025년 10월                  |

#### 구조적 차이

**MCP:**
- 전체 프로토콜 사양 (호스트, 클라이언트, 서버, 리소스, 프롬프트, 도구)
- 서버 구축 및 프로토콜 준수 필요
- 코드 기반 구현 (Python, TypeScript)

**Skills:**
- 간단한 YAML 메타데이터 + Markdown
- 선택적 실행 스크립트
- 파일 기반 구성

#### 토큰 효율성

**MCP:**
- GitHub MCP 단독으로 수만 개 토큰 소비
- 모든 도구 정의가 항상 컨텍스트에 로드됨

**Skills:**
- 메타데이터 스캔: ~100개 토큰
- 활성화 시: 5k 토큰 미만
- 필요할 때만 세부 정보 로드

#### 배포 및 사용성

**MCP:**
- 중앙 호스팅 가능
- Python 패키지처럼 배포
- 버전 관리, 릴리스, 다수 사용자 업데이트 용이
- 팀 전체 표준화에 적합

**Skills:**
- 로컬 우선 방식
- 드래그 앤 드롭으로 설치
- 커스터마이징 쉬움
- 팀 전체 업데이트 전파 어려움

## 실무 적용

### 선택 기준

**Claude Skills 우선 사용:**
- 개인/팀 내부 반복 작업 자동화
- 빠른 프로토타이핑이 필요한 실험적 기능
- 토큰 효율성이 중요한 경우
- 비개발자도 접근 가능해야 할 때

**MCP 우선 사용:**
- 외부 API와의 안정적인 연동
- 커뮤니티와 공유할 도구 개발
- 표준화가 중요한 엔터프라이즈 환경
- 다수 사용자에게 일관된 업데이트 배포 필요

### 하이브리드 전략: "유연함 + 정확성"

LLM의 유연함과 스크립트의 정확성을 결합하는 것이 핵심입니다.

1. **고정 작업 분리**: 반복되는 로직은 `bash`나 `python` 스크립트로 분리
2. **스킬 정의**: Skills의 `instructions.md`에서 해당 스크립트를 호출하도록 자연어로 지시
3. **효과**:
   - 토큰 소모 절감
   - 실행 속도 향상
   - 예외 상황 발생 시 LLM이 스스로 스크립트 수정 후 재시도 가능

### CLI 도구 활용

MCP가 달성할 수 있는 거의 모든 것을 CLI 도구로도 처리 가능합니다. LLM은 `cli-tool --help`를 호출하는 방법을 이미 알고 있어, 많은 토큰을 사용법 설명에 소비할 필요가 없습니다.

### 보안 고려사항

**Skills 사용 시:**
- 신뢰할 수 있는 출처의 Skills만 사용
- 설치 전 Skill 폴더 내 모든 파일 내용 검토
- 코드 의존성과 번들된 리소스 확인

## 관련 노트

<!-- 추후 추가 -->

## 참고 자료

**MCP 관련:**
- [Claude MCP로 엔지니어링 업무 자동화하기](https://insight.infograb.net/blog/2025/01/22/mcp/)
- [Model Context Protocol (MCP): 개발자 구현 가이드](https://www.jenova.ai/ko/resources/a-developers-guide-to-model-context-protocol-mcp)
- [Claude MCP 완벽 가이드](https://www.frontoverflow.com/magazine/21/Model%20Context%20Protocol)

**Claude Skills 관련:**
- [Claude Skills 종합 가이드](https://tilnote.io/en/pages/68f194458a9f56cc87d63eaf)
- [API를 사용하여 Agent Skills 활용하기](https://platform.claude.com/docs/ko/build-with-claude/skills-guide)
- [GitHub - awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills)

**비교 분석:**
- [Claude Skills vs. MCP: A Technical Comparison](https://intuitionlabs.ai/articles/claude-skills-vs-mcp)
- [Skills vs Dynamic MCP Loadouts](https://lucumr.pocoo.org/2025/12/13/skills-vs-mcp/)
- [Claude Skills는 굉장하다, MCP보다 더 큰 혁신일지도](https://news.hada.io/topic?id=23734)
- [요즘IT - MCP의 다음은 클로드 스킬인가?](https://yozm.wishket.com/magazine/detail/3508/)