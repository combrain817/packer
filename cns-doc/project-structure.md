# Packer 프로젝트 구조 상세 가이드

## 📌 개요

HashiCorp Packer는 Go 언어로 작성된 오픈소스 도구로, 여러 플랫폼에서 동일한 머신 이미지를 자동으로 생성합니다. 이 문서는 Packer 코드베이스의 구조와 각 컴포넌트의 역할을 상세히 설명합니다.

## 🗂️ 프로젝트 디렉토리 구조

```
packer/
├── main.go                    # 프로그램 진입점
├── commands.go               # CLI 명령어 등록
├── config.go                 # 전역 설정 관리
├── checkpoint.go             # 버전 체크 기능
├── panic.go                  # 패닉 처리
│
├── command/                  # CLI 명령어 구현
├── hcl2template/            # HCL2 템플릿 처리
├── packer/                  # 코어 실행 엔진
├── internal/                # 내부 패키지
├── builder/                 # 내장 빌더
├── provisioner/            # 내장 프로비저너
├── post-processor/         # 내장 후처리기
├── datasource/             # 데이터 소스
├── fix/                    # 템플릿 자동 수정
├── acctest/                # 수용 테스트
├── version/                # 버전 정보
└── website/                # 문서 소스
```

## 🎯 핵심 컴포넌트

### 1. 진입점과 CLI (`/` 루트)

#### `main.go`
- **역할**: 프로그램의 진입점
- **주요 기능**:
  - `panicwrap`을 사용한 크래시 처리
  - 로깅 설정
  - CLI 초기화
- **핵심 함수**:
  ```go
  func main()         // 진입점
  func realMain()     // 실제 메인 로직
  func wrappedMain()  // panicwrap 내부 실행
  ```

#### `commands.go`
- **역할**: 사용 가능한 모든 CLI 명령어 정의
- **등록된 명령어**:
  - `build`: 이미지 빌드
  - `validate`: 템플릿 검증
  - `fmt`: 템플릿 포매팅
  - `init`: 플러그인 초기화
  - `console`: 대화형 콘솔
  - `plugins`: 플러그인 관리

### 2. 명령어 구현 (`command/`)

각 파일은 하나의 CLI 명령어를 구현합니다:

#### `command/build.go`
- **BuildCommand 구조체**: `packer build` 명령 구현
- **주요 메서드**:
  - `Run()`: 명령 실행
  - `ParseArgs()`: 인자 파싱
  - `RunContext()`: 빌드 컨텍스트 실행

#### `command/validate.go`
- **ValidateCommand**: 템플릿 유효성 검사
- HCL 문법 검증
- 변수와 빌더 설정 확인

### 3. HCL2 템플릿 시스템 (`hcl2template/`)

HCL2(HashiCorp Configuration Language v2) 템플릿 파싱과 처리를 담당합니다.

#### 핵심 파일들

##### `types.packer_config.go`
```go
type PackerConfig struct {
    Packer struct {
        VersionConstraints []VersionConstraint
        RequiredPlugins    []*RequiredPlugins
    }
    Basedir            string
    Sources            map[SourceRef]SourceBlock
    InputVariables     Variables
    LocalVariables     Variables
    Datasources        Datasources
    Builds             Builds
    HCPPackerRegistry  *HCPPackerRegistryBlock
}
```

##### `parser.go`
- **Parser 구조체**: HCL 파일 파싱
- **주요 메서드**:
  - `Parse()`: HCL 파일을 PackerConfig로 변환
  - `getBuilds()`: Build 블록 추출
  - `getSources()`: Source 블록 추출

##### `types.build.go`
- Build 블록 정의와 처리
- 빌더, 프로비저너, 후처리기 연결

##### `types.variables.go`
- 변수 처리 (input variables, local variables)
- 변수 보간(interpolation) 로직

### 4. 코어 실행 엔진 (`packer/`)

Packer의 핵심 실행 로직을 포함합니다.

#### `core.go`
```go
type Core struct {
    Template   *template.Template
    components ComponentFinder
    variables  map[string]string
    builds     map[string]*template.Builder
}

type CoreConfig struct {
    Components         ComponentFinder
    Template           *template.Template
    Variables          map[string]string
    SensitiveVariables []string
}
```

#### `build.go`
```go
type CoreBuild struct {
    Type           string
    Builder        Builder
    builderConfig  interface{}
    builderType    string
    hooks          map[string][]Hook
    postProcessors [][]CoreBuildPostProcessor
    provisioners   []CoreBuildProvisioner
}
```
- 빌드 오케스트레이션
- Hook 시스템 관리
- 프로비저너 실행 순서 제어

#### `plugin.go`, `plugin_client.go`
- 플러그인 검색과 로딩
- RPC 클라이언트/서버 설정
- 플러그인 생명주기 관리

### 5. 내장 컴포넌트

#### Builders (`builder/`)

##### `null/`
- 테스트용 빌더
- 실제 리소스를 생성하지 않음
- Builder 인터페이스 구현 예제

##### `file/`
- 파일 기반 아티팩트 생성
- 간단한 빌더 구현 예제

#### Provisioners (`provisioner/`)

##### `shell/`
```go
type Config struct {
    common.PackerConfig
    InlineShebang    string
    Script           string
    Scripts          []string
    Inline           []string
    ExecuteCommand   string
    RemotePath       string
    Env              []string
    EnvVarFormat     string
}
```
- 셸 스크립트 실행
- Unix/Linux 시스템용

##### `powershell/`
- Windows PowerShell 스크립트 실행
- Windows 전용 프로비저너

##### `file/`
- 파일/디렉토리 전송
- 소스에서 대상으로 복사

#### Post-Processors (`post-processor/`)

##### `compress/`
- 빌드 아티팩트 압축
- 여러 압축 포맷 지원

##### `checksum/`
- 체크섬 파일 생성
- 무결성 검증용

##### `manifest/`
- 빌드 정보 기록
- JSON 형태로 저장

### 6. 데이터 소스 (`datasource/`)

동적 데이터를 템플릿에서 사용할 수 있게 합니다.

#### `http/`
- HTTP/HTTPS 요청으로 데이터 가져오기
- REST API 통합

#### `hcp-packer-*`
- HCP Packer Registry 통합
- 이미지 메타데이터 조회

### 7. 내부 패키지 (`internal/`)

#### `hcp/`
- HashiCorp Cloud Platform 통합
- Registry API 클라이언트
- 이미지 메타데이터 관리

#### `dag/`
- 방향성 비순환 그래프 구현
- 의존성 해결
- 병렬 실행 최적화

### 8. 플러그인 시스템 (`packer/plugin-getter/`)

#### 플러그인 검색 경로
1. `~/.packer.d/plugins/` - 사용자 플러그인
2. `./packer_cache/` - 프로젝트 캐시
3. 내장 플러그인

#### 플러그인 통신
- **프로토콜**: gRPC/Protocol Buffers
- **전송**: Unix Socket (Linux/Mac), Named Pipe (Windows)
- **인터페이스**: Builder, Provisioner, PostProcessor, Datasource

## 🔄 실행 흐름

### 1. 템플릿 파싱 단계
```
사용자 명령 → CLI 파싱 → HCL2 파싱 → PackerConfig 생성
```

### 2. 검증 단계
```
변수 확인 → 플러그인 존재 확인 → 의존성 검증
```

### 3. 빌드 실행 단계
```
Builder.Prepare() → Builder.Run() → Artifact 생성
→ Provisioners 실행 → Post-Processors 실행
```

## 🧩 인터페이스 구조

### Builder 인터페이스
```go
type Builder interface {
    Prepare(...) (warnings []string, generatedVars []string, err error)
    Run(ctx context.Context, ui Ui, hook Hook) (Artifact, error)
}
```

### Provisioner 인터페이스
```go
type Provisioner interface {
    Prepare(...) error
    Provision(ctx context.Context, ui Ui, comm Communicator, 
              generatedData map[string]interface{}) error
}
```

### PostProcessor 인터페이스
```go
type PostProcessor interface {
    Configure(raws ...interface{}) error
    PostProcess(ctx context.Context, ui Ui, artifact Artifact) (Artifact, bool, bool, error)
}
```

## 🔧 개발 관련 디렉토리

### `fix/`
- 레거시 템플릿 자동 수정
- JSON → HCL2 마이그레이션 지원

### `acctest/`
- 수용 테스트 프레임워크
- 실제 클라우드 리소스 테스트

### `version/`
- 버전 정보 관리
- 빌드 시 버전 임베딩

### `website/`
- 공식 문서 소스
- MDX 형식
- Next.js 기반

## 📝 설정 파일들

### `.golangci.yml`
- Go 린터 설정
- 코드 품질 규칙

### `go.mod`
- Go 모듈 의존성
- 최소 Go 버전: 1.23.0

### `Makefile`
- 빌드 타겟
- 테스트 명령어
- 개발 도구

## 🎓 학습 추천 경로

1. **기초 이해**
   - `main.go` → `commands.go` → `command/build.go`

2. **템플릿 시스템**
   - `hcl2template/parser.go` → `types.packer_config.go`

3. **코어 엔진**
   - `packer/core.go` → `packer/build.go`

4. **컴포넌트 구현**
   - `builder/null/` → `provisioner/shell/`

5. **플러그인 시스템**
   - `packer/plugin.go` → `packer/plugin_client.go`