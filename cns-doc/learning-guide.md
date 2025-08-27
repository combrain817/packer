# Packer 코드베이스 학습 가이드

## 🎯 개요

이 가이드는 Packer 코드베이스를 처음 접하는 개발자가 효과적으로 프로젝트를 이해하고 학습할 수 있도록 체계적인 학습 경로를 제공합니다. 기존 분석 문서와 코드를 기반으로 단계별 학습 방법을 안내합니다.

## 📚 권장 학습 경로

### 1단계: 문서로 개념 파악 (1일)

Packer의 핵심 개념과 구조를 이해하는 것부터 시작합니다.

#### 학습 순서
1. **`cns-doc/core-concepts.md`**
   - Builder, Provisioner, Post-Processor 등 핵심 개념 이해
   - Artifact, Communicator, Template 개념 학습
   - Hook 시스템과 Data Source 파악

2. **`cns-doc/project-structure.md`**
   - 디렉토리 구조와 각 패키지의 역할 이해
   - 주요 파일들의 위치와 기능 파악
   - 개발 관련 디렉토리 구조 학습

3. **`cns-diagram/architecture-overview.mmd`**
   - 전체 아키텍처 흐름을 시각적으로 이해
   - 컴포넌트 간 상호작용 파악
   - 플러그인 시스템 구조 이해

#### 학습 목표
- Packer의 기본 용어와 개념 숙지
- 프로젝트 전체 구조에 대한 큰 그림 이해
- 각 컴포넌트의 역할과 관계 파악

### 2단계: 진입점 분석 (1-2일)

프로그램이 어떻게 시작되고 명령어가 처리되는지 이해합니다.

#### 학습 파일
1. **`main.go`**
   ```go
   // 주요 학습 포인트
   - panicwrap을 통한 에러 처리
   - 로깅 시스템 초기화
   - CLI 초기화 과정
   ```

2. **`commands.go`**
   ```go
   // 주요 학습 포인트
   - CLI 명령어 등록 방식
   - mitchellh/cli 라이브러리 사용법
   - 명령어와 핸들러 매핑
   ```

3. **`command/build.go`**
   ```go
   // 주요 학습 포인트
   - BuildCommand 구조체와 메서드
   - 템플릿 파싱과 검증
   - 빌드 실행 흐름
   ```

#### 실습 방법
```bash
# 디버그 모드로 실행하여 흐름 추적
PACKER_LOG=1 packer build -debug template.pkr.hcl
```

### 3단계: 간단한 빌더로 시작 (2일)

가장 단순한 빌더부터 시작하여 Builder 인터페이스 구현을 이해합니다.

#### 학습 순서
1. **`builder/null/`** - Null Builder
   - 실제 리소스를 생성하지 않는 가장 단순한 빌더
   - Builder 인터페이스 구현 패턴 학습
   - 테스트하기 안전한 환경

   ```go
   // 주요 학습 포인트
   type Builder interface {
       Prepare(raws ...interface{}) (warnings, generatedVars []string, err error)
       Run(ctx context.Context, ui Ui, hook Hook) (Artifact, error)
   }
   ```

2. **`builder/file/`** - File Builder
   - 파일 기반의 간단한 빌더
   - 실제 구현 패턴 학습
   - Artifact 생성 과정 이해

#### 실습 예제
```hcl
# null_builder_test.pkr.hcl
source "null" "example" {
  communicator = "none"
}

build {
  sources = ["source.null.example"]
  
  provisioner "shell-local" {
    inline = ["echo 'Hello from Null Builder'"]
  }
}
```

### 4단계: HCL2 템플릿 시스템 (2-3일)

HCL2 템플릿이 어떻게 파싱되고 처리되는지 이해합니다.

#### 핵심 파일
1. **`hcl2template/parser.go`**
   - HCL 파일 파싱 로직
   - 문법 검증과 에러 처리
   - 변수 보간(interpolation) 처리

2. **`hcl2template/types.packer_config.go`**
   ```go
   // PackerConfig 구조체 - 모든 설정의 중심
   type PackerConfig struct {
       Sources        map[SourceRef]SourceBlock
       InputVariables Variables
       LocalVariables Variables
       Builds         Builds
       // ...
   }
   ```

3. **`hcl2template/types.build.go`**
   - Build 블록 처리 로직
   - Source 참조와 프로비저너 체인

#### 학습 포인트
- HCL2 문법과 블록 타입 (source, build, variable, locals, data)
- 변수 처리와 표현식 평가
- 템플릿 검증 프로세스

### 5단계: 코어 실행 엔진 (2-3일)

빌드가 실제로 어떻게 실행되는지 핵심 엔진을 이해합니다.

#### 핵심 파일
1. **`packer/core.go`**
   ```go
   // Core 구조체 - 실행의 중심
   type Core struct {
       Template   *template.Template
       Components ComponentFinder
       Builds     map[string]*BuildConfig
   }
   ```

2. **`packer/build.go`**
   - Build 실행 오케스트레이션
   - 병렬 빌드 처리
   - Hook 시스템 실행

3. **`packer/plugin.go`**
   - 플러그인 검색과 로딩
   - 플러그인 초기화
   - RPC 클라이언트 설정

