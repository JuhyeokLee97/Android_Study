# NowInAndroid Architecture 학습

> 이 글은 Now In Android App의 [Architecture Learning Journey](https://github.com/android/nowinandroid/blob/main/docs/ArchitectureLearningJourney.md) 문서 기반으로 학습한 내용을 정리한 글입니다.

`Architecture Learning Journey`는 Now In Android(이상 NIA) 앱의 전체 아키텍쳐 철학과 구조를 이해할 수 있도록 도와주는 로드맵이다.

## 목표와 요구사항

NIA 앱의 아키텍쳐 목표는 다음과 같다.

- [official architecture guidance](https://developer.android.com/topic/architecture)를 최대한 따른다.
- 너무 실험적이지 않고, 개발자가 쉽게 이해할 수 있도록 한다.
- 협업에서 여러 개발자가 같은 코드베이스에서 함께 작업할 수 있도록 지원한다.
- 테스트를 로컬 개발 환경에서도 쉽게 돌릴 수 있고, CI 같은 자동화된 빌드 환경에서도 동일하게 신뢰할 수 있게 테스트가 실행되도록 구성한다.
- 빌드 시간을 최소화한다.

## 학습한 내용 
### [1. 아키텍쳐 개요](./아키텍쳐%20개요.md)

### [2. 데이터 레이어](./Data%20Layer.md)

### [3. 도메인 레이어](./Domain%20layer.md)

### [4. UI 레이어](./UI%20Layer.md)