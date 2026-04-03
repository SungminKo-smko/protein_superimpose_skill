# superimpose-protein 스킬

단백질 CIF 구조 파일의 superimposition(구조 정렬)을 위한 Claude Code/Desktop 스킬입니다.
`protein-superimpose` MCP 서버와 연동하여 워크플로 가이드를 제공합니다.

## 설치 방법

### 1. 전역 설치 (모든 프로젝트에서 사용)

```bash
bash install_skill.sh
```

스킬이 `~/.claude/skills/superimpose-protein/`에 설치됩니다.

### 2. 프로젝트 설치 (특정 프로젝트에서만 사용)

```bash
bash install_skill.sh --project
# 또는 프로젝트 경로 지정
bash install_skill.sh --project /path/to/project
```

스킬이 `<project>/.claude/skills/superimpose-protein/`에 설치됩니다.

## 사용 방법

설치 후 새 Claude Code 세션을 시작하면 스킬이 자동으로 인식됩니다.
다음과 같이 요청하세요:

- "이 디렉토리의 CIF 파일들을 superimpose 해줘"
- "구조 정렬 해줘"
- "chain A 기준으로 alignment 해줘"

## MCP 서버 설치 (필수)

이 스킬은 워크플로 가이드만 제공하며, 실제 정렬 작업은 `protein-superimpose` MCP 서버가 수행합니다.
MCP 서버를 별도로 설치해야 합니다:

```bash
pip install protein-superimpose-mcp
```

또는 소스에서 설치:

```bash
git clone https://github.com/SungminKo-smko/protein_superimpose_mcp.git
cd protein_superimpose_mcp
pip install -e .
```

설치 후 Claude 설정 파일에 MCP 서버를 등록하세요:

```json
{
  "mcpServers": {
    "protein-superimpose": {
      "command": "python",
      "args": ["-m", "protein_superimpose_mcp"]
    }
  }
}
```

## 라이선스

MIT