#### 실행 흐름
```
템플릿 파싱 → Core 초기화 → Build 준비 → Builder.Run() 
→ Provisioners 실행 → Post-Processors 실행 → Artifact 생성
```

### 6단계: 플러그인 시스템 (선택적, 2-3일)

외부 플러그인과의 통신 메커니즘을 이해합니다.

#### 학습 자료
1. **`cns-doc/plugin-system.md`**
   - 플러그인 아키텍처 상세 설명
   - RPC 통신 프로토콜
   - 플러그인 개발 가이드

2. **`packer/plugin_client.go`**
   - go-plugin 라이브러리 사용
   - RPC를 통한 플러그인 통신
   - 플러그인 생명주기 관리

3. **`cns-diagram/plugin-architecture.mmd`**
   - 플러그인 시스템 다이어그램
   - 통신 흐름 시각화

## 💡 학습 팁

### 효과적인 학습 방법

1. **CLAUDE.md 파일 활용**
   - 코드베이스 학습 가이드가 잘 정리되어 있음
   - 프로젝트별 특수 지침 확인

2. **Null Builder부터 시작**
   - 실제 리소스를 생성하지 않아 테스트하기 안전
   - Builder 인터페이스의 최소 구현 확인

3. **다이어그램 활용**
   - `cns-diagram/` 폴더의 mermaid 다이어그램으로 시각적 이해
   - 복잡한 흐름을 그림으로 먼저 파악

4. **디버깅 기법**
   ```bash
   # 상세 로그 활성화
   PACKER_LOG=1 packer build template.pkr.hcl
   
   # 특정 컴포넌트 로그만 보기
   PACKER_LOG=1 PACKER_LOG_PATH=packer.log packer build template.pkr.hcl
   ```

### 코드 읽기 전략

1. **인터페이스 중심으로 이해**
   - Builder, Provisioner, Post-Processor 인터페이스부터 파악
   - 구현체보다 인터페이스 계약을 먼저 이해

2. **실행 흐름 추적**
   - main.go부터 시작하여 함수 호출 체인 따라가기
   - 디버거나 로그를 활용한 실행 경로 확인

3. **테스트 코드 활용**
   - 각 컴포넌트의 테스트 코드로 사용법 학습
   - 엣지 케이스와 에러 처리 방법 파악

## 🧪 실습 환경 구성

### 개발 환경 설정
```bash
# 1. Go 1.23+ 설치 확인
go version

# 2. 저장소 클론
git clone https://github.com/hashicorp/packer.git
cd packer

# 3. 의존성 설치
make install-build-deps
make install-gen-deps

# 4. 개발 빌드
make dev

# 5. 테스트 실행
make test TEST=./builder/null/...
```

### 간단한 테스트 템플릿
```hcl
# learning-test.pkr.hcl
variable "message" {
  type    = string
  default = "Learning Packer"
}

source "null" "learning" {
  communicator = "none"
}

build {
  sources = ["source.null.learning"]
  
  provisioner "shell-local" {
    inline = ["echo '${var.message}'"]
  }
}
```

## 📈 학습 진도 체크리스트

### 기초 단계 ✅
- [ ] 핵심 개념 문서 읽기 완료
- [ ] 프로젝트 구조 파악 완료
- [ ] main.go와 진입점 이해
- [ ] 첫 번째 null 빌드 실행 성공

### 중급 단계 📚
- [ ] Builder 인터페이스 구현 이해
- [ ] HCL2 템플릿 파싱 과정 이해
- [ ] 간단한 커스텀 빌더 작성 가능
- [ ] Provisioner 실행 흐름 파악

### 고급 단계 🚀
- [ "] 플러그인 RPC 통신 이해
- [ ] Hook 시스템 활용 가능
- [ ] 병렬 빌드 메커니즘 이해
- [ ] 코드 기여 가능한 수준

## 🔗 추가 학습 자료

### 내부 문서
- [아키텍처 개요](architecture-overview.md)
- [핵심 개념](core-concepts.md)
- [프로젝트 구조](project-structure.md)
- [플러그인 시스템](plugin-system.md)

### 외부 리소스
- [Packer 공식 문서](https://developer.hashicorp.com/packer/docs)
- [Packer Plugin SDK](https://github.com/hashicorp/packer-plugin-sdk)
- [HCL2 스펙](https://github.com/hashicorp/hcl)
- [Contributing 가이드](../.github/CONTRIBUTING.md)

## 📝 학습 노트 작성 권장

학습 과정에서 다음 사항들을 기록하는 것을 권장합니다:

1. **이해한 개념**: 각 컴포넌트의 역할과 동작 방식
2. **코드 스니펫**: 중요한 패턴이나 구현 예제
3. **질문 사항**: 이해가 안 되는 부분이나 궁금한 점
4. **개선 아이디어**: 코드를 읽으며 떠오른 개선점

## 🤝 도움 받기

- **GitHub Issues**: 코드에 대한 질문이나 버그 리포트
- **Community Forum**: 일반적인 사용법이나 베스트 프랙티스
- **Contributing Guide**: 코드 기여 방법과 규칙

이 가이드를 따라 단계적으로 학습하면, Packer 코드베이스의 전체 구조와 동작 원리를 체계적으로 이해할 수 있습니다.