---
name: superimpose-protein
description: >
  단백질 CIF 구조 파일들을 지정한 chain의 Ca 원자를 기준으로 superimpose.
  "구조 정렬", "superimpose", "alignment", "CIF 정렬" 키워드가 포함될 때 사용.
  MCP 서버(protein-superimpose)를 통해 두 가지 모드를 지원:
  (1) 단일 디렉토리 내 design_id 그룹별 정렬 (superimpose_group)
  (2) 최상위 폴더 하위 모든 CIF 파일을 단일 reference에 정렬 (superimpose_all)
---

# Protein Structure Superimposition Skill (MCP)

사용자가 단백질 CIF 구조 파일들의 superimposition을 요청했습니다.
이 스킬은 `protein-superimpose` MCP 서버를 통해 구조 정렬을 수행합니다.

## Step 1 — 파라미터 파악

아래 정보를 컨텍스트에서 추론하거나, 불명확하면 사용자에게 확인합니다:

| 파라미터 | 설명 | 기본값 |
|---------|------|--------|
| **mode** | `group` = 디렉토리 내 design_id 그룹별 정렬 / `all` = 전체 파일을 단일 reference에 정렬 | 컨텍스트 판단 |
| **input** | 입력 CIF 파일 경로 (단일 디렉토리 또는 최상위 폴더) | — |
| **output** | 출력 디렉토리 경로 | input + `_aligned` 자동 생성 |
| **chain** | Superimpose 기준 chain ID | `A` |
| **reference** | (mode=all 전용) 기준 CIF 파일 경로 | 자동 선택 (알파벳 첫 번째) |

**사전 점검 — inspect 도구 활용:**
파라미터 확인 단계에서 입력 경로의 파일 구조가 불명확하면, `inspect_structure` MCP 도구를 사용하여 대표 CIF 파일 하나를 먼저 확인합니다. 이를 통해 사용 가능한 chain ID, 잔기 수, 파일 형식 등을 사전에 파악할 수 있습니다.

**모드 판단 기준:**
- 파일명에 `_model_N.cif` 패턴이 있고 같은 디자인의 여러 예측 모델을 정렬 -> `group` 모드
- 여러 서브폴더에 걸친 모든 구조를 한 기준에 정렬 (h_/m_ 접두사 등) -> `all` 모드

## Step 2 — MCP 서버 확인

`protein-superimpose` MCP 서버가 사용 가능한지 확인합니다.

**확인 방법:**
MCP 도구 목록에서 다음 도구들이 존재하는지 확인합니다:
- `inspect_structure`
- `superimpose_group`
- `superimpose_all`

**MCP 서버가 없는 경우:**
사용자에게 다음 안내를 제공합니다:

```
protein-superimpose MCP 서버가 설치되어 있지 않습니다.

설치 방법:
  pip install protein-superimpose-mcp

또는 소스에서 설치:
  git clone https://github.com/SungminKo-smko/protein_superimpose_mcp.git
  cd protein_superimpose_mcp
  pip install -e .

설치 후 Claude Desktop/Code 설정에서 MCP 서버를 등록하세요:
  {
    "mcpServers": {
      "protein-superimpose": {
        "command": "python",
        "args": ["-m", "protein_superimpose_mcp"]
      }
    }
  }
```

## Step 3 — 사전 검증 (Pre-flight Inspection)

정렬 실행 전에 `inspect_structure` MCP 도구로 입력 파일을 검증합니다:

1. 대표 CIF 파일 1~2개를 선택하여 `inspect_structure`를 호출합니다
2. 확인 항목:
   - 파일이 유효한 CIF 형식인지
   - 지정한 chain ID가 실제로 존재하는지
   - Ca 원자 수가 정렬에 충분한지 (최소 3개 이상)
3. 문제가 발견되면 사용자에게 보고하고, 파라미터 수정을 제안합니다

```
예시: chain A가 없는 경우
  "지정한 chain A가 파일에 존재하지 않습니다.
   사용 가능한 chain: B, C, H, L
   chain ID를 변경하시겠습니까?"
```

## Step 4 — 정렬 실행

### mode = group (superimpose_group)

MCP 도구 `superimpose_group`을 호출합니다:
- `input_dir`: 입력 디렉토리 경로
- `output_dir`: 출력 디렉토리 경로
- `chain`: 기준 chain ID
- `reference_model`: (선택) 기준 모델 번호

### mode = all (superimpose_all)

MCP 도구 `superimpose_all`을 호출합니다:
- `input_root`: 입력 최상위 폴더 경로
- `output_root`: 출력 최상위 폴더 경로
- `chain`: 기준 chain ID
- `reference`: (선택) 기준 CIF 파일 경로

## Step 5 — 결과 검증

실행 완료 후:
1. MCP 도구 반환 결과에서 성공/스킵/오류 건수를 확인하여 사용자에게 보고
2. 출력 디렉토리 파일 수 확인 (`find <output> -name "*.cif" | wc -l`)
3. 필요시 `inspect_structure`로 정렬된 결과 파일을 샘플 검증
4. 오류가 있으면 원인 파악 후 사용자에게 안내

## 문제 해결 (Troubleshooting)

### MCP 서버 연결 실패
- MCP 서버가 설치되어 있는지 확인: `pip show protein-superimpose-mcp`
- Claude 설정 파일에 서버가 등록되어 있는지 확인
- Python 환경이 올바른지 확인 (가상환경 활성화 여부)

### "chain not found" 오류
- `inspect_structure`로 실제 chain ID를 확인
- CIF 파일에 따라 chain 명명이 다를 수 있음 (A, B vs H, L 등)

### "No Ca atoms" 오류
- 입력 파일이 Ca 원자를 포함하지 않는 경우 (예: 리간드만 있는 파일)
- 해당 chain에 단백질 잔기가 없는 경우 -> 다른 chain 지정 필요

### 파일 수가 많아 속도가 느린 경우
- 10,000개 이상의 파일은 처리 시간이 길 수 있음
- 사용자에게 예상 소요 시간 안내
- 가능하면 하위 디렉토리 단위로 분할 실행 제안

### 정렬 결과 RMSD가 비정상적으로 높은 경우
- 기준 chain이 올바른지 재확인
- 잔기 번호 체계가 파일 간 일치하는지 확인
- 구조적으로 매우 다른 설계들일 수 있음 -> 정상적인 결과일 수 있음

## MCP 도구 레퍼런스

| 도구명 | 설명 | 주요 파라미터 |
|--------|------|--------------|
| `inspect_structure` | CIF 파일의 구조 정보를 조회 (chain 목록, 잔기 수, Ca 원자 수 등) | `file_path` |
| `superimpose_group` | 디렉토리 내 CIF 파일을 design_id 그룹별로 정렬 | `input_dir`, `output_dir`, `chain`, `reference_model` |
| `superimpose_all` | 최상위 폴더 하위 모든 CIF 파일을 단일 reference에 정렬 | `input_root`, `output_root`, `chain`, `reference` |

## 주의사항

- 출력 디렉토리가 없으면 자동 생성됨 (별도 확인 불필요)
- pLDDT 등 원본 CIF 메타데이터는 gemmi를 통해 자동 보존됨
- CDR 길이가 다른 설계들 간에도 잔기 번호 기준 공통 Ca로 정렬됨 (mode=all)
- 파일이 많을 경우(>10,000) 실행 시간이 길 수 있음 — 사용자에게 미리 안내
- MCP 서버는 별도 설치가 필요함 — 이 스킬은 워크플로 가이드만 제공
