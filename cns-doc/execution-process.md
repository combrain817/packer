# Packer 실행 과정과 프로세스 관리

## 🎯 개요

이 문서는 Packer의 실행 과정과 프로세스 관리 메커니즘을 상세히 설명합니다. 특히 플러그인 프로세스의 생명주기와 Packer가 어떻게 효율적으로 리소스를 관리하는지를 중점적으로 다룹니다.

## 🏗️ Packer의 기본 아키텍처

### Packer는 단명(Short-lived) CLI 도구

**핵심 특징:**
- **상주 프로세스가 아님**: 평소에는 어떤 프로세스도 실행되지 않음
- **명령 기반 실행**: 사용자가 명령을 실행할 때만 프로세스 생성
- **작업 완료 후 즉시 종료**: 빌드가 끝나면 모든 프로세스 정리
- **데몬/서비스 아님**: systemd나 Windows 서비스로 동작하지 않음

```bash
# Packer 실행 전
$ ps aux | grep packer
# (결과 없음)

# Packer 실행 중
$ packer build template.pkr.hcl &
$ ps aux | grep packer
user  12345  packer
user  12346  packer-plugin-aws
user  12347  packer-plugin-ansible

# Packer 실행 완료 후
$ ps aux | grep packer
# (결과 없음)
```

## 🔄 전체 실행 흐름

### 실행 단계별 상세 분석

```
사용자가 'packer build template.pkr.hcl' 실행
        ↓
[1. 프로세스 시작 - main.go]
    ├→ panicwrap으로 crash 처리 준비
    ├→ 로깅 시스템 초기화
    └→ CLI 명령어 라우팅 (mitchellh/cli)
        ↓
[2. 명령 처리 - command/build.go]
    └→ ParseArgs() → RunContext()
        ↓
[3. 초기화 단계]
    ├→ HCL2 템플릿 파싱
    ├→ 필요한 플러그인 식별
    └→ 플러그인 Discovery
        - ~/.packer.d/plugins/ 스캔
        - 실행 파일 디렉토리 검색
        - 플러그인 바이너리 경로 매핑
        ↓
[4. 준비 단계]
    ├→ 변수 해결 (우선순위 적용)
    ├→ 필요한 플러그인만 프로세스 시작
    │   └→ exec.Command("packer-plugin-xxx")
    ├→ RPC 연결 설정 (포트 10000-25000)
    └→ Builder.Prepare() 호출 (설정 검증)
        ↓
[5. 실행 단계]
    ├→ PreBuild Hook
    ├→ Builder.Run() [RPC 호출]
    │   ├→ 임시 리소스 생성 (VM/컨테이너)
    │   ├→ Provision Hook
    │   ├→ Provisioners 실행 [각각 RPC]
    │   ├→ 이미지 생성 (스냅샷)
    │   └→ 임시 리소스 정리
    ├→ PostBuild Hook
    └→ Post-Processors 실행 [RPC]
        ↓
[6. 정리 단계]
    ├→ CleanupClients() 호출
    ├→ 모든 플러그인 프로세스 Kill
    └→ Packer 메인 프로세스 종료
```

## 🔌 플러그인 프로세스 생명주기

### 플러그인 검색 vs 실행

**중요한 구분:**
- **검색(Discovery)**: 플러그인 바이너리의 위치를 찾는 것
- **실행(Execution)**: 실제로 플러그인 프로세스를 시작하는 것

### 1. 플러그인 검색 (Discovery)

```go
// packer/plugin.go:51
func (c *PluginConfig) Discover() error {
    // 플러그인 설치 경로 검색
    installations, err := plugingetter.Requirement{}.ListInstallations(...)
    
    // 플러그인 경로를 맵에 저장만 함
    for _, install := range installations {
        pluginMap[pluginName] = install.BinaryPath
    }
    // 아직 프로세스는 시작하지 않음!
}
```

**검색 순서:**
1. 명시적 경로 (절대 경로)
2. 실행 파일 디렉토리
3. `~/.packer.d/plugins/`
4. `PACKER_PLUGIN_PATH` 환경 변수
5. 내장 컴포넌트

### 2. 플러그인 실행 (지연 실행)

```go
// packer/plugin_client.go:136
func (c *PluginClient) Builder() (packersdk.Builder, error) {
    client, err := c.Client() // 여기서 처음 프로세스 시작!
    if err != nil {
        return nil, err
    }
    return &cmdBuilder{client.Builder(), c}, nil
}

// packer/plugin_client.go:239
func (c *PluginClient) Start() (net.Addr, error) {
    // 실제 프로세스 시작
    log.Printf("Starting plugin: %s %#v", cmd.Path, cmd.Args)
    err := cmd.Start()
    
    // RPC 서버 주소 대기
    log.Printf("Waiting for RPC address for: %s", cmd.Path)
    // ...
}
```

### 3. 플러그인 통신

```go
// RPC를 통한 통신 설정
env := []string{
    fmt.Sprintf("PACKER_PLUGIN_MIN_PORT=%d", c.config.MinPort),
    fmt.Sprintf("PACKER_PLUGIN_MAX_PORT=%d", c.config.MaxPort),
}

// Unix Socket 또는 Named Pipe를 통한 통신
client := rpc.NewClient(plugin.Stdout, plugin.Stdin)
builder := &RPCBuilder{client: client}
```

### 4. 플러그인 종료

```go
// packer/plugin_client.go:194
func (c *PluginClient) Kill() {
    cmd := c.config.Cmd
    if cmd.Process == nil {
        return
    }
    
    // 프로세스 종료
    cmd.Process.Kill()
    
    // 로그 대기
    <-c.doneLogging
}

// packer/plugin_client.go:78
func CleanupClients() {
    // 모든 관리되는 플러그인 병렬 종료
    var wg sync.WaitGroup
    for _, client := range managedClients {
        wg.Add(1)
        go func(client *PluginClient) {
            client.Kill()
            wg.Done()
        }(client)
    }
    
    log.Println("waiting for all plugin processes to complete...")
    wg.Wait()
}
```

## 📊 프로세스 상태 매트릭스

| 시점 | Packer 메인 | 플러그인 프로세스 | 메모리 사용 | 비고 |
|------|------------|----------------|------------|------|
| **평소** | ❌ | ❌ | 0 MB | 아무것도 실행되지 않음 |
| **packer build 시작** | ✅ | ❌ | ~50 MB | 템플릿 파싱, 플러그인 검색 |
| **빌드 준비** | ✅ | ✅ (필요한 것만) | ~200 MB | 플러그인 프로세스 시작, RPC 연결 |
| **빌드 실행 중** | ✅ | ✅ (활성) | ~500 MB+ | 실제 작업 수행 |
| **빌드 완료** | ❌ | ❌ | 0 MB | 모든 프로세스 정리 |

## 🎯 효율적인 리소스 관리

### 지연 실행 (Lazy Loading)

```hcl
# template.pkr.hcl
source "amazon-ebs" "example" { ... }
source "docker" "example" { ... }  # 정의만 되어 있음

build {
  sources = ["source.amazon-ebs.example"]  # AWS 플러그인만 실행됨
  # Docker 플러그인은 시작되지 않음!
}
```

### 병렬 빌드 시 프로세스 관리

```hcl
build {
  sources = [
    "source.amazon-ebs.example",
    "source.googlecompute.example",
    "source.azure-arm.example"
  ]
  # 3개의 플러그인이 동시에 실행되지만
  # -parallel-builds 옵션으로 제어 가능
}
```

```bash
# 동시 실행 빌드 수 제한
packer build -parallel-builds=2 template.pkr.hcl
```

## 🔍 실제 코드 예시

### 플러그인이 필요할 때만 시작되는 예

```go
// command/build.go에서 빌드 실행
func (c *BuildCommand) RunContext(ctx context.Context, args *BuildArgs) int {
    // 1. 템플릿 파싱 (플러그인 시작 안 함)
    cfg, diags := c.Meta.Core.Init(&packer.InitOptions{})
    
    // 2. 빌드 준비 (여기서 필요한 플러그인만 시작)
    for _, build := range cfg.Builds {
        // build에 정의된 빌더의 플러그인만 시작
        builder, err := c.Meta.Core.GetBuilder(build.Type)
        // GetBuilder() 내부에서 PluginClient.Start() 호출
    }
    
    // 3. 빌드 실행
    artifacts, err := build.Run(ctx, ui)
    
    // 4. 자동 정리 (defer로 등록됨)
    defer CleanupClients()
}
```

## 🛡️ 안정성 보장

### Panic 처리

```go
// main.go:42
func realMain() int {
    var wrapConfig panicwrap.WrapConfig
    
    if !panicwrap.Wrapped(&wrapConfig) {
        // 부모 프로세스: panic 감지 및 처리
        wrapConfig.Handler = panicHandler(logTempFile)
        exitStatus, err := panicwrap.Wrap(&wrapConfig)
    } else {
        // 자식 프로세스: 실제 작업 수행
        return wrappedMain()
    }
}
```

### 플러그인 크래시 격리

```go
// 플러그인이 크래시해도 메인 프로세스는 안전
defer func() {
    if r := recover(); r != nil {
        log.Printf("Plugin crashed: %v", r)
        // 메인 프로세스는 계속 실행
    }
}()
```

## 📋 핵심 요약

### ✅ 올바른 이해
1. **Packer는 상주 프로세스가 아님** - 명령 실행 시에만 동작
2. **플러그인은 지연 실행** - 템플릿에서 사용하는 것만 실행
3. **자동 리소스 정리** - 작업 완료 후 모든 프로세스 종료
4. **격리된 실행** - 각 플러그인은 독립 프로세스로 안전하게 실행

### ❌ 흔한 오해
1. ~~Packer가 백그라운드에서 계속 실행됨~~ → 명령 실행 시에만
2. ~~모든 플러그인이 한 번에 시작됨~~ → 필요한 것만 시작
3. ~~플러그인이 메모리에 상주함~~ → 사용 후 즉시 종료
4. ~~`packer build` 시 모든 플러그인 로드~~ → 템플릿에 정의된 것만

## 🔗 관련 문서

- [아키텍처 개요](architecture-overview.md) - 전체 아키텍처 이해
- [플러그인 시스템](plugin-system.md) - 플러그인 상세 메커니즘
- [학습 가이드](learning-guide.md) - 코드베이스 학습 경로

## 📚 참고 코드 위치

- `main.go`: 프로그램 진입점과 panicwrap
- `packer/plugin.go`: 플러그인 검색 로직
- `packer/plugin_client.go`: 플러그인 프로세스 관리
- `command/build.go`: 빌드 명령 실행 흐름