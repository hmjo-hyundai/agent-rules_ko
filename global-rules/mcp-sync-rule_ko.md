# MCP 서버 설정 동기화 규칙

## 목적
Claude Code, VS Code, Windsurf, Cursor, Claude Desktop 애플리케이션 간 MCP (Model Context Protocol) 서버 설정을 동기화합니다.

## 설정 파일 위치

### macOS
- **Claude Desktop**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Cursor**: `~/.cursor/mcp.json`
- **Windsurf**: `~/.codeium/windsurf/mcp_config.json`
- **Claude Code**: `~/.claude.json` (MCP 설정 외에도 더 많은 것을 포함)
- **VS Code (사용자)**: `~/Library/Application Support/Code/User/settings.json` (`mcp.servers` 하위)

### Windows
- **Claude Desktop**: `%APPDATA%\Claude\claude_desktop_config.json`
- **Cursor**: `%USERPROFILE%\.cursor\mcp.json`
- **Windsurf**: `%USERPROFILE%\.codeium\windsurf\mcp_config.json`
- **VS Code (사용자)**: `%APPDATA%\Code\User\settings.json` (`mcp.servers` 하위)

### Linux
- **Claude Desktop**: `~/.config/Claude/claude_desktop_config.json`
- **Cursor**: `~/.cursor/mcp.json`
- **Windsurf**: `~/.codeium/windsurf/mcp_config.json`
- **VS Code (사용자)**: `~/.config/Code/User/settings.json` (`mcp.servers` 하위)

## 설정 형식 차이점

### Claude Desktop & Cursor 형식
```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "package-name"],
      "env": {
        "ENV_VAR": "value"
      }
    }
  }
}
```

### VS Code 형식
```json
{
  "mcp": {
    "servers": {
      "server-name": {
        "command": "npx",
        "args": ["-y", "package-name"],
        "env": {
          "ENV_VAR": "value"
        }
      }
    }
  }
}
```

### Claude Code 형식
**참고**: Claude Code (CLI 도구)는 현재 MCP 서버 설정을 지원하지 않습니다. `.claude.json` 파일은 프로젝트 설정과 기타 설정을 포함하지만 MCP 서버는 포함하지 않습니다.

## 동기화 스크립트

```bash
#!/bin/bash

# MCP 서버 설정 동기화기
# 이 스크립트는 다양한 애플리케이션에서 MCP 서버 설정을 찾고 비교합니다

# 출력용 컬러 코드
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # 컬러 없음

# 설정 파일 경로
CLAUDE_DESKTOP_CONFIG="$HOME/Library/Application Support/Claude/claude_desktop_config.json"
CURSOR_CONFIG="$HOME/.cursor/mcp.json"
CLAUDE_CODE_CONFIG="$HOME/.claude.json"

# jq가 설치되어 있는지 확인
if ! command -v jq &> /dev/null; then
    echo -e "${RED}오류: jq가 설치되지 않았습니다. 다음으로 설치하세요: brew install jq${NC}"
    exit 1
fi

echo "🔍 MCP 서버 설정 동기화기"
echo "======================================="
echo ""

# 설정 파일에서 MCP 서버를 추출하는 함수
extract_servers() {
    local file="$1"
    local format="$2"
    
    if [ ! -f "$file" ]; then
        echo "{}"
        return
    fi
    
    if [ "$format" = "vscode" ]; then
        jq -r '.mcp.servers // {}' "$file" 2>/dev/null || echo "{}"
    else
        jq -r '.mcpServers // {}' "$file" 2>/dev/null || echo "{}"
    fi
}

# JSON에서 서버 이름을 가져오는 함수
get_server_names() {
    echo "$1" | jq -r 'keys[]' 2>/dev/null | sort
}

# 어떤 설정 파일이 존재하는지 확인
echo "📁 설정 파일 확인 중:"
echo ""

if [ -f "$CLAUDE_DESKTOP_CONFIG" ]; then
    echo -e "${GREEN}✓${NC} Claude Desktop: $CLAUDE_DESKTOP_CONFIG"
    CLAUDE_DESKTOP_SERVERS=$(extract_servers "$CLAUDE_DESKTOP_CONFIG" "standard")
else
    echo -e "${RED}✗${NC} Claude Desktop: 찾을 수 없음"
    CLAUDE_DESKTOP_SERVERS="{}"
fi

if [ -f "$CURSOR_CONFIG" ]; then
    echo -e "${GREEN}✓${NC} Cursor: $CURSOR_CONFIG"
    CURSOR_SERVERS=$(extract_servers "$CURSOR_CONFIG" "standard")
else
    echo -e "${RED}✗${NC} Cursor: 찾을 수 없음"
    CURSOR_SERVERS="{}"
fi

if [ -f "$CLAUDE_CODE_CONFIG" ]; then
    echo -e "${GREEN}✓${NC} Claude Code: $CLAUDE_CODE_CONFIG"
    CLAUDE_CODE_SERVERS=$(extract_servers "$CLAUDE_CODE_CONFIG" "standard")
else
    echo -e "${RED}✗${NC} Claude Code: 찾을 수 없음"
    CLAUDE_CODE_SERVERS="{}"
fi

# VS Code 워크스페이스 설정 찾기
echo ""
echo "🔍 VS Code 워크스페이스 설정 검색 중..."
VSCODE_CONFIGS=$(find ~ -path "*/.vscode/mcp.json" -type f 2>/dev/null | head -10)
if [ -n "$VSCODE_CONFIGS" ]; then
    echo -e "${GREEN}✓${NC} VS Code 워크스페이스 설정 발견:"
    echo "$VSCODE_CONFIGS" | while read -r config; do
        echo "  - $config"
    done
else
    echo -e "${YELLOW}⚠${NC} VS Code 워크스페이스 설정을 찾을 수 없음"
fi

echo ""
echo "📊 서버 분석:"
echo "=================="

# 모든 고유한 서버 이름 가져오기
ALL_SERVERS=$(echo "$CLAUDE_DESKTOP_SERVERS $CURSOR_SERVERS $CLAUDE_CODE_SERVERS" | \
    jq -s 'add | keys' | jq -r '.[]' | sort -u)

# 추적용 배열 생성
declare -a COMMON_SERVERS=()
declare -a CLAUDE_DESKTOP_ONLY=()
declare -a CURSOR_ONLY=()
declare -a CLAUDE_CODE_ONLY=()

# 각 서버 분석
for server in $ALL_SERVERS; do
    in_claude_desktop=$(echo "$CLAUDE_DESKTOP_SERVERS" | jq --arg s "$server" 'has($s)')
    in_cursor=$(echo "$CURSOR_SERVERS" | jq --arg s "$server" 'has($s)')
    in_claude_code=$(echo "$CLAUDE_CODE_SERVERS" | jq --arg s "$server" 'has($s)')
    
    count=0
    [ "$in_claude_desktop" = "true" ] && ((count++))
    [ "$in_cursor" = "true" ] && ((count++))
    [ "$in_claude_code" = "true" ] && ((count++))
    
    if [ $count -ge 2 ]; then
        COMMON_SERVERS+=("$server")
    elif [ "$in_claude_desktop" = "true" ] && [ $count -eq 1 ]; then
        CLAUDE_DESKTOP_ONLY+=("$server")
    elif [ "$in_cursor" = "true" ] && [ $count -eq 1 ]; then
        CURSOR_ONLY+=("$server")
    elif [ "$in_claude_code" = "true" ] && [ $count -eq 1 ]; then
        CLAUDE_CODE_ONLY+=("$server")
    fi
done

# 결과 표시
echo ""
echo "🤝 공통 서버 (2개 이상 앱에서):"
if [ ${#COMMON_SERVERS[@]} -eq 0 ]; then
    echo "  없음"
else
    for server in "${COMMON_SERVERS[@]}"; do
        echo -e "  ${GREEN}•${NC} $server"
        
        # 어떤 앱에 있는지 표시
        apps=""
        [ "$(echo "$CLAUDE_DESKTOP_SERVERS" | jq --arg s "$server" 'has($s)')" = "true" ] && apps+="Claude Desktop, "
        [ "$(echo "$CURSOR_SERVERS" | jq --arg s "$server" 'has($s)')" = "true" ] && apps+="Cursor, "
        [ "$(echo "$CLAUDE_CODE_SERVERS" | jq --arg s "$server" 'has($s)')" = "true" ] && apps+="Claude Code, "
        apps=${apps%, }
        echo "    위치: $apps"
    done
fi

echo ""
echo "📱 앱별 전용 서버:"

if [ ${#CLAUDE_DESKTOP_ONLY[@]} -gt 0 ]; then
    echo -e "\n  ${YELLOW}Claude Desktop 전용:${NC}"
    for server in "${CLAUDE_DESKTOP_ONLY[@]}"; do
        echo "  • $server"
    done
fi

if [ ${#CURSOR_ONLY[@]} -gt 0 ]; then
    echo -e "\n  ${YELLOW}Cursor 전용:${NC}"
    for server in "${CURSOR_ONLY[@]}"; do
        echo "  • $server"
    done
fi

if [ ${#CLAUDE_CODE_ONLY[@]} -gt 0 ]; then
    echo -e "\n  ${YELLOW}Claude Code 전용:${NC}"
    for server in "${CLAUDE_CODE_ONLY[@]}"; do
        echo "  • $server"
    done
fi

# 공통 서버의 설정 차이점
echo ""
echo "🔧 설정 차이점:"
echo "============================"

for server in "${COMMON_SERVERS[@]}"; do
    echo ""
    echo "서버: $server"
    echo "---------------"
    
    # 각 앱에서 설정 가져오기
    claude_desktop_config=$(echo "$CLAUDE_DESKTOP_SERVERS" | jq --arg s "$server" '.[$s]' 2>/dev/null)
    cursor_config=$(echo "$CURSOR_SERVERS" | jq --arg s "$server" '.[$s]' 2>/dev/null)
    claude_code_config=$(echo "$CLAUDE_CODE_SERVERS" | jq --arg s "$server" '.[$s]' 2>/dev/null)
    
    # 설정 비교
    configs_match=true
    
    if [ "$claude_desktop_config" != "null" ] && [ "$cursor_config" != "null" ]; then
        if [ "$claude_desktop_config" != "$cursor_config" ]; then
            configs_match=false
        fi
    fi
    
    if [ "$configs_match" = true ]; then
        echo -e "  ${GREEN}✓${NC} 앱 간 설정이 일치함"
    else
        echo -e "  ${RED}✗${NC} 설정이 다름:"
        
        if [ "$claude_desktop_config" != "null" ]; then
            echo "    Claude Desktop:"
            echo "$claude_desktop_config" | jq . | sed 's/^/      /'
        fi
        
        if [ "$cursor_config" != "null" ] && [ "$cursor_config" != "$claude_desktop_config" ]; then
            echo "    Cursor:"
            echo "$cursor_config" | jq . | sed 's/^/      /'
        fi
        
        if [ "$claude_code_config" != "null" ] && [ "$claude_code_config" != "$claude_desktop_config" ] && [ "$claude_code_config" != "$cursor_config" ]; then
            echo "    Claude Code:"
            echo "$claude_code_config" | jq . | sed 's/^/      /'
        fi
    fi
done

# 요약
echo ""
echo "📈 요약:"
echo "=========="
total_servers=$(echo "$ALL_SERVERS" | wc -w | tr -d ' ')
echo "총 고유 서버: $total_servers"
echo "공통 서버: ${#COMMON_SERVERS[@]}"
echo "앱별 전용 서버: $((total_servers - ${#COMMON_SERVERS[@]}))"

# 동기화 권장사항
echo ""
echo "💡 동기화 권장사항:"
echo "======================="

if [ ${#COMMON_SERVERS[@]} -gt 0 ]; then
    echo "• 공통 서버의 설정 차이점 검토"
fi

total_unique=$((${#CLAUDE_DESKTOP_ONLY[@]} + ${#CURSOR_ONLY[@]} + ${#CLAUDE_CODE_ONLY[@]}))
if [ $total_unique -gt 0 ]; then
    echo "• 일관성을 위해 앱별 전용 서버를 다른 앱에 추가 고려"
fi

echo ""
echo "설정을 수동으로 동기화하려면 설정 파일 간 mcpServers 섹션을 복사하세요."
echo "VS Code의 경우 형식을 조정하세요 ('mcpServers' 대신 'mcp.servers'로 래핑)."
```

