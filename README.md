# Castle-Guardian-VR

> 본 저장소는 포트폴리오 공개용으로 소스 코드와 기술 문서 중심으로 정리한 버전입니다. 원본 프로젝트의 외부 에셋, 사운드, 모델, 씬 파일은 저작권과 용량 문제로 제외했습니다.

Unity 기반 VR 디펜스 게임 포트폴리오입니다. 플레이어가 VR 환경에서 활을 조작해 적 웨이브를 막고, 타워와 화살을 활용해 성을 방어하는 구조로 제작했습니다.

> VR 입력, 투사체, 웨이브, 적 AI, 타워, 오브젝트 풀링, ScriptableObject 데이터 분리 등 클라이언트 시스템 구현 경험을 보여주기 위한 개인 프로젝트입니다.

## 프로젝트 정보

| 항목 | 내용 |
| --- | --- |
| 장르 | VR 디펜스 |
| 개발 형태 | 개인 프로젝트 |
| 개발 도구 | Unity |
| 주요 기술 | C#, Meta XR SDK, XR Interaction Toolkit, NavMesh, ScriptableObject, Object Pooling |
| 플랫폼 | PC VR / Meta Quest 계열 테스트 목적 |

## 포트폴리오 포인트

- VR 환경에서 활 조작, 시선/입력 처리, 투사체 발사 흐름을 구현했습니다.
- 적 웨이브, 스폰, 진형, 이동, 성 공격 흐름을 시스템 단위로 분리했습니다.
- 타워 설치와 업그레이드, 화살 타입, 투사체 효과를 상속 구조와 데이터 기반으로 관리했습니다.
- ScriptableObject로 적, 타워, 투사체, 웨이브 데이터를 분리해 밸런싱과 확장성을 확보했습니다.
- 오브젝트 풀링으로 적, 투사체, 파티클, 오디오 소스의 반복 생성 비용을 줄였습니다.

## 주요 구현

### VR 상호작용

- Meta XR SDK와 XR Interaction Toolkit을 활용해 VR 환경에서 플레이어 입력과 시점 제어를 구성했습니다.
- 활 조준, 화살 발사, 투사체 궤적 표현 등 VR 기반 조작 흐름을 구현했습니다.
- 일반 화살, 독 화살, 폭발 화살 등 화살 타입별 동작을 분리했습니다.

### 투사체 시스템

- `Projectile` 기반 클래스를 중심으로 공통 이동, 충돌, 데미지 처리를 구성했습니다.
- `NormalArrow`, `PoisonArrow`, `ExplosiveArrow`로 화살별 효과를 분리했습니다.
- 포물선 궤적 시각화와 충돌 후 이펙트 처리를 연결했습니다.

### 적 AI와 웨이브 시스템

- `WaveManager`, `SpawnManager`, `FormationManager`를 통해 적 생성과 웨이브 진행을 관리했습니다.
- 적 데이터는 ScriptableObject로 분리해 체력, 이동 속도, 보상, 타입별 속성을 관리했습니다.
- NavMesh 기반 이동을 활용해 적이 목표 지점으로 이동하고 성을 공격하도록 구성했습니다.

### 타워 시스템

- `Tower` 기반 클래스를 중심으로 공통 공격 로직을 구성했습니다.
- `ArcherTower`, `ExplosiveTower`, `SlowTower`로 타워 역할을 분리했습니다.
- `BuildManager`와 `BuildableNode`를 통해 타워 설치 위치와 설치 가능 여부를 관리했습니다.

### 데이터와 최적화

- Enemy, Tower, Projectile, Wave 데이터를 ScriptableObject로 분리했습니다.
- Enemy, Projectile, Particle, AudioSource 풀을 구성해 런타임 생성/삭제 부담을 줄였습니다.
- SoundDatabase와 AudioSourcePool을 활용해 사운드 재생 흐름을 관리했습니다.

## 프로젝트 구조

```text
Assets/
├── 06.Scripts
│   ├── 00.System             # GameManager, UIManager, SoundManager 등 공통 시스템
│   ├── 01.WaveSystem         # 적 생성, 웨이브, 진형 관리
│   ├── 02.BowArrow           # VR 활 조작, 화살, 투사체
│   ├── 03.Tower              # 타워 설치 및 타워별 공격 로직
│   ├── 04.ScriptableObject   # 적, 타워, 투사체, 웨이브 데이터
│   ├── 05.Interface          # 데미지, 투사체, 업그레이드 인터페이스
│   └── 06.ObjectPool         # 적, 투사체, 파티클, 오디오 풀링
├── Packages                  # Unity 패키지 정보
└── ProjectSettings           # 프로젝트 설정
```

## 실행 방법

1. Unity Hub에서 프로젝트를 엽니다.
2. XR 관련 패키지와 Meta/Oculus 환경 설정을 확인합니다.
3. VR 장비 연결 또는 XR 시뮬레이터 환경에서 시작 씬을 실행합니다.

## 개발하며 배운 점

- VR 프로젝트는 일반 PC 게임보다 입력, 시점, 상호작용 피드백을 더 명확하게 분리해야 한다는 점을 배웠습니다.
- 웨이브, 적 AI, 타워, 투사체처럼 서로 연결되는 시스템은 데이터 구조와 이벤트 흐름을 먼저 정리해야 유지보수가 쉬워진다는 점을 체감했습니다.
- 반복 생성되는 객체는 풀링 구조로 관리해야 VR 환경에서 프레임 안정성을 확보하는 데 도움이 된다는 점을 확인했습니다.

## 개선 예정

- README에 플레이 영상과 기술 문서 링크 추가
- VR 입력 튜토리얼 및 조작 안내 UI 보강
- 타워 밸런싱 데이터 정리
- 불필요한 외부 에셋과 긴 경로를 정리해 저장소 checkout 안정성 개선
