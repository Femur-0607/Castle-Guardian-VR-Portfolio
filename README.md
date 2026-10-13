# Castle Guardian VR - 활 조작과 웨이브 방어를 VR 입력에 맞춰 연결한 디펜스

![Unity](https://img.shields.io/badge/Unity-222222?style=flat-square&logo=unity&logoColor=white) ![C#](https://img.shields.io/badge/C%23-68217A?style=flat-square&logo=csharp&logoColor=white) ![Meta XR](https://img.shields.io/badge/Meta_XR_SDK-0467DF?style=flat-square&logo=meta&logoColor=white) ![Solo](https://img.shields.io/badge/1인-개발-555555?style=flat-square) [![YouTube](https://img.shields.io/badge/YouTube-플레이_영상-FF0000?style=flat-square&logo=youtube&logoColor=white)](https://youtu.be/KxXcOsSTjww)

> 본 저장소는 포트폴리오 공개용 **코드 열람 저장소**입니다. 원본 프로젝트의 외부 에셋, 사운드, 모델, 씬 파일은 저작권과 용량 문제로 제외했으므로 Unity에서 그대로 실행되지 않습니다. 실제 플레이는 아래 영상으로 확인해 주세요.

[![Castle Guardian VR 플레이 영상](https://img.youtube.com/vi/KxXcOsSTjww/maxresdefault.jpg)](https://youtu.be/KxXcOsSTjww)

▶︎ [플레이 영상](https://youtu.be/KxXcOsSTjww) · 📄 [기술문서 PDF](https://drive.google.com/file/d/1QAfVO60DJtMbQT3aGXmA1sMr7wmtxyMy/view?usp=sharing) · 👤 [GitHub 프로필](https://github.com/Femur-0607)

## 프로젝트 정보

| 항목 | 내용 |
| --- | --- |
| 장르 | 3D VR 타워 디펜스 |
| 개발 기간 | 2025.03 – 2025.04 (8주) |
| 개발 형태 | 1인 프로젝트 |
| 개발 환경 | Unity 6000.0.32f1, C#, Meta XR SDK |
| 플랫폼 | PC VR / Meta Quest 계열 |
| 주요 기술 | VR 입력, 투사체 궤적, 웨이브·스폰, NavMesh, ScriptableObject, Unity ObjectPool API |
| 상태 | 완료 (플레이 가능 빌드 기준, 영상 참고) |

VR 환경에서 트리거로 차징한 화살을 쏘고 타워를 설치해 적 웨이브로부터 성을 지키는 게임입니다. PC용 핵심 시스템(투사체·웨이브·타워)을 먼저 만든 뒤, VR 컨트롤러 입력과 피드백을 붙이는 순서로 개발했습니다.

## 담당 범위

1인 프로젝트로 기획·구현·테스트를 모두 진행했습니다.

- VR 입력(트리거 유지 시간 차징)과 화살 발사, 포물선 궤적 표시
- `Projectile` 추상 클래스와 Normal / Poison / Explosive 화살 분리
- 타입별 `IObjectPool<Projectile>`을 가진 `ProjectilePool`, 적·파티클·오디오 풀
- `WaveManager` → `SpawnManager` → `Enemy` 흐름과 `FormationManager` 진형 배치
- `Tower` 공통 로직과 Archer / Explosive / Slow 타워 파생, `BuildManager` 설치
- Enemy / Tower / Projectile / Wave 데이터를 ScriptableObject로 분리
- 외부 에셋: Meta XR SDK 샘플, 포물선 궤적 에셋, 3D 모델·사운드(공개본에서 제외)

## 핵심 구현

### 1. 트리거 유지 시간을 발사 세기로 바꾸는 VR 활 입력

**문제.** VR에서는 트리거를 누르고 유지한 뒤 놓는 입력을 구분해야 하고, 조준 결과가 바로 보이지 않으면 조작이 불안합니다.

**선택.** 발사를 `OnFireStart` → `OnFireCharging` → `OnFireRelease` 세 단계로 나누고, 트리거 유지 시간을 누적해 `chargeTimer / chargeTime`을 0~1로 제한했습니다. 이 값으로 `minSpeed`~`maxSpeed` 사이의 발사 세기를 보간하고, 차징 중 `ProjectileCurveVisualizer`가 LineRenderer로 궤적과 예상 충돌 지점을 표시하도록 했습니다. `chargeTime`의 코드 기본값은 1.5초입니다.

**결과.** 발사 전에 궤적이 보이므로 플레이어가 힘 조절을 예측할 수 있습니다. 세 단계가 분리돼 있어 화살 타입을 바꿔도 입력 코드는 손대지 않습니다.

- [ArrowShooter.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/02.BowArrow/ArrowShooter.cs) · [ProjectileCurveVisualizer.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/02.BowArrow/ProjectileCurveVisualizer.cs)

### 2. 화살 타입별 풀과 재사용 전 상태 초기화

**문제.** 화살·적·파티클·오디오가 반복 생성·파괴되고, 재사용한 화살에 이전 발사의 속도·충돌 상태가 남으면 엉뚱한 궤적이 나옵니다.

**선택.** `ProjectilePool`이 화살 타입을 키로 `Dictionary<ProjectileType, IObjectPool<Projectile>>`을 보관하고, 꺼낼 때와 반환할 때 모두 `Projectile.ResetState()`를 호출합니다. `ResetState`는 자식 변환·속도·각속도·충돌·생성 시각·데미지를 초기화합니다. 적 풀은 활성화·위치만 맡고, 데이터 주입과 `Enemy.ResetState`는 `SpawnManager`가 합니다.

**결과.** 현재 선택한 화살 타입에 맞는 풀에서 바로 꺼내 쓰고, 재사용 시 잔류 상태 문제가 코드 경로상 차단됩니다.

- [ProjectilePool.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/06.ObjectPool/ProjectilePool.cs) · [Projectile.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/02.BowArrow/Projectile.cs) · [EnemyPool.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/06.ObjectPool/EnemyPool.cs)

### 3. 데이터로 정의한 웨이브와 코루틴 기반 진행

**문제.** 웨이브마다 스폰 지점·적 종류·간격이 달라 코드에 직접 쓰면 밸런스 조정 때마다 재빌드가 필요합니다.

**선택.** `WaveData`에 스폰 패턴 목록을 두고, `WaveManager.ProcessWave`가 패턴별 `SpawnEnemiesForPattern` 코루틴을 모두 시작한 뒤 완료를 기다리도록 했습니다. 종료는 스폰 완료와 별개로 `Update`에서 활성 스폰 지점의 생존 수가 0 이하인지 확인했습니다. 보상·영혼 수집 후 종료 이벤트를 보내고, 상점에서 다음 웨이브를 요청하면 인덱스 9 이상에서 보스 BGM·시작 이벤트 경로로 분기했습니다.

**결과.** 웨이브 구성은 ScriptableObject로 분리하고, 스폰 진행과 생존 수 기반 종료 판정은 따로 처리했습니다.

- [WaveManager.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/01.WaveSystem/WaveManager.cs) · [WaveData.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/04.ScriptableObject/WaveData.cs) · [SpawnManager.cs](https://github.com/Femur-0607/Castle-Guardian-VR-Portfolio/blob/32f4f295413da73c82fcdd691d86c9fa91eb9b84/Assets/06.Scripts/01.WaveSystem/SpawnManager.cs)

## 트러블슈팅 — 처음 다루는 VR 입력을 작은 단위로 검증

VR 개발 경험이 없어 입력·상호작용·발사를 한 번에 붙이면 어디서 문제가 나는지 알 수 없었습니다. Meta 공식 문서와 샘플을 기준으로 **컨트롤러 입력 읽기 → 차징 상태 표시 → 궤적 표시 → 실제 발사** 순서로 하나씩 확인하며 붙였습니다. 각 단계가 독립적으로 동작한 뒤에야 다음 단계를 연결했기 때문에, 발사가 안 될 때 입력 문제인지 궤적 계산 문제인지 바로 구분할 수 있었습니다.

## 프로젝트 구조

```text
Assets/06.Scripts/
├── 00.System          # GameManager, EventManager, UIManager, SoundManager, 풀 매니저
├── 01.WaveSystem      # WaveManager, SpawnManager, FormationManager, Enemy, Boss
├── 02.BowArrow        # ArrowShooter, Projectile 계열, 궤적 시각화
├── 03.Tower           # Tower 계열, BuildManager, BuildableNode
├── 04.ScriptableObject# Enemy / Tower / Projectile / Wave / Sound 데이터
├── 05.Interface       # IDamageable, IProjectile, IUpgradeable
└── 06.ObjectPool      # Projectile / Enemy / Particle / AudioSource 풀
```

## 알고 있는 한계와 다음 과제

- `ProjectilePool`은 `collectionCheck`를 끄고 있고 `ReturnToPool`에 독립적인 반환 가드가 없습니다. 같은 프레임에 수명 만료와 충돌이 겹칠 때 이중 반환이 없는지 별도 검증 장면으로 확인할 계획입니다.
- 풀링으로 생성·파괴 호출은 줄었지만 FPS·GC 수치는 측정하지 않았습니다. 수치는 측정 후에만 적습니다.
- Unity ObjectPool의 `defaultCapacity`는 미리 생성 개수가 아니라 내부 저장 용량입니다. 코드 주석 일부가 이를 혼동하고 있어 정리할 예정입니다.

## 공개 범위와 확인 방법

- 이 저장소는 **코드 열람용**입니다. 씬·프리팹·에셋이 없어 Unity Hub에서 열어도 플레이할 수 없습니다.
- 플레이 결과는 [영상](https://youtu.be/KxXcOsSTjww), 구현 설명은 [기술문서 PDF](https://drive.google.com/file/d/1QAfVO60DJtMbQT3aGXmA1sMr7wmtxyMy/view?usp=sharing)를 참고해 주세요.
- 코드 링크는 커밋 `32f4f29`에 고정돼 있어 이후 변경과 무관하게 같은 내용을 가리킵니다.

© 2026 Yang Ji-seop. Portfolio use only.