## 사용 지침

1. 위 스크립트를 `mcp-sync.sh`로 저장
2. 실행 가능하게 만들기: `chmod +x mcp-sync.sh`
3. 다양한 옵션으로 스크립트 사용:

### 기본 사용법

```bash
# 모든 서버 나열 및 차이점 표시
./mcp-sync.sh

# 도움말 표시
./mcp-sync.sh --help
```

### 로컬과 글로벌 설정 간 전환

```bash
# 서버를 로컬 경로에서 글로벌 npm 패키지로 전환
./mcp-sync.sh --to-global terminator --app cursor

# 서버를 글로벌 npm 패키지에서 로컬 경로로 전환
./mcp-sync.sh --to-local peekaboo --app claude-desktop
```

### 기능

스크립트는 다음을 수행합니다:
- 시스템의 모든 MCP 설정 파일 찾기
- 서버 설정 파싱 및 비교
- 앱 간 공통 서버 표시
- 특정 앱에만 있는 서버 표시
- 공통 서버의 설정 차이점 강조
- 동기화 권장사항 제공
- **신규**: 로컬 개발 경로와 글로벌 npm 패키지 간 서버 전환

### 전환 세부사항

**로컬에서 글로벌로 (`--to-global`)**:
- 로컬 파일 경로 설정을 npm 패키지와 함께 `npx`를 사용하도록 변환
- 공통 서버들을 자동 인식 (terminator, agent, automator, conduit, peekaboo)
- 알 수 없는 서버에 대해 npm 패키지 이름 입력 요청
- 환경 변수 및 기타 설정 보존

**글로벌에서 로컬로 (`--to-local`)**:
- npm 패키지 설정을 로컬 파일 경로 사용으로 변환
- 파일 타입을 자동 감지하고 적절한 명령 선택 (node, python, bash)
- 로컬 파일이 존재하는지 검증 (어쨌든 계속하는 옵션 포함)
- 환경 변수 및 기타 설정 보존

## 수동 동기화 단계

1. **차이점 식별**: 스크립트를 실행하여 어떤 서버가 다른지 확인
2. **신뢰 원본 선택**: 어떤 앱이 올바른 설정을 가지고 있는지 결정
3. **설정 복사**: 
   - Claude Desktop/Cursor의 경우: `mcpServers`에서 전체 서버 객체 복사
   - VS Code의 경우: `mcp.servers`로 복사 (다른 구조 주의)
4. **민감한 데이터 처리**: API 키와 비밀번호 주의
5. **테스트**: 설정이 작동하는지 확인하기 위해 애플리케이션 재시작

## 보안 고려사항

- **API 키**: 가능한 경우 환경 변수에 민감한 키 저장
- **비밀번호**: 설정 파일에 평문 비밀번호 저장 피하기
- **파일 권한**: 설정 파일이 적절한 권한(600)을 가지는지 확인
- **버전 관리**: 민감한 데이터가 있는 설정 파일을 커밋하지 않기

## 자동화 아이디어

1. **예약된 동기화**: cron을 사용하여 정기적으로 동기화 스크립트 실행
2. **Git 훅**: 변경사항을 pull/push할 때 설정 동기화
3. **설정 템플릿**: 단일 소스에서 설정을 생성하는 템플릿 시스템 사용
4. **환경 기반**: 다른 환경에 대해 다른 설정 사용