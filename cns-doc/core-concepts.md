# Packer 핵심 개념

## 🎯 개요

Packer를 이해하기 위해 반드시 알아야 할 핵심 개념들을 설명합니다. 이 개념들은 코드베이스 전반에 걸쳐 사용되며, Packer의 동작 방식을 이해하는 데 필수적입니다.

## 📦 1. Builder (빌더)

### 정의
Builder는 단일 플랫폼에서 머신 이미지를 생성하는 컴포넌트입니다.

### 인터페이스
```go
type Builder interface {
    // 설정 준비 및 검증
    Prepare(raws ...interface{}) (warnings []string, generatedVars []string, err error)
    
    // 실제 빌드 실행
    Run(ctx context.Context, ui Ui, hook Hook) (Artifact, error)
}
```

### 빌더 유형

#### 1. 클라우드 빌더
- **amazon-ebs**: AWS EC2 AMI 생성
- **googlecompute**: GCP 이미지 생성
- **azure-arm**: Azure 이미지 생성

#### 2. 가상화 빌더
- **virtualbox-iso**: VirtualBox 이미지
- **vmware-iso**: VMware 이미지
- **qemu**: QEMU/KVM 이미지

#### 3. 컨테이너 빌더
- **docker**: Docker 이미지
- **lxc**: LXC 컨테이너

### 빌더 생명주기
```
Prepare() → Validate Config → Allocate Resources 
→ Run() → Create Instance → Return Artifact
```

### 코드 예제
```go
// builder/null/builder.go
type Builder struct {
    config Config
    runner multistep.Runner
}

func (b *Builder) Prepare(raws ...interface{}) ([]string, []string, error) {
    // 설정 디코딩
    err := config.Decode(&b.config, &config.DecodeOpts{
        Interpolate: true,
    }, raws...)
    
    // 설정 검증
    if b.config.Host == "" {
        return nil, nil, errors.New("host must be specified")
    }
    
    return warnings, generatedVars, nil
}

func (b *Builder) Run(ctx context.Context, ui Ui, hook Hook) (Artifact, error) {
    // 단계별 실행
    steps := []multistep.Step{
        &StepConnect{},
        &StepProvision{},
    }
    
    b.runner = &multistep.BasicRunner{Steps: steps}
    b.runner.Run(ctx, state)
    
    return artifact, nil
}
```

## 🛠️ 2. Provisioner (프로비저너)

### 정의
Provisioner는 생성된 머신 이미지에 소프트웨어를 설치하고 구성하는 컴포넌트입니다.

### 인터페이스
```go
type Provisioner interface {
    // 설정 준비
    Prepare(raws ...interface{}) error
    
    // 프로비저닝 실행
    Provision(ctx context.Context, ui Ui, comm Communicator, 
              generatedData map[string]interface{}) error
}
```

### 프로비저너 유형

#### 1. 스크립트 기반
- **shell**: Linux/Unix 셸 스크립트
- **powershell**: Windows PowerShell
- **windows-shell**: Windows CMD

#### 2. 구성 관리 도구
- **ansible**: Ansible 플레이북
- **chef**: Chef 레시피
- **puppet**: Puppet 매니페스트

#### 3. 파일 전송
- **file**: 파일/디렉토리 복사
- **shell-local**: 로컬 스크립트 실행

### 프로비저너 실행 순서
```hcl
build {
  sources = ["source.amazon-ebs.example"]
  
  provisioner "shell" {
    inline = ["apt-get update"]  # 1번째 실행
  }
  
  provisioner "file" {
    source = "app.tar.gz"        # 2번째 실행
    destination = "/tmp/"
  }
  
  provisioner "shell" {
    script = "install.sh"        # 3번째 실행
  }
}
```

## 📮 3. Post-Processor (후처리기)

### 정의
Post-Processor는 빌드 완료 후 아티팩트를 처리하는 컴포넌트입니다.

### 인터페이스
```go
type PostProcessor interface {
    // 설정
    Configure(raws ...interface{}) error
    
    // 후처리 실행
    PostProcess(ctx context.Context, ui Ui, artifact Artifact) (Artifact, bool, bool, error)
    // 반환값: (새 아티팩트, keep input, force continue, error)
}
```

### 후처리기 체인
```hcl
post-processors {
  # 체인 1
  post-processor "compress" {    # 압축
    output = "image.tar.gz"
  }
  
  post-processor "upload" {      # 업로드
    destination = "s3://bucket/"
  }
  
  # 체인 2 (병렬 실행)
  post-processor "manifest" {    # 매니페스트 생성
    output = "manifest.json"
  }
}
```

## 🗃️ 4. Artifact (아티팩트)

### 정의
Artifact는 빌드의 결과물을 나타내는 인터페이스입니다.

### 인터페이스
```go
type Artifact interface {
    BuilderId() string           // 빌더 ID
    Files() []string             // 생성된 파일 목록
    Id() string                  // 아티팩트 ID
    String() string              // 사람이 읽을 수 있는 설명
    State(name string) interface{} // 상태 정보
    Destroy() error              // 아티팩트 삭제
}
```

### 아티팩트 예제
```go
// AWS AMI Artifact
type Artifact struct {
    Amis  map[string]string  // region -> AMI ID
    BuilderIdValue string
    StateData map[string]interface{}
}

func (a *Artifact) String() string {
    return fmt.Sprintf("AMIs were created:\n%s", a.Amis)
}
```

## 🔌 5. Communicator (통신자)

### 정의
Communicator는 원격 머신과 통신하는 추상화 인터페이스입니다.

### 인터페이스
```go
type Communicator interface {
    Start(ctx context.Context, cmd *RemoteCmd) error
    Upload(path string, r io.Reader, fi *os.FileInfo) error
    UploadDir(dst string, src string, exclude []string) error
    Download(path string, w io.Writer) error
    DownloadDir(src string, dst string, exclude []string) error
}
```

### 통신자 유형
- **SSH**: Linux/Unix 시스템
- **WinRM**: Windows 시스템
- **Docker**: Docker 컨테이너
- **None**: 통신 없음 (로컬 빌드)

## 📋 6. Template (템플릿)

### HCL2 템플릿 구조
```hcl
# 변수 정의
variable "ami_name" {
  type    = string
  default = "my-custom-ami"
}

# 로컬 변수
locals {
  timestamp = regex_replace(timestamp(), "[- TZ:]", "")
}

# 소스 정의 (재사용 가능)
source "amazon-ebs" "example" {
  ami_name      = var.ami_name
  instance_type = "t2.micro"
  region        = "us-west-2"
  
  source_ami_filter {
    filters = {
      name = "ubuntu/images/*ubuntu-focal-20.04-amd64-server-*"
    }
    owners = ["099720109477"]
  }
}

# 빌드 정의
build {
  sources = ["source.amazon-ebs.example"]
  
  provisioner "shell" {
    inline = ["echo 'Hello, Packer!'"]
  }
  
  post-processor "compress" {
    output = "{{.BuildName}}.tar.gz"
  }
}
```

### 템플릿 컴포넌트
1. **Variables**: 입력 변수
2. **Locals**: 로컬 변수
3. **Sources**: 빌더 설정
4. **Build**: 빌드 워크플로우
5. **Data Sources**: 외부 데이터

## 🪝 7. Hook (훅)

### 정의
Hook은 빌드 프로세스의 특정 지점에서 실행되는 콜백입니다.

### Hook 인터페이스
```go
type Hook interface {
    Run(ctx context.Context, name string, ui Ui, comm Communicator, 
        data interface{}) error
    Cancel()
}
```

### 주요 Hook Points
```go
const (
    HookProvision    = "packer_provision"
    HookCleanup      = "packer_cleanup"
    HookPrepare      = "packer_prepare"
    HookBuild        = "packer_build"
)
```

## 📊 8. Data Source (데이터 소스)

### 정의
Data Source는 템플릿에서 사용할 수 있는 외부 데이터를 제공합니다.

### 데이터 소스 예제
```hcl
# HTTP 데이터 소스
data "http" "example" {
  url = "https://api.example.com/ami-id"
}

# HCP Packer 데이터 소스
data "hcp-packer-iteration" "ubuntu" {
  bucket_name = "my-ubuntu-base"
  channel     = "production"
}

# 데이터 사용
source "amazon-ebs" "example" {
  source_ami = data.hcp-packer-iteration.ubuntu.source_ami
}
```

## 🔄 9. Parallel Builds (병렬 빌드)

### 개념
여러 빌더를 동시에 실행하여 빌드 시간을 단축합니다.

### 구현
```hcl
build {
  sources = [
    "source.amazon-ebs.ubuntu",    # 병렬 실행
    "source.googlecompute.ubuntu", # 병렬 실행
    "source.azure-arm.ubuntu"      # 병렬 실행
  ]
}
```

### 제어
```bash
# 최대 병렬 수 제한
packer build -parallel-builds=2 template.pkr.hcl
```

## 🎭 10. Context (컨텍스트)

### Build Context
```go
type BuildContext struct {
    BuildName     string
    BuildType     string
    TemplatePath  string
    UserVariables map[string]string
    SharedState   map[string]interface{}
}
```

### 사용 가능한 컨텍스트 변수
```hcl
provisioner "shell" {
  inline = [
    "echo Build: ${build.name}",
    "echo Type: ${build.type}",
    "echo Path: ${path.root}",
    "echo Var: ${var.my_var}",
    "echo Time: ${local.timestamp}"
  ]
}
```

## 🔐 11. Sensitive Values (민감한 값)

### 정의
로그에 노출되지 않아야 하는 민감한 정보를 관리합니다.

### 사용 방법
```hcl
variable "password" {
  type      = string
  sensitive = true  # 로그에서 마스킹
}

locals {
  api_key = sensitive(data.vault.secret.api_key)
}
```

## 📈 12. HCP Packer Registry

### 개념
HashiCorp Cloud Platform에서 이미지 메타데이터를 중앙 관리합니다.

### 통합
```hcl
packer {
  required_plugins {
    amazon = {
      version = ">= 1.0.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

build {
  hcp_packer_registry {
    bucket_name = "my-app-images"
    description = "Base images for my application"
    
    bucket_labels = {
      "team"        = "platform"
      "environment" = "production"
    }
  }
  
  sources = ["source.amazon-ebs.base"]
}
```

## 🎓 학습 팁

1. **시작하기**: `null` 빌더로 기본 개념 이해
2. **실습하기**: 간단한 템플릿 작성 및 실행
3. **디버깅**: `PACKER_LOG=1`로 상세 로그 확인
4. **확장하기**: 커스텀 플러그인 작성 시도