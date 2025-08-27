# Packer 코드베이스 문서

이 디렉토리는 HashiCorp Packer 프로젝트의 구조와 아키텍처를 설명하는 상세 문서를 포함합니다.

## 📚 문서 목록

### 1. [프로젝트 구조](project-structure.md)
- 디렉토리 구조 상세 설명
- 각 패키지와 파일의 역할
- 주요 컴포넌트 위치
- 개발 관련 디렉토리 가이드

### 2. [아키텍처 개요](architecture-overview.md)
- Packer의 전체 아키텍처 철학
- 주요 아키텍처 패턴 (플러그인, RPC, Hook)
- 실행 생명주기
- HCP 통합
- 성능 최적화 전략

### 3. [핵심 개념](core-concepts.md)
- Builder, Provisioner, Post-Processor 설명
- Artifact, Communicator, Template 개념
- Hook 시스템과 Data Source
- 병렬 빌드와 민감한 값 처리

### 4. [플러그인 시스템](plugin-system.md)
- 플러그인 아키텍처 상세
- 플러그인 개발 가이드
- RPC 통신 프로토콜
- 플러그인 배포와 설치

## 🎯 독자 대상

이 문서는 다음과 같은 사람들을 위해 작성되었습니다:

- **Packer 코드베이스를 학습하려는 개발자**
- **Packer 플러그인을 개발하려는 엔지니어**
- **Packer에 기여하려는 오픈소스 컨트리뷰터**
- **Packer의 내부 동작을 이해하려는 DevOps 엔지니어**

## 📖 읽기 순서 추천

### 초급자 (Packer 처음 접하는 경우)
1. **핵심 개념** - 기본 용어와 개념 이해
2. **프로젝트 구조** - 코드 위치 파악
3. **아키텍처 개요** - 전체 그림 이해
4. **플러그인 시스템** - 확장성 이해

### 중급자 (Packer 사용 경험 있음)
1. **아키텍처 개요** - 내부 동작 이해
2. **플러그인 시스템** - 커스텀 플러그인 개발
3. **프로젝트 구조** - 코드 네비게이션
4. **핵심 개념** - 깊은 이해

### 고급자 (코드 기여 목적)
1. **프로젝트 구조** - 코드 구조 파악
2. **플러그인 시스템** - 인터페이스 이해
3. **아키텍처 개요** - 설계 결정 이해
4. **핵심 개념** - 구현 디테일

## 🗺️ 관련 다이어그램

아키텍처 다이어그램은 [`../cns-diagram/`](../cns-diagram/) 디렉토리에서 확인할 수 있습니다:

- `architecture-overview.mmd` - 전체 구조
- `directory-structure.mmd` - 디렉토리 계층
- `build-workflow.mmd` - 빌드 워크플로우
- `core-interfaces.mmd` - 인터페이스 관계
- `execution-flow.mmd` - 실행 흐름
- `plugin-architecture.mmd` - 플러그인 구조

## 🔧 문서 업데이트

이 문서들은 Packer 코드베이스를 분석하여 작성되었습니다. 코드가 변경되면 문서도 업데이트가 필요할 수 있습니다.

### 기여 방법
1. 문서에서 오류나 개선점 발견 시 Issue 생성
2. Pull Request로 직접 수정 제안
3. 새로운 섹션이나 문서 추가 제안

## 📞 추가 리소스

- [Packer 공식 문서](https://developer.hashicorp.com/packer/docs)
- [Packer GitHub 저장소](https://github.com/hashicorp/packer)
- [Packer Plugin SDK](https://github.com/hashicorp/packer-plugin-sdk)
- [HCL2 문서](https://github.com/hashicorp/hcl)

## ⚖️ 라이선스

이 문서는 Packer 프로젝트의 일부로 BUSL-1.1 라이선스를 따릅니다.