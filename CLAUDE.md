# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 🎯 Packer 코드베이스 학습 가이드

HashiCorp Packer는 단일 소스 구성에서 여러 플랫폼용 동일한 머신 이미지를 빌드하는 도구입니다. 이 가이드는 Packer 코드베이스를 효과적으로 이해하기 위한 학습 경로를 제공합니다.

## 📚 학습 로드맵

### 1단계: 진입점과 기본 흐름 이해 (1-2일)

**시작 파일들:**
- `main.go`: 프로그램 진입점, panicwrap 사용법 학습
- `commands.go`: CLI 명령어 등록 방식 이해
- `command/build.go`: 가장 핵심적인 `packer build` 명령 구현

**학습 포인트:**
- CLI 구조 (mitchellh/cli 라이브러리 사용)
- 명령어 라우팅 방식
- panicwrap을 통한 에러 처리

### 2단계: 템플릿 시스템 이해 (2-3일)

**핵심 디렉토리:** `hcl2template/`

**주요 파일들:**
- `hcl2template/parser.go`: HCL 파싱 로직
- `hcl2template/types.packer_config.go`: PackerConfig 구조체 - 모든 설정의 중심
- `hcl2template/types.build.go`: Build 블록 처리
- `hcl2template/types.source.go`: Source 블록 처리

**학습 포인트:**
- HCL2 문법과 파싱
- 변수 보간(interpolation)
- 블록 타입별 처리 (source, build, variable, locals, data)

### 3단계: 코어 실행 엔진 (2-3일)

**핵심 디렉토리:** `packer/`

**주요 파일들:**
- `packer/core.go`: Core 구조체 - 실행의 중심
- `packer/build.go`: Build 실행 로직
- `packer/provisioner.go`: Provisioner 인터페이스
- `packer/plugin.go`: 플러그인 검색과 로딩

**학습 포인트:**
- Builder, Provisioner, Post-processor 인터페이스
- 플러그인 아키텍처 (RPC 기반)
- 빌드 실행 흐름과 Hook 시스템

### 4단계: 내장 컴포넌트 학습 (3-4일)

**간단한 것부터 시작:**

1. **Null Builder** (`builder/null/`): 가장 단순한 빌더
   - 실제 리소스를 생성하지 않음
   - Builder 인터페이스 구현 패턴 학습

2. **File Provisioner** (`provisioner/file/`): 파일 전송 provisioner
   - Provisioner 인터페이스 구현
   - Communicator 사용법

3. **Shell Provisioner** (`provisioner/shell/`): 스크립트 실행
   - 더 복잡한 provisioner 로직
   - 플랫폼별 처리

### 5단계: 플러그인 시스템 심화 (2-3일)

**주요 파일들:**
- `packer/plugin_client.go`: 플러그인 클라이언트
- `packer/cmd_builder.go`: 빌더 플러그인 명령어 래퍼
- `packer/plugin-getter/`: 플러그인 다운로드와 관리

**학습 포인트:**
- go-plugin 라이브러리 사용
- RPC를 통한 플러그인 통신
- 플러그인 검색 메커니즘

### 6단계: 고급 기능 (선택적)

**HCP Integration** (`internal/hcp/`)
- HCP Packer Registry 통합
- 이미지 메타데이터 관리

**Data Sources** (`datasource/`)
- 동적 데이터 소스 구현
- HCP 데이터소스 예제

## 🛠 개발 환경 설정

```bash
# 1. Go 1.23+ 설치 확인
go version

# 2. 저장소 클론
git clone https://github.com/hashicorp/packer.git
cd packer

# 3. 의존성 설치
make install-build-deps
make install-gen-deps

# 4. 개발 빌드 (version/version.go에 prerelease 태그 필요)
make dev

# 5. 테스트 실행
make test TEST=./builder/null/...
```

## 🔍 코드 읽기 팁

### 인터페이스 중심으로 이해하기

Packer는 인터페이스 기반 설계를 따릅니다:

```go
// packer-plugin-sdk/packer/builder.go
type Builder interface {
    Prepare(...) ([]string, []string, error)
    Run(context.Context, Ui, Hook) (Artifact, error)
}

// packer-plugin-sdk/packer/provisioner.go  
type Provisioner interface {
    Prepare(...) error
    Provision(context.Context, Ui, Communicator, map[string]interface{}) error
}
```

### 실행 흐름 추적하기

1. **템플릿 파싱**: HCL → PackerConfig
2. **검증**: 변수 확인, 플러그인 존재 확인
3. **빌드 준비**: Builder.Prepare()
4. **빌드 실행**: Builder.Run()
5. **프로비저닝**: 각 Provisioner.Provision()
6. **후처리**: PostProcessor.PostProcess()

### 디버깅 방법

```bash
# 상세 로그 활성화
PACKER_LOG=1 packer build template.pkr.hcl

# 특정 파일 디버깅
# main.go:154-157 참조 - 로그 레벨 설정
```

## 📖 핵심 개념

### 1. Component 시스템
- **Builder**: 머신 이미지 생성 (AWS AMI, Docker 이미지 등)
- **Provisioner**: 이미지 구성 (소프트웨어 설치, 설정)
- **Post-Processor**: 빌드 후 처리 (압축, 업로드 등)
- **Data Source**: 동적 데이터 조회

### 2. 플러그인 아키텍처
- 각 플러그인은 별도 프로세스로 실행
- RPC(gRPC)를 통한 통신
- 자동 검색: `~/.packer.d/plugins/`

### 3. HCL2 템플릿
- Variables: 입력 변수
- Locals: 로컬 변수
- Sources: 재사용 가능한 빌더 설정
- Builds: 실제 빌드 정의

## 🧪 테스트 작성 및 실행

```bash
# 단위 테스트
make test TEST=./builder/null/...

# 특정 테스트 실행
go test -v -run TestBuilderPrepare ./builder/null/

# 수용 테스트 (실제 리소스 생성)
ACC_TEST_BUILDERS=null make provisioners-acctest TEST=./provisioner/shell
```

## 📝 코드 기여 체크리스트

- [ ] 코드 포맷팅: `make fmt`
- [ ] 코드 생성: `make generate` (구조체 변경 시)
- [ ] 린팅: `make lint`
- [ ] 테스트: `make test TEST=./path/to/package`
- [ ] HCL2 spec 생성: `go generate` (config 구조체 변경 시)

## 🔗 유용한 리소스

- [Packer 공식 문서](https://developer.hashicorp.com/packer/docs)
- [Plugin SDK](https://github.com/hashicorp/packer-plugin-sdk)
- [HCL2 스펙](https://github.com/hashicorp/hcl)
- [Contributing 가이드](.github/CONTRIBUTING.md)