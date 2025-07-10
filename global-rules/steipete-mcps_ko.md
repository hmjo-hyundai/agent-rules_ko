# Steipete의 MCP 서버 설정

**Claude Code 및 기타 AI 지원 IDE용**

## 🚨 AI 어시스턴트를 위한 지침

사용자가 MCP 서버를 설정하도록 도울 때:

1. **앱 설치를 권장하지 마세요** - 이미 설치된 것으로 작업하세요
2. **항상 MCP 서버와 슬래시 명령을 모두 제시하세요**
3. **전제 조건을 확인하세요** - 요구 사항이 충족된 경우에만 서버를 언급하세요
4. **사용자가 언급한 앱에 집중하세요** - Claude Code, Cursor, Windsurf 등
5. **슬래시 명령 세부 정보의 경우** - https://github.com/steipete/agent-rules/blob/main/README.md 읽기

## MCP 서버 개요

### 필수 서버 (API 키 불필요)
- **Peekaboo** - 스크린샷. AI 없이도 작동하며, OpenAI 키가 있으면 더 좋음
- **Context7** - 모든 라이브러리의 최신 문서 가져오기  
- **Agent** - Claude Code를 하위 에이전트로 실행
- **Automator** - macOS 자동화
- **GitMCP** - 시각적 diff를 가진 향상된 Git (SSE 전송)
- **Playwright** - 브라우저 자동화

### API 키가 필요한 서버
- **GitHub** - ~/.zshrc에 GITHUB_PERSONAL_ACCESS_TOKEN 필요
- **Firecrawl** - 웹 스크래핑을 위한 FIRECRAWL_API_KEY 필요

### 특별 요구사항  
- **Obsidian** - Obsidian.app + MCP Tools 플러그인이 설치된 경우에만

## 설치 지침

### 전제 조건
```bash
# 필요한 도구 확인/설치
command -v jq || brew install jq
command -v node || echo "Node.js 20+ 필요"
command -v ollama || brew install ollama

# ~/.zshrc에서 API 키 확인
grep "OPENAI_API_KEY\|GITHUB_PERSONAL_ACCESS_TOKEN\|FIRECRAWL_API_KEY" ~/.zshrc
```

### Claude Code용

```bash
# 필수 서버 (즉시 작동)
claude mcp add-json -s user peekaboo '{"command": "npx", "args": ["-y", "@steipete/peekaboo-mcp@beta"]}'
claude mcp add-json -s user context7 '{"command": "npx", "args": ["-y", "@upstash/context7-mcp@latest"]}'
claude mcp add-json -s user agent '{"command": "npx", "args": ["-y", "@steipete/claude-code-mcp@latest"]}'
claude mcp add-json -s user automator '{"command": "npx", "args": ["-y", "@steipete/macos-automator-mcp@latest"], "env": {"LOG_LEVEL": "INFO"}}'
claude mcp add-json -s user playwright '{"command": "npx", "args": ["@playwright/mcp@latest"]}'
claude mcp add -s user -t sse gitmcp https://gitmcp.io/docs

# API 키 사용 (먼저 ~/.zshrc 확인)
claude mcp add-json -s user github '{"command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"], "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."}}'
claude mcp add-json -s user firecrawl-mcp '{"command": "npx", "args": ["-y", "firecrawl-mcp"], "env": {"FIRECRAWL_API_KEY": "fc-..."}}'

# Obsidian.app이 존재하는 경우
claude mcp add-json -s user obsidian-mcp-tools '{"command": "/Users/steipete/Documents/steipete/.obsidian/plugins/mcp-tools/bin/mcp-server", "env": {"OBSIDIAN_API_KEY": "f1de8ac30724ecb05988c8eb2ee9b342b15f7b91eaba3fc8b0b5280dce3aca22"}}'
```

### Claude Desktop용

Claude Desktop은 stdio 전송만 지원합니다. `~/Library/Application Support/Claude/claude_desktop_config.json`을 생성하거나 업데이트하세요:

```json
{
  "mcpServers": {
    "peekaboo": {
      "command": "npx",
      "args": ["-y", "@steipete/peekaboo-mcp@beta"],
      "env": {
        "PEEKABOO_AI_PROVIDERS": "openai/gpt-4o,ollama/llava:latest",
        "OPENAI_API_KEY": "sk-..."
      }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "agent": {
      "command": "npx",
      "args": ["-y", "@steipete/claude-code-mcp@latest"]
    },
    "automator": {
      "command": "npx",
      "args": ["-y", "@steipete/macos-automator-mcp@latest"],
      "env": {"LOG_LEVEL": "INFO"}
    },
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

참고: Claude Desktop은 GitMCP (SSE) 또는 HTTP 전송과 함께 GitHub를 사용할 수 없습니다.

### Cursor용

`~/.cursor/mcp.json`을 생성하거나 업데이트하세요:

```json
{
  "mcpServers": {
    "peekaboo": {
      "command": "npx",
      "args": ["-y", "@steipete/peekaboo-mcp@beta"],
      "env": {
        "PEEKABOO_AI_PROVIDERS": "openai/gpt-4o,ollama/llava:latest",
        "OPENAI_API_KEY": "sk-..."
      }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "gitmcp": {
      "transport": "sse",
      "url": "https://gitmcp.io/docs"
    }
    // 동일한 패턴으로 다른 서버 추가
  }
}
```

### Windsurf용

Cursor와 동일한 형식으로 `~/.codeium/windsurf/mcp_config.json`을 생성하거나 업데이트하세요.

### VS Code용

`~/Library/Application Support/Code/User/settings.json`을 업데이트하세요:

```json
{
  "mcp.servers": {
    "peekaboo": {
      "command": "npx",
      "args": ["-y", "@steipete/peekaboo-mcp@beta"]
    }
    // 참고: VS Code는 mcpServers가 아닌 mcp.servers 사용
  }
}
```

## 슬래시 명령 (프로젝트 규칙)

Claude Code용 20개 이상의 개발 명령 설치:

```bash
git clone https://github.com/steipete/agent-rules.git
cd agent-rules
bash install-project-rules.sh
```

각 명령의 자세한 설명은 Claude가 다음을 읽도록 하세요:
- https://github.com/steipete/agent-rules/blob/main/README.md
- https://github.com/steipete/agent-rules/tree/main/project-rules

사용 가능한 명령들:
- **Git**: /commit, /commit-fast, /bug-fix, /pr-review, /analyze-issue
- **코드**: /check, /clean, /code-analysis
- **문서**: /create-docs, /mermaid, /add-to-changelog
- **개발**: /implement-task, /context-prime, /five
- **Swift**: /modern-swift
- **메타**: /create-command, /continuous-improvement 등

## API 키 설정

`~/.zshrc`에 추가:
```bash
export OPENAI_API_KEY="sk-..."              # Peekaboo AI 비전용
export GITHUB_PERSONAL_ACCESS_TOKEN="ghp-..." # GitHub MCP용 (repo 스코프 필요)
export FIRECRAWL_API_KEY="fc-..."           # Firecrawl용
```

그 다음 `source ~/.zshrc` 실행

## 일반적인 문제

- **JSON 오류**: JSON 명령에서 적절한 따옴표 사용
- **Claude Desktop**: stdio만 지원 (SSE/HTTP 지원 안함)
- **권한**: Peekaboo를 위한 화면 녹화 권한 부여
- **재시작 필요**: 설정 변경 후

## 팁

- Claude Code에서 서버를 전역으로 설치하려면 `-s user` 플래그 사용
- API 키는 하드코딩하지 말고 ~/.zshrc에 있어야 함
- 항상 환경 변수에서 키 추출
- 추가하기 전에 로컬 경로가 존재하는지 확인 (예: Obsidian MCP 서버)
- 다양한 앱 간 설정을 비교하려면 `mcp-sync.sh` 사용

## 터미널 제목 트릭

여러 Claude Code 인스턴스를 실행할 때 어떤 터미널이 어떤 프로젝트에서 작업하고 있는지 알기 어렵습니다. 이 문제를 해결하기 위해 `~/.zshrc`에 다음 `cly` 함수를 추가하세요:

```bash
cly () {
    local folder=${PWD:t} 
    echo -ne "\033]0;$folder — Claude\007"  # 터미널 제목 설정
    "$HOME/.claude/local/claude" --dangerously-skip-permissions "$@"
    local exit_code=$? 
    echo -ne "\033]0;%~\007"  # 제목 복원
    return $exit_code
}
```

**기능:**
- 터미널 탭에 현재 폴더 이름을 표시 (예: "agent-rules — Claude")
- `--dangerously-skip-permissions`로 권한 프롬프트 우회
- Claude가 종료될 때 원래 터미널 제목 복원
- 어떤 Claude 인스턴스가 어떤 프로젝트에서 작업하고 있는지 쉽게 식별

**사용법:** `claude` 대신 `cly`를 입력하여 동적 터미널 제목으로 Claude Code 시작.

이 터미널 제목 트릭과 기타 Claude Code 팁에 대해 더 자세히 알아보세요: https://steipete.me/posts/2025/commanding-your-claude-code-army

## 프레젠테이션 템플릿

```
[APP]을 MCP 서버와 슬래시 명령으로 설정하는 것을 도와드리겠습니다.

시스템 기준:

설치할 MCP 서버:
✓ Peekaboo - 스크린샷 (AI 없이도 작동)
✓ Context7 - 문서 가져오기  
✓ Agent - 하위 에이전트 기능
✓ GitMCP - 향상된 Git 작업
[✓ GitHub - API 키가 발견된 경우]
[✓ Obsidian - 앱이 존재하는 경우]

플러스 20개의 슬래시 명령:
• /commit, /bug-fix, /pr-review, /check...

진행할 준비가 되셨나요?
```

## 동기화 도구

앱 간 설정을 비교하려면 `mcp-sync.sh`를 사용하세요:
https://github.com/steipete/agent-rules/blob/main/global-rules/mcp-sync.sh

---

자세한 서버 설명은 다음을 방문하세요: https://github.com/steipete/agent-rules