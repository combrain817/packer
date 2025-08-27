# Packer 아키텍처 개요

## 🏛️ 아키텍처 철학

Packer는 다음 원칙에 기반하여 설계되었습니다:

1. **플러그인 기반 확장성**: 핵심 기능을 제외한 모든 것은 플러그인
2. **병렬 실행**: 여러 빌더를 동시에 실행하여 효율성 극대화
3. **선언적 구성**: HCL2를 사용한 인프라 as 코드
4. **크로스 플랫폼**: 모든 주요 OS와 클라우드 플랫폼 지원

## 🔗 주요 아키텍처 패턴

### 1. 플러그인 아키텍처

#### 설계 목적
- **격리성**: 플러그인 크래시가 코어에 영향을 주지 않음
- **확장성**: 새로운 플랫폼 쉽게 추가 가능
- **버전 독립성**: 플러그인별 독립적인 버전 관리

#### 구현 방식
```go
// 플러그인은 별도 프로세스로 실행
plugin := exec.Command("packer-plugin-aws")
plugin.Start()

// RPC를 통한 통신
client := rpc.NewClient(plugin.Stdout, plugin.Stdin)
builder := &RPCBuilder{client: client}
```

#### 플러그인 타입
- **Builder Plugins**: 머신 이미지 생성
- **Provisioner Plugins**: 이미지 구성
- **Post-Processor Plugins**: 빌드 후처리
- **Datasource Plugins**: 동적 데이터 제공

### 2. RPC 통신 구조

#### 통신 프로토콜
```
Packer Core <--> Unix Socket/Named Pipe <--> Plugin Process
              [Protocol Buffers/MessagePack]
```

#### 주요 RPC 메서드
```go
// Builder RPC Interface
type BuilderRPC struct {
    Prepare(args PrepareArgs) PrepareResponse
    Run(args RunArgs) RunResponse
    Cancel() error
}

// Provisioner RPC Interface
type ProvisionerRPC struct {
    Prepare(config map[string]interface{}) error
    Provision(ui Ui, comm Communicator) error
}
```

### 3. Hook 시스템

#### Hook Points
```go
const (
    HookProvision = "packer_provision"
    HookCleanup   = "packer_cleanup"
    HookPreBuild  = "packer_pre_build"
    HookPostBuild = "packer_post_build"
)
```

#### Hook 실행 흐름
```
PreBuild Hook → Builder.Run() → Provision Hook 
→ Provisioners → PostBuild Hook → Post-Processors
```

### 4. 템플릿 처리 파이프라인

#### HCL2 파싱 단계
1. **Lexing**: 텍스트 → 토큰
2. **Parsing**: 토큰 → AST (추상 구문 트리)
3. **Evaluation**: AST → PackerConfig
4. **Validation**: 설정 검증

#### 변수 처리
```hcl
# 변수 우선순위 (높은 것부터)
1. 명령줄 -var 플래그
2. -var-file 파일
3. 환경 변수 (PKR_VAR_*)
4. *.auto.pkrvars.hcl 파일
5. 변수 기본값
```

## 🎭 핵심 컴포넌트 상세

### Core Engine

#### Core 구조체
```go
type Core struct {
    Template   *template.Template    // 파싱된 템플릿
    components ComponentFinder       // 컴포넌트 레지스트리
    variables  map[string]string     // 변수 저장소
    builds     map[string]*Builder   // 빌더 인스턴스
}
```

#### CoreBuild 구조체
```go
type CoreBuild struct {
    Type           string
    Builder        Builder
    hooks          map[string][]Hook
    provisioners   []CoreBuildProvisioner
    postProcessors [][]CoreBuildPostProcessor
}
```

### Component Registry

#### 컴포넌트 검색 순서
1. 명시적 경로 (절대 경로)
2. 실행 파일 디렉토리
3. `~/.packer.d/plugins/`
4. `PACKER_PLUGIN_PATH` 환경 변수
5. 내장 컴포넌트

#### 플러그인 네이밍 규칙
```
packer-plugin-{name}     # 멀티 컴포넌트 플러그인
packer-builder-{name}    # 단일 빌더
packer-provisioner-{name}  # 단일 프로비저너
packer-post-processor-{name}  # 단일 후처리기
```

## 🔄 실행 생명주기

### 1. 초기화 단계
```go
// 1. 플러그인 검색
plugins := discover.FindPlugins()

// 2. 플러그인 로드
for _, plugin := range plugins {
    client := startPlugin(plugin)
    registry.Register(client)
}

// 3. 템플릿 파싱
config := parser.Parse(template)
```

### 2. 준비 단계
```go
// 1. 변수 해결
variables := resolveVariables(config)

// 2. 빌더 준비
for _, build := range config.Builds {
    builder := registry.GetBuilder(build.Type)
    warnings, _, err := builder.Prepare(build.Config)
}
```

### 3. 실행 단계
```go
// 1. 빌드 시작
artifact, err := builder.Run(ctx, ui, hook)

// 2. 프로비저닝
for _, provisioner := range build.Provisioners {
    err := provisioner.Provision(ctx, ui, comm)
}

// 3. 후처리
for _, pp := range build.PostProcessors {
    artifact, _, _, err = pp.PostProcess(ctx, ui, artifact)
}
```

## 🌐 HCP Integration

### HCP Packer Registry
```go
type HCPPackerRegistry struct {
    BucketName   string
    BucketLabels map[string]string
    Description  string
}
```

### 메타데이터 수집
```go
// 빌드 시작 시
registry.StartBuild(buildID, metadata)

// 아티팩트 생성 시
registry.RegisterArtifact(artifactID, details)

// 빌드 완료 시
registry.CompleteBuild(buildID, status)
```

## 🔐 보안 고려사항

### 1. 민감 정보 처리
```go
// 로그 필터링
LogSecretFilter.Set(sensitiveVars)

// 환경 변수 마스킹
for _, key := range sensitiveKeys {
    os.Setenv(key, "***")
}
```

### 2. 플러그인 검증
```go
// 체크섬 검증
if !verifyChecksum(plugin, expectedSum) {
    return ErrInvalidPlugin
}

// 서명 검증 (선택적)
if requireSigned && !verifySignature(plugin) {
    return ErrUnsignedPlugin
}
```

## 🚀 성능 최적화

### 1. 병렬 실행
```go
// 세마포어를 사용한 동시 실행 제어
sem := semaphore.NewWeighted(maxParallel)

for _, build := range builds {
    go func(b *CoreBuild) {
        sem.Acquire(ctx, 1)
        defer sem.Release(1)
        b.Run(ctx)
    }(build)
}
```

### 2. 리소스 풀링
```go
// 커넥션 풀
type ConnectionPool struct {
    connections chan *Connection
    maxSize     int
}

// 플러그인 재사용
type PluginCache struct {
    plugins map[string]*Plugin
    mu      sync.RWMutex
}
```

## 📊 모니터링과 로깅

### 로그 레벨
```bash
PACKER_LOG=1      # 디버그 로깅 활성화
PACKER_LOG_PATH=  # 로그 파일 경로
```

### 텔레메트리
```go
// Checkpoint를 통한 익명 사용 통계
if !config.DisableCheckpoint {
    go checkpoint.Check(&checkpoint.CheckParams{
        Product: "packer",
        Version: version.Version,
    })
}
```

## 🔮 향후 발전 방향

1. **gRPC 완전 전환**: 현재 net/rpc에서 gRPC로 마이그레이션 중
2. **DAG 기반 실행**: 의존성 기반 병렬 실행 최적화
3. **클라우드 네이티브**: Kubernetes, 컨테이너 환경 최적화
4. **AI/ML 통합**: 이미지 최적화를 위한 머신러닝 활용