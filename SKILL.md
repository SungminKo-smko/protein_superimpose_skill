---
name: superimpose-protein
description: >
  단백질 CIF 구조 파일들을 지정한 chain의 Ca 원자를 기준으로 superimpose.
  "구조 정렬", "superimpose", "alignment", "CIF 정렬" 키워드가 포함될 때 사용.
  MCP 서버(protein-superimpose)를 통해 두 가지 모드를 지원:
  (1) 단일 디렉토리 내 design_id 그룹별 정렬 (superimpose_group)
  (2) 최상위 폴더 하위 모든 CIF 파일을 단일 reference에 정렬 (superimpose_all)
  원격 서버에서 실행되므로 파일 업로드/다운로드 워크플로를 포함.
---

# Protein Structure Superimposition Skill (MCP)

사용자가 단백질 CIF 구조 파일들의 superimposition을 요청했습니다.
이 스킬은 `protein-superimpose` MCP 서버를 통해 구조 정렬을 수행합니다.
MCP 서버는 Azure Container Apps에서 원격 실행되며, 파일은 업로드/다운로드로 전달합니다.

## Step 1 — 파라미터 파악

아래 정보를 컨텍스트에서 추론하거나, 불명확하면 사용자에게 확인합니다:

| 파라미터 | 설명 | 기본값 |
|---------|------|--------|
| **mode** | `group` = 디렉토리 내 design_id 그룹별 정렬 / `all` = 전체 파일을 단일 reference에 정렬 | 컨텍스트 판단 |
| **input** | 입력 CIF 파일 (로컬 경로 또는 URL) | — |
| **output** | 출력 디렉토리 경로 | input + `_aligned` 자동 생성 |
| **chain** | Superimpose 기준 chain ID | `A` |
| **reference** | (mode=all 전용) 기준 CIF 파일 경로 | 자동 선택 (알파벳 첫 번째) |

**모드 판단 기준:**
- 파일명에 `_model_N.cif` 패턴이 있고 같은 디자인의 여러 예측 모델을 정렬 -> `group` 모드
- 여러 서브폴더에 걸친 모든 구조를 한 기준에 정렬 (h_/m_ 접두사 등) -> `all` 모드

## Step 2 — MCP 서버 확인

`protein-superimpose` MCP 서버가 사용 가능한지 확인합니다.

**필수 도구 목록:**
- `get_upload_urls` — 대량 업로드용 SAS URL 생성
- `sync_uploaded_files` — Blob에서 서버로 동기화
- `upload_file` — 파일 업로드 (base64, 소량 fallback)
- `get_download_urls` — 결과 다운로드 URL 생성
- `download_file` — 파일 다운로드 (base64, 소량 fallback)
- `list_server_files` — 서버 파일 목록 조회
- `inspect_structure` — CIF 구조 검사
- `superimpose_group` — 그룹별 정렬
- `superimpose_all` — 전체 정렬
- `cleanup` — 서버 데이터 정리

**MCP 서버가 없는 경우:**
```
protein-superimpose MCP 서버가 연결되어 있지 않습니다.

원격 서버 (SSE):
  Claude Desktop 설정에 다음을 추가하세요:
  {
    "mcpServers": {
      "protein-superimpose": {
        "url": "https://protein-superimpose-mcp.politebay-55ff119b.westus3.azurecontainerapps.io/sse"
      }
    }
  }

로컬 서버 (stdio):
  pip install protein-superimpose-mcp
  {
    "mcpServers": {
      "protein-superimpose": {
        "command": "protein-superimpose-mcp"
      }
    }
  }
```

## Step 3 — 파일 업로드

MCP 서버는 원격에서 실행되므로, 로컬 CIF 파일을 서버에 업로드해야 합니다.

### 권장 방식: SAS URL 직접 업로드 (대량 파일)

Azure Blob Storage에 SAS URL로 직접 업로드하는 방식입니다. base64 인코딩 없이 바이너리로 전송하므로 빠르고 효율적입니다.

1. `get_upload_urls` MCP 도구를 호출하여 SAS URL을 생성합니다
   - `filenames`: 업로드할 파일명 목록 (예: `["design1.cif", "design2.cif"]`)
   - `subfolder`: (선택) 하위 폴더명으로 파일을 그룹화
   - 반환값: 각 파일에 대한 SAS URL (1시간 유효)
2. 각 SAS URL에 HTTP PUT 요청으로 파일을 직접 업로드합니다
   - `Content-Type: application/octet-stream` 헤더 필요
   - `x-ms-blob-type: BlockBlob` 헤더 필요
3. 모든 파일 업로드 후 `sync_uploaded_files` MCP 도구를 호출하여 Blob에서 서버 로컬로 동기화합니다
   - `subfolder`: 업로드 시 사용한 하위 폴더명
4. `list_server_files`로 동기화된 파일을 확인합니다

```
예시: SAS URL 업로드
  # 1) SAS URL 생성
  get_upload_urls(
    filenames=["design1_model_0.cif", "design1_model_1.cif"],
    subfolder="batch1"
  )
  -> { "urls": { "design1_model_0.cif": "https://...blob.core.windows.net/...?sv=...", ... } }

  # 2) HTTP PUT으로 파일 업로드
  curl -X PUT -H "x-ms-blob-type: BlockBlob" \
    -H "Content-Type: application/octet-stream" \
    --data-binary @design1_model_0.cif \
    "https://...blob.core.windows.net/...?sv=..."

  # 3) Blob → 서버 동기화
  sync_uploaded_files(subfolder="batch1")
  -> 서버 경로: /data/upload/batch1/design1_model_0.cif
```

### Fallback 방식: base64 업로드 (1~2개 소량 파일)

파일이 1~2개인 경우 간편하게 base64 방식을 사용할 수 있습니다.

1. 로컬 CIF 파일을 읽고 base64로 인코딩합니다
2. `upload_file` MCP 도구를 호출하여 서버에 업로드합니다
   - `filename`: 파일명
   - `content_base64`: base64 인코딩된 파일 내용
   - `subfolder`: (선택) 하위 폴더명으로 파일을 그룹화

```
예시: base64 업로드 (소량 fallback)
  upload_file(
    filename="design1_model_0.cif",
    content_base64="<base64 encoded content>",
    subfolder="batch1"
  )
  -> 서버 경로: /data/upload/batch1/design1_model_0.cif
```

**서버 디렉토리 구조:**
- `/data/upload/` — 업로드된 입력 파일
- `/data/output/` — 정렬 결과 파일

## Step 4 — 사전 검증 (Pre-flight Inspection)

정렬 실행 전에 `inspect_structure` MCP 도구로 업로드된 파일을 검증합니다:

1. 대표 CIF 파일 1~2개를 선택하여 `inspect_structure`를 호출합니다
   - 경로는 서버 내 경로를 사용 (예: `/data/upload/batch1/design1_model_0.cif`)
2. 확인 항목:
   - 파일이 유효한 CIF 형식인지
   - 지정한 chain ID가 실제로 존재하는지
   - Ca 원자 수가 정렬에 충분한지 (최소 3개 이상)
3. 문제가 발견되면 사용자에게 보고하고, 파라미터 수정을 제안합니다

## Step 5 — 정렬 실행

### mode = group (superimpose_group)

MCP 도구 `superimpose_group`을 호출합니다:
- `input_dir`: 서버 내 업로드 경로 (예: `/data/upload/batch1`)
- `output_dir`: 서버 내 출력 경로 (예: `/data/output/batch1_aligned`)
- `chain`: 기준 chain ID
- `reference_model`: (선택) 기준 모델 번호

### mode = all (superimpose_all)

MCP 도구 `superimpose_all`을 호출합니다:
- `input_root`: 서버 내 업로드 경로 (예: `/data/upload`)
- `output_root`: 서버 내 출력 경로 (예: `/data/output/aligned`)
- `chain`: 기준 chain ID
- `reference`: (선택) 기준 CIF 파일의 서버 내 경로

## Step 6 — 결과 다운로드 및 검증

### 결과 확인:
1. `list_server_files`로 output 디렉토리의 결과 파일 목록을 확인합니다
2. MCP 도구 반환 결과에서 성공/스킵/오류 건수를 사용자에게 보고합니다

### 권장 방식: SAS URL 직접 다운로드 (대량 파일)

1. `get_download_urls` MCP 도구로 결과 파일의 다운로드 URL을 생성합니다
   - `directory`: 서버 내 결과 디렉토리 경로 (예: `/data/output/aligned`)
   - 반환값: 각 파일에 대한 다운로드 URL (24시간 유효)
2. 각 URL에서 HTTP GET으로 직접 다운로드합니다
3. 필요시 `inspect_structure`로 정렬된 결과 파일을 샘플 검증합니다

```
예시: SAS URL 다운로드
  get_download_urls(directory="/data/output/aligned")
  -> { "urls": { "design1_model_0.cif": "https://...blob.core.windows.net/...?sv=...", ... } }

  # HTTP GET으로 직접 다운로드
  curl -o design1_model_0.cif "https://...blob.core.windows.net/...?sv=..."
```

### Fallback 방식: base64 다운로드 (단일 파일)

파일이 1개인 경우 간편하게 base64 방식을 사용할 수 있습니다.

1. `download_file` MCP 도구로 결과 파일을 다운로드합니다
   - `path`: 서버 내 파일 경로 (예: `/data/output/aligned/design1_model_1.cif`)
   - 반환값: base64 인코딩된 파일 내용
2. base64를 디코딩하여 로컬에 저장합니다

### 정리:
작업 완료 후 `cleanup` 도구로 서버 데이터를 정리합니다:
- `cleanup(directory="output")` — 결과만 삭제
- `cleanup(directory="upload")` — 업로드 파일만 삭제
- `cleanup(directory="all")` — 전체 삭제

## 문제 해결 (Troubleshooting)

### MCP 서버 연결 실패
- Claude Desktop 설정에서 SSE URL이 올바른지 확인
- 서버 URL: `https://protein-superimpose-mcp.politebay-55ff119b.westus3.azurecontainerapps.io/sse`
- scale-to-zero 설정으로 첫 요청 시 콜드 스타트 지연(~30초)이 있을 수 있음

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

### 업로드/다운로드 실패
- 파일이 너무 큰 경우 base64 인코딩 오버헤드로 실패할 수 있음
- 대용량 파일은 SAS URL 방식 사용 권장
- 서버 스토리지 용량 확인: Azure Files 10GB 제한

### SAS URL 관련 문제
- **"SAS URL expired" 오류**: 업로드 URL은 1시간, 다운로드 URL은 24시간 유효. 만료 시 `get_upload_urls` 또는 `get_download_urls`를 다시 호출하여 새 URL 생성
- **"Storage key not configured" 오류**: 서버에 Azure Storage 연결 정보가 설정되지 않음. 서버 관리자에게 `AZURE_STORAGE_CONNECTION_STRING` 환경 변수 설정을 요청
- **PUT 요청 실패 (403/404)**: SAS URL이 만료되었거나, 필수 헤더(`x-ms-blob-type: BlockBlob`)가 누락됨
- **`sync_uploaded_files` 후 파일이 보이지 않음**: Blob 업로드가 완료되기 전에 동기화를 호출한 경우. 모든 PUT 요청 완료 후 동기화 호출 필요

## MCP 도구 레퍼런스

| 도구명 | 설명 | 주요 파라미터 |
|--------|------|--------------|
| `get_upload_urls` | 대량 업로드용 SAS URL 생성 | `filenames`, `subfolder` |
| `sync_uploaded_files` | Blob에서 서버 로컬로 동기화 | `subfolder` |
| `upload_file` | CIF 파일을 서버에 업로드 (base64, 소량 fallback) | `filename`, `content_base64`, `subfolder` |
| `get_download_urls` | 결과 다운로드 URL 생성 (24시간 유효) | `directory` |
| `download_file` | 서버 파일을 base64로 다운로드 (소량 fallback) | `path` |
| `list_server_files` | 서버 데이터 디렉토리 파일 목록 조회 | `directory` |
| `inspect_structure` | CIF 파일의 구조 정보를 조회 (chain, 잔기, Ca 수) | `path` |
| `superimpose_group` | design_id 그룹별 CIF 정렬 | `input_dir`, `output_dir`, `chain`, `reference_model` |
| `superimpose_all` | 전체 CIF를 단일 reference에 정렬 | `input_root`, `output_root`, `chain`, `reference` |
| `cleanup` | 서버 upload/output 디렉토리 정리 | `directory` ("upload", "output", "all") |

## 주의사항

- MCP 서버는 원격(ACA)에서 실행됨 — 로컬 파일 직접 접근 불가, 반드시 업로드 필요
- 출력 디렉토리가 없으면 자동 생성됨 (별도 확인 불필요)
- pLDDT 등 원본 CIF 메타데이터는 gemmi를 통해 자동 보존됨
- CDR 길이가 다른 설계들 간에도 잔기 번호 기준 공통 Ca로 정렬됨 (mode=all)
- 파일이 많을 경우(>10,000) 실행 시간이 길 수 있음 — 사용자에게 미리 안내
- 작업 완료 후 `cleanup`으로 서버 저장 공간을 정리할 것
