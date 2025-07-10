# Agent Rules

Claude Code와 Cursor 같은 AI 코딩 어시스턴트를 위한 재사용 가능한 규칙과 지식 문서 모음입니다.

## 저장소 구조

### 📁 project-rules/
개발 중 AI 어시스턴트가 따라야 할 실행 가능한 규칙들:

**개발 워크플로우** (크레딧: [@vincenthopf](https://github.com/vincenthopf/claude-code))
- **[commit.mdc](./project-rules/commit.mdc)** - 컨벤셔널 포맷과 이모지를 사용한 표준 커밋
- **[commit-fast.mdc](./project-rules/commit-fast.mdc)** - 첫 번째 메시지를 자동 선택하는 빠른 커밋 워크플로우
- **[add-to-changelog.mdc](./project-rules/add-to-changelog.mdc)** - Keep a Changelog 형식을 따르는 구조화된 변경로그 업데이트
- **[pr-review.mdc](./project-rules/pr-review.mdc)** - 다중 역할 풀 리퀘스트 리뷰 체크리스트

**코드 품질 & 분석**
- **[check.mdc](./project-rules/check.mdc)** - 다양한 언어에 대한 포괄적인 코드 품질 검사
- **[clean.mdc](./project-rules/clean.mdc)** - 모든 포맷팅 및 린팅 이슈 수정
- **[code-analysis.mdc](./project-rules/code-analysis.mdc)** - 고급 다면적 코드 분석 옵션

**문제 해결 & 구현**
- **[analyze-issue.mdc](./project-rules/analyze-issue.mdc)** - GitHub 이슈 분석 및 구현 명세
- **[bug-fix.mdc](./project-rules/bug-fix.mdc)** - 이슈부터 PR까지 완전한 버그 수정 워크플로우
- **[implement-task.mdc](./project-rules/implement-task.mdc)** - 체계적인 작업 구현 접근법
- **[five.mdc](./project-rules/five.mdc)** - 5가지 Why 근본 원인 분석 기법

**문서화 & 시각화**
- **[create-docs.mdc](./project-rules/create-docs.mdc)** - 포괄적인 문서 생성
- **[mermaid.mdc](./project-rules/mermaid.mdc)** - 다양한 시각화를 위한 Mermaid 다이어그램 생성

**프로젝트 설정 & 메타**
- **[context-prime.mdc](./project-rules/context-prime.mdc)** - 프로젝트 컨텍스트 포괄적 로딩
- **[create-command.mdc](./project-rules/create-command.mdc)** - 새로운 사용자 정의 명령 생성 가이드
- **[continuous-improvement.mdc](./project-rules/continuous-improvement.mdc)** - AI 어시스턴트 규칙 개선을 위한 체계적 접근법
- **[cursor-rules-meta-guide.mdc](./project-rules/cursor-rules-meta-guide.mdc)** - Cursor 규칙 생성 및 유지보수 가이드라인

**자동화 & 통합**
- **[mcp-inspector-debugging.mdc](./project-rules/mcp-inspector-debugging.mdc)** - Inspector UI로 MCP 서버 디버깅
- **[safari-automation.mdc](./project-rules/safari-automation.mdc)** - 고급 Safari 브라우저 자동화 기법
- **[screenshot-automation.mdc](./project-rules/screenshot-automation.mdc)** - 자동화된 스크린샷을 위한 AppleScript 패턴

**언어별**
- **[modern-swift.mdc](./project-rules/modern-swift.mdc)** - Apple의 최신 모범 사례를 따르는 모던 SwiftUI 아키텍처 ([Dimillian의 "Forget MVVM"](https://dimillian.medium.com/swiftui-in-2025-forget-mvvm-262ff2bbd2ed)에서 영감을 받음)

### 📁 docs/
참조 문서 및 지식 베이스:
- **Swift 개발**
  - [swift-observable.mdc](./docs/swift-observable.mdc) - ObservableObject에서 @Observable 매크로로의 마이그레이션 가이드
  - [swift-observation.mdc](./docs/swift-observation.mdc) - Swift Observation 프레임워크 문서
  - [swift-testing-api.mdc](./docs/swift-testing-api.mdc) - Swift Testing 프레임워크 API 참조
  - [swift-testing-playbook.mdc](./docs/swift-testing-playbook.mdc) - Swift Testing으로 마이그레이션하기 위한 포괄적 가이드
  - [swift-argument-parser.mdc](./docs/swift-argument-parser.mdc) - Swift Argument Parser 프레임워크 문서
  - [swift6-migration.mdc](./docs/swift6-migration.mdc) - 동시성과 함께 Swift 6로 마이그레이션하는 가이드

- **MCP 개발**
  - [mcp-best-practices.mdc](./docs/mcp-best-practices.mdc) - Model Context Protocol 서버 구축 모범 사례
  - [mcp-releasing.mdc](./docs/mcp-releasing.mdc) - MCP 서버를 NPM 패키지로 릴리스하는 가이드

### 📁 global-rules/
글로벌 Claude Code 설정 및 자동화 스크립트 (`~/.claude/CLAUDE.md`에 배치):
- **[github-issue-creation.mdc](./global-rules/github-issue-creation.mdc)** - 잘 구조화된 GitHub 이슈 생성 (크레딧: [@nityeshaga](https://x.com/nityeshaga/status/1933113428379574367))
- **[mcp-peekaboo-setup.mdc](./global-rules/mcp-peekaboo-setup.mdc)** - 비전 지원 Peekaboo MCP 서버 설정 가이드
- **[terminal-title-wrapper.zsh](./global-rules/terminal-title-wrapper.zsh)** - 동적 터미널 제목을 위한 ZSH 래퍼
- **[mcp-sync.sh](./global-rules/mcp-sync.sh)** - Claude 설치 간 MCP 서버 동기화 스크립트
- **[mcp-sync-rule.md](./global-rules/mcp-sync-rule.md)** - MCP 동기화 기능에 대한 문서

## 사용법

### Cursor 사용자용
1. `project-rules/`에서 임의의 `.mdc` 파일을 프로젝트의 `.cursor/rules/` 디렉토리로 복사
2. Cursor가 frontmatter의 glob 패턴에 따라 자동으로 규칙 적용
3. `alwaysApply: true`가 있는 규칙들은 모든 파일에 대해 활성화
4. `docs/`의 문서는 필요에 따라 참조하거나 가져오기 가능

### Claude Code 사용자용
1. 임의의 `.mdc` 파일의 내용(frontmatter 제외)을 `CLAUDE.md` 파일에 복사
2. 또는 `CLAUDE.md`에서 `@import` 구문을 사용하여 전체 파일 참조
3. 프로젝트 루트나 글로벌 규칙을 위해 `~/.claude/CLAUDE.md`에 배치
4. 프로젝트 규칙과 문서 모두 포함 가능

## 글로벌 Claude Code 규칙

모든 프로젝트에서 Claude Code의 기능을 향상시키기 위해 `~/.claude/CLAUDE.md`에 배치할 수 있는 강력한 글로벌 규칙들입니다. ["Commanding Your Claude Code Army"](https://steipete.me/posts/2025/commanding-your-claude-code-army)의 전략을 기반으로 합니다.

### 사용 가능한 글로벌 규칙

#### 1. GitHub 이슈 생성
기능 설명을 모범 사례를 따르는 잘 구조화된 GitHub 이슈로 변환합니다.
- **크레딧:** [@nityeshaga](https://x.com/nityeshaga/status/1933113428379574367)
- **기능:** 저장소 연구, 컨벤션 분석, 자동 `gh issue create` 통합
- **사용법:** 기능 설명과 저장소 URL 제공

#### 2. MCP 서버 설정 - Peekaboo
비전 지원 Peekaboo MCP 서버의 자동화된 설정.
- **기능:** AI 분석을 통한 스크린샷 캡처, 이중 제공자 지원 (OpenAI/Ollama)
- **보안:** `~/.zshrc`에서 안전한 API 키 추출
- **요구사항:** Node.js 20.0+, macOS 14.0+

#### 3. 터미널 제목 관리
더 나은 다중 인스턴스 조직화를 위한 동적 터미널 제목.
- **기능:** `~/path/to/project — Claude` 형식 표시
- **구현:** 백그라운드 제목 지속성을 가진 ZSH 래퍼 함수 (`cly`)
- **이점:** 다중 Claude 인스턴스의 쉬운 식별

### 설치

1. **Claude 설정 디렉토리 생성:**
   ```bash
   mkdir -p ~/.claude
   ```

2. **글로벌 규칙 설정:**
   ```bash
   # 글로벌 CLAUDE.md 생성 또는 편집
   nano ~/.claude/CLAUDE.md
   # 이 저장소에서 원하는 규칙 추가
   ```

3. **터미널 제목 관리용:**
   ```bash
   # 래퍼 스크립트 복사
   cp global-rules/terminal-title-wrapper.zsh ~/.config/zsh/claude-wrapper.zsh
   mkdir -p ~/.config/zsh
   # claude-wrapper.zsh 내용 추가
   
   # ~/.zshrc에서 소스
   echo '[[ -f ~/.config/zsh/claude-wrapper.zsh ]] && source ~/.config/zsh/claude-wrapper.zsh' >> ~/.zshrc
   ```

## 기여하기

자신만의 규칙을 자유롭게 기여해주세요! 다음을 확인해주세요:
1. `.mdc` 확장자 사용
2. `description`, `globs`, `alwaysApply` 필드가 있는 적절한 YAML frontmatter 포함
3. 명확하고 실행 가능한 지침 포함
4. 프로젝트 간 재사용할 수 있을 만큼 일반적
5. 적절한 디렉토리에 배치:
   - `project-rules/` - 실행 가능한 AI 어시스턴트 규칙용
   - `docs/` - 참조 문서용

## 왜 이 형식인가?

이 저장소는 Claude Code와 Cursor 모두에서 원활하게 작동하는 통합된 접근법을 제공하는 `.mdc` (Markdown with Configuration) 형식을 사용합니다:

- **Cursor**는 규칙 설정을 위한 YAML frontmatter가 있는 `.mdc` 파일을 기본적으로 지원
- **Claude Code**는 frontmatter 메타데이터를 무시하고 마크다운 내용을 읽음
- YAML frontmatter는 Cursor가 지능적인 규칙 적용에 사용하는 선택적 메타데이터(설명, 파일 glob, alwaysApply) 제공
- 표준 마크다운 내용으로 다른 AI 어시스턴트 간 호환성 보장

이 통합된 형식으로 수정 없이 두 도구에서 동일한 규칙 파일을 사용할 수 있습니다.

## 라이선스

MIT License - 자세한 내용은 [LICENSE](./LICENSE)를 참조하세요