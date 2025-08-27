# Packer 플러그인 시스템 상세 가이드

## 🔌 개요

Packer의 플러그인 시스템은 코어 기능과 확장 기능을 분리하여 유연하고 확장 가능한 아키텍처를 제공합니다. 각 플러그인은 독립된 프로세스로 실행되며 RPC를 통해 통신합니다.

## 🏗️ 플러그인 아키텍처

### 설계 원칙

1. **프로세스 격리**: 플러그인 크래시가 코어에 영향 없음
2. **언어 독립성**: Go 외 언어로도 플러그인 작성 가능 (이론적)
3. **버전 독립성**: 플러그인별 독립적 버전 관리
4. **동적 로딩**: 런타임에 플러그인 검색 및 로드

### 아키텍처 다이어그램
```
┌─────────────────┐         ┌──────────────────┐
│   Packer Core   │   RPC   │   Plugin Process │
│                 │◄────────►│                  │
│  - CLI Handler  │  Socket  │  - Builder       │
│  - Template     │   or     │  - Provisioner   │
│  - Orchestrator │   Pipe   │  - PostProcessor │
└─────────────────┘         └──────────────────┘
```

## 📦 플러그인 타입

### 1. Single-Component Plugins
단일 컴포넌트만 제공하는 플러그인

```go
// packer-builder-custom/main.go
package main

import (
    "github.com/hashicorp/packer-plugin-sdk/plugin"
)

func main() {
    pps := plugin.NewSet()
    pps.RegisterBuilder("custom", new(Builder))
    err := pps.Run()
}
```

### 2. Multi-Component Plugins
여러 컴포넌트를 제공하는 플러그인

```go
// packer-plugin-custom/main.go
func main() {
    pps := plugin.NewSet()
    pps.RegisterBuilder("instance", new(InstanceBuilder))
    pps.RegisterBuilder("volume", new(VolumeBuilder))
    pps.RegisterProvisioner("configure", new(ConfigProvisioner))
    pps.RegisterPostProcessor("upload", new(UploadPostProcessor))
    pps.RegisterDatasource("metadata", new(MetadataDatasource))
    err := pps.Run()
}
```

## 🔍 플러그인 검색 메커니즘

### 검색 경로 우선순위

```go
// packer/plugin_folders.go
func PluginFolders(dirs ...string) []string {
    var paths []string
    
    // 1. 명시적 경로
    paths = append(paths, dirs...)
    
    // 2. 현재 실행 파일 디렉토리
    paths = append(paths, filepath.Dir(os.Args[0]))
    
    // 3. 사용자 플러그인 디렉토리
    paths = append(paths, "~/.packer.d/plugins")
    
    // 4. 환경 변수
    if path := os.Getenv("PACKER_PLUGIN_PATH"); path != "" {
        paths = append(paths, filepath.SplitList(path)...)
    }
    
    return paths
}
```

### 플러그인 네이밍 규칙

```
# Single-component
packer-builder-{name}
packer-provisioner-{name}
packer-post-processor-{name}

# Multi-component
packer-plugin-{name}

# 버전 포함
packer-plugin-{name}_v{version}_{os}_{arch}
예: packer-plugin-amazon_v1.0.0_linux_amd64
```

### 플러그인 검색 코드

```go
// packer/plugin_discover.go
func (c *PluginConfig) Discover() error {
    for _, path := range c.PluginDirectories {
        err := c.discoverPath(path)
        if err != nil {
            log.Printf("Error discovering plugins in %s: %s", path, err)
        }
    }
    return nil
}

func (c *PluginConfig) discoverPath(path string) error {
    var paths []string
    
    err := filepath.Walk(path, func(path string, info os.FileInfo, err error) error {
        if strings.HasPrefix(filepath.Base(path), "packer-") {
            paths = append(paths, path)
        }
        return nil
    })
    
    for _, path := range paths {
        c.discoverPlugin(path)
    }
    
    return err
}
```

## 🔐 플러그인 통신 프로토콜

### RPC 초기화

```go
// packer/plugin_client.go
type PluginClient struct {
    config  *ClientConfig
    process *os.Process
    client  *rpc.Client
}

func NewClient(config *ClientConfig) *PluginClient {
    cmd := exec.Command(config.Cmd, config.Args...)
    
    // 표준 입출력을 통한 RPC 연결
    stdin, _ := cmd.StdinPipe()
    stdout, _ := cmd.StdoutPipe()
    
    cmd.Start()
    
    // RPC 클라이언트 생성
    client := rpc.NewClient(stdout, stdin)
    
    return &PluginClient{
        config:  config,
        process: cmd.Process,
        client:  client,
    }
}
```

### 메시지 프로토콜

```go
// RPC 메시지 구조
type Message struct {
    Type   MessageType
    Id     uint32
    Method string
    Args   interface{}
}

// 메시지 타입
const (
    MessageTypeRequest  = 0
    MessageTypeResponse = 1
    MessageTypeError    = 2
)
```

### 핸드셰이크 프로토콜

```go
// 플러그인 시작 시 핸드셰이크
type HandshakeConfig struct {
    ProtocolVersion  uint
    MagicCookieKey   string
    MagicCookieValue string
}

var Handshake = HandshakeConfig{
    ProtocolVersion:  1,
    MagicCookieKey:   "PACKER_PLUGIN_MAGIC_COOKIE",
    MagicCookieValue: "d602bf8f470bc67ca7faa0386276bbdd4330efaf76d1a219cb4d6991ca9872b2",
}
```

## 🛠️ 플러그인 개발

### 플러그인 인터페이스 구현

#### Builder 플러그인
```go
package main

import (
    "context"
    "github.com/hashicorp/packer-plugin-sdk/packer"
)

type MyBuilder struct {
    config Config
}

func (b *MyBuilder) ConfigSpec() hcldec.ObjectSpec {
    return b.config.FlatMapstructure().HCL2Spec()
}

func (b *MyBuilder) Prepare(raws ...interface{}) ([]string, []string, error) {
    err := config.Decode(&b.config, nil, raws...)
    if err != nil {
        return nil, nil, err
    }
    
    // 설정 검증
    var errs *packer.MultiError
    if b.config.Host == "" {
        errs = packer.MultiErrorAppend(errs, errors.New("host is required"))
    }
    
    if errs != nil && len(errs.Errors) > 0 {
        return nil, nil, errs
    }
    
    return nil, nil, nil
}

func (b *MyBuilder) Run(ctx context.Context, ui packer.Ui, hook packer.Hook) (packer.Artifact, error) {
    // 빌드 로직 구현
    steps := []multistep.Step{
        &stepConnect{},
        &stepCreateInstance{},
        &stepProvision{},
    }
    
    runner := multistep.BasicRunner{Steps: steps}
    runner.Run(ctx, state)
    
    return &Artifact{
        BuilderId: b.config.PackerBuildName,
        Files:     []string{outputPath},
    }, nil
}
```

#### Provisioner 플러그인
```go
type MyProvisioner struct {
    config Config
}

func (p *MyProvisioner) ConfigSpec() hcldec.ObjectSpec {
    return p.config.FlatMapstructure().HCL2Spec()
}

func (p *MyProvisioner) Prepare(raws ...interface{}) error {
    err := config.Decode(&p.config, nil, raws...)
    return err
}

func (p *MyProvisioner) Provision(
    ctx context.Context, 
    ui packer.Ui, 
    comm packer.Communicator,
    generatedData map[string]interface{},
) error {
    ui.Say("Starting provisioning...")
    
    // 스크립트 업로드
    err := comm.Upload("/tmp/script.sh", strings.NewReader(p.config.Script))
    if err != nil {
        return err
    }
    
    // 스크립트 실행
    cmd := &packer.RemoteCmd{
        Command: "bash /tmp/script.sh",
    }
    
    err = comm.Start(ctx, cmd)
    if err != nil {
        return err
    }
    
    cmd.Wait()
    
    if cmd.ExitStatus() != 0 {
        return fmt.Errorf("script failed with exit code %d", cmd.ExitStatus())
    }
    
    return nil
}
```

### 플러그인 설정 구조체

```go
type Config struct {
    common.PackerConfig `mapstructure:",squash"`
    
    // 필수 필드
    Host     string `mapstructure:"host" required:"true"`
    Username string `mapstructure:"username" required:"true"`
    
    // 선택 필드
    Port     int      `mapstructure:"port"`
    Timeout  string   `mapstructure:"timeout"`
    Tags     []string `mapstructure:"tags"`
    
    // 계산된 필드
    ctx interpolate.Context
}

// HCL2 스펙 생성
func (*Config) FlatMapstructure() interface{} { 
    return (*FlatConfig)(nil) 
}
```

## 📥 플러그인 설치

### packer init
```bash
# required_plugins 블록 기반 자동 설치
packer init config.pkr.hcl
```

### 수동 설치
```bash
# GitHub에서 다운로드
wget https://github.com/org/packer-plugin-custom/releases/download/v1.0.0/packer-plugin-custom_v1.0.0_linux_amd64.zip

# 압축 해제
unzip packer-plugin-custom_v1.0.0_linux_amd64.zip -d ~/.packer.d/plugins/

# 실행 권한 부여
chmod +x ~/.packer.d/plugins/packer-plugin-custom_v1.0.0_linux_amd64
```

### required_plugins 설정
```hcl
packer {
  required_plugins {
    custom = {
      version = ">= 1.0.0"
      source  = "github.com/org/custom"
    }
  }
}
```

## 🔧 플러그인 디버깅

### 디버그 모드 실행
```bash
# 플러그인 디버그 모드
PACKER_LOG=1 packer build template.pkr.hcl

# 플러그인 직접 실행 (디버그용)
./packer-plugin-custom -PACKER_PLUGIN_MAGIC_COOKIE="..." -PACKER_PLUGIN_PROTOCOL_VERSIONS="5"
```

### 플러그인 로깅
```go
func (b *MyBuilder) Run(ctx context.Context, ui packer.Ui, hook packer.Hook) (packer.Artifact, error) {
    // UI를 통한 로깅
    ui.Say("Starting build...")
    ui.Message("Debug: Connecting to host")
    ui.Error("Failed to connect")
    
    // 표준 로깅
    log.Printf("[DEBUG] Connection details: %+v", b.config)
    log.Printf("[INFO] Build started")
    log.Printf("[ERROR] Connection failed: %v", err)
    
    return artifact, nil
}
```

## 🏭 플러그인 빌드 시스템

### Makefile 예제
```makefile
NAME=packer-plugin-custom
VERSION=1.0.0

.PHONY: build
build:
	go build -o $(NAME)_v$(VERSION)

.PHONY: install
install: build
	mkdir -p ~/.packer.d/plugins
	cp $(NAME)_v$(VERSION) ~/.packer.d/plugins/

.PHONY: test
test:
	go test ./...

.PHONY: release
release:
	goreleaser release --clean
```

### GoReleaser 설정
```yaml
# .goreleaser.yml
builds:
  - id: packer-plugin-custom
    binary: '{{ .ProjectName }}_v{{ .Version }}_{{ .Os }}_{{ .Arch }}'
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64

archives:
  - format: zip
    files:
      - none*

checksum:
  name_template: '{{ .ProjectName }}_{{ .Version }}_SHA256SUMS'

release:
  github:
    owner: myorg
    name: packer-plugin-custom
```

## 🔒 플러그인 보안

### 체크섬 검증
```go
func verifyChecksum(path string, expected string) error {
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()
    
    h := sha256.New()
    if _, err := io.Copy(h, file); err != nil {
        return err
    }
    
    actual := hex.EncodeToString(h.Sum(nil))
    if actual != expected {
        return fmt.Errorf("checksum mismatch: expected %s, got %s", expected, actual)
    }
    
    return nil
}
```

### 서명 검증
```bash
# GPG 서명 생성
gpg --armor --detach-sign packer-plugin-custom

# 서명 검증
gpg --verify packer-plugin-custom.asc packer-plugin-custom
```

## 🚀 플러그인 배포

### GitHub Releases
```yaml
# GitHub Actions workflow
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-go@v2
      - uses: goreleaser/goreleaser-action@v2
        with:
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Packer Plugin Registry
```hcl
# 레지스트리 메타데이터
packer-plugin {
  version = "1.0.0"
  description = "Custom plugin for Packer"
  
  components {
    builder {
      custom = "Custom Builder"
    }
    provisioner {
      configure = "Configuration Provisioner"
    }
  }
}
```

## 📚 플러그인 예제

### 간단한 로컬 빌더
```go
package main

import (
    "context"
    "github.com/hashicorp/packer-plugin-sdk/packer"
    "github.com/hashicorp/packer-plugin-sdk/plugin"
)

type LocalBuilder struct{}

func (b *LocalBuilder) Prepare(raws ...interface{}) ([]string, []string, error) {
    return nil, nil, nil
}

func (b *LocalBuilder) Run(ctx context.Context, ui packer.Ui, hook packer.Hook) (packer.Artifact, error) {
    ui.Say("Building local artifact...")
    
    return &LocalArtifact{
        files: []string{"/tmp/output.txt"},
    }, nil
}

type LocalArtifact struct {
    files []string
}

func (a *LocalArtifact) BuilderId() string     { return "local.builder" }
func (a *LocalArtifact) Files() []string       { return a.files }
func (a *LocalArtifact) Id() string            { return "local-artifact" }
func (a *LocalArtifact) String() string        { return "Local Artifact" }
func (a *LocalArtifact) State(string) interface{} { return nil }
func (a *LocalArtifact) Destroy() error        { return nil }

func main() {
    pps := plugin.NewSet()
    pps.RegisterBuilder("local", new(LocalBuilder))
    err := pps.Run()
    if err != nil {
        panic(err)
    }
}
```

## 🎓 플러그인 개발 팁

1. **SDK 사용**: packer-plugin-sdk를 활용하여 보일러플레이트 줄이기
2. **테스트 작성**: 단위 테스트와 통합 테스트 모두 작성
3. **문서화**: 각 설정 옵션에 대한 명확한 문서 제공
4. **에러 처리**: 명확하고 도움이 되는 에러 메시지 제공
5. **버전 관리**: Semantic Versioning 준수