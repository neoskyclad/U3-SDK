# Unturned 코드 인사이트

최근 공개된 Unturned 3.x Unity 프로젝트를 처음 훑어보면서 정리한 개인 분석 메모.

## 전체 구조

- 메인 런타임 코드는 `Assets/Runtime/Assembly-CSharp` 아래에 있다.
- 게임 고유 로직은 대부분 `Assets/Runtime/Assembly-CSharp/Unturned` 아래에 모여 있다.
- 프로젝트는 Unity, Steamworks.NET, 자체 네트워킹 계층을 함께 사용한다.
  - `NetMessaging`
  - `NetInvokable`
  - `NetPak`
  - Steam Networking, Steam Networking Sockets, System Sockets, loopback, legacy UNet 전송 구현
- 오래된 `ask*` / `tell*` 메서드는 obsolete 호환 래퍼로 많이 남아 있고, 비교적 최신 코드는 `ClientStaticMethod`, `ServerStaticMethod`, `ClientInstanceMethod`, `ServerInstanceMethod` 같은 typed RPC 핸들을 쓰는 쪽으로 이동한 흔적이 보인다.

## 서버 권위 구조

게임플레이 코드에서 말하는 "server"는 보통 dedicated server만 의미하지 않고 `Provider.isServer`를 의미한다. dedicated server는 그중 더 좁은 케이스이고, 이때는 `Dedicator.IsDedicatedServer`로 따로 검사한다.

즉 Unturned는 대략 이렇게 동작할 수 있다.

- Dedicated server: 화면 없는 권위 서버.
- Listen/local host: 한 플레이어의 클라이언트가 동시에 권위 서버 역할을 한다.
- Client: 위 둘 중 하나에 접속한 순수 클라이언트.

Steam 로비나 매치메이킹은 게임 시작 전 연결 조율 레이어에 가깝다. 실제 게임플레이가 시작되면 어느 한 피어가 서버 권위를 가진다.

## 좀비 시뮬레이션

관련 파일:

- `Assets/Runtime/Assembly-CSharp/Unturned/Managers/ZombieManager.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Managers/ZombieRegion.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Zombies/Zombie.cs`

좀비 업데이트는 크게 두 흐름으로 나뉜다.

### AI Tick

`Zombie.tick()`은 비싼 의사결정 로직을 처리한다.

- 타겟 플레이어가 아직 유효한가?
- 타겟이 죽었거나, 물에 들어갔거나, nav bounds 밖으로 나갔는가?
- 좀비가 끼었는가?
- 바리케이드, 구조물, 차량, 오브젝트, 플레이어 중 무엇을 공격해야 하는가?
- 특수 좀비가 특수 능력을 써야 하는가?
- pathfinding이 어느 지점으로 이동해야 하는가?

실제 이동은 pathfinding movement 컴포넌트에 `seeker.Move(delta)` 형태로 위임된다.

중요한 점은 모든 좀비가 항상 풀 시뮬레이션되는 것이 아니라는 점이다. `Zombie.isHunting`이 바뀔 때 좀비를 `ZombieManager.tickingZombies`에 넣거나 뺀다. 즉 가만히 있는 좀비는 비교적 저렴하고, 실제로 활성화되었거나 어그로가 끌린 좀비만 AI tick 비용을 쓴다.

Dedicated server에서는 이 작업을 프레임 단위로 더 쪼갠다.

- `ZombieManager.Update()`가 `tickingZombies`의 일부만 처리한다.
- dedicated server에서는 한 프레임에 최대 약 50마리까지만 tick한다.
- non-dedicated 모드에서는 전체 리스트를 한 번에 tick할 수 있다.

배회 행동에도 별도 제한이 있다.

```csharp
ZombieManager.canSpareWanderer => wanderingCount < 8 && tickingZombies.Count < 50
```

즉 배경 배회 좀비 수를 의도적으로 제한해서 전체 좀비 집단이 전부 활성 AI 에이전트가 되는 상황을 피한다.

### 상태 / 네트워크 / 애니메이션 업데이트

`Zombie.OnUpdate()`는 원래 Unity `Update()` 메시지로 호출되던 흐름이었는데, 2026년 주석을 보면 2500마리 이상의 좀비가 있는 맵에서 이 방식이 비싸졌다고 적혀 있다. 현재는 `ZombieManager`가 플레이어가 있는 region의 좀비에 대해서만 수동으로 `OnUpdate()`를 호출하는 방식이다.

서버에서 `OnUpdate()`는 위치나 yaw가 충분히 바뀌었는지 확인해 state update를 보낼지 결정한다.

- 위치가 `Provider.UPDATE_DISTANCE`보다 많이 바뀌었거나, yaw가 1도 이상 바뀌면 updated로 표시한다.
- `zombieRegion.updates`를 증가시킨다.
- 이후 `ZombieManager.updateRegionsAndSendZombieStates()`가 해당 nav region의 업데이트된 좀비 상태를 직렬화해서 클라이언트에 보낸다.

클라이언트에서는 `OnUpdate()`가 서버에서 받은 최신 목표값으로 보간한다.

- `interpPositionTarget`
- `interpYawTarget`
- `Provider.INTERP_SPEED`

요약하면 전형적인 구조다.

- 서버가 진실을 가진다.
- 클라이언트는 원격 엔티티를 부드럽게 보이도록 보간한다.

## 플레이어 입력, 예측, 재조정

관련 파일:

- `Assets/Runtime/Assembly-CSharp/Unturned/Player/PlayerInput.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Player/PlayerMovement.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Player/CapsuleHistory.cs`

입력 모델은 멀티플레이 액션 게임에서 흔히 보이는 형태다.

- 클라이언트는 자기 이동을 즉시 예측해서 먼저 시뮬레이션한다.
- 서버는 같은 입력을 권위적으로 다시 시뮬레이션한다.
- 서버는 예측이 맞았으면 ack를 보내고, 틀렸으면 correction을 보낸다.
- 클라이언트는 correction 지점으로 되감은 뒤 아직 ack되지 않은 입력을 다시 재생한다.

중요한 상수:

- `PlayerInput.RATE = 0.08f`
- `PlayerInput.SAMPLES = 4`
- `PlayerInput.TOCK_PER_SECOND = 50`

즉 이동 입력 패킷은 대략 12.5Hz 정도로 보내지고, 장비/총기 `tock` 처리는 50Hz 리듬을 가진다.

### 클라이언트 흐름

`PlayerInput.FixedUpdate()`에서 로컬 플레이어는 다음 일을 한다.

1. 이동, 자세, 점프, 달리기, 기울이기, 조준 안정화, 플러그인 키, 공격 입력을 캡처한다.
2. life, stance, movement, equipment, animator를 로컬에서 먼저 시뮬레이션한다.
3. 예측된 이동 결과를 `clientInputHistory`에 저장한다.
4. `WalkingPlayerInputPacket` 또는 `DrivingPlayerInputPacket`을 만든다.
5. `SendInputs`로 서버에 보낸다.

걷기 이동 패킷에는 클라이언트가 시뮬레이션을 끝낸 뒤의 위치도 들어간다.

- `WalkingPlayerInputPacket.clientPosition`

### 서버 흐름

서버는 `PlayerInput.ReceiveInputs()`에서 입력을 받는다.

- 입력 스팸을 rate-limit한다.
- 순서가 어긋난 simulation frame number를 거부한다.
- 패킷을 `serversidePackets`에 큐잉한다.
- 입력 사이에 큰 gap이 있으면 fake lag 가능성으로 보고 penalty frame counter를 적용한다.

이후 서버 `FixedUpdate()`가 패킷을 하나 꺼내 같은 시뮬레이션 단계를 실행한다.

- `player.life.simulate(...)`
- `player.look.simulate(...)`
- `player.stance.simulate(...)`
- `player.movement.simulate(...)`
- `player.equipment.simulate(...)`
- `player.animator.simulate(...)`

걷기 이동에서는 서버가 두 위치를 비교한다.

- 클라이언트가 보고한 위치: `walkingPacket.clientPosition`
- 서버가 직접 계산한 결과: `transform.position`

허용 오차는 Unity 월드 단위 기준 2cm다.

```csharp
const float errorToleranceDistance = 0.02f; // 2cm
const float sqrErrorToleranceDistance = errorToleranceDistance * errorToleranceDistance;
if ((walkingPacket.clientPosition - serverPosition).sqrMagnitude > sqrErrorToleranceDistance)
{
    SendSimulateMispredictedInputs.Invoke(...);
}
else
{
    SendAckGoodInputs.Invoke(...);
}
```

거리는 세 축의 유클리드 거리 제곱으로 잰다. `sqrMagnitude`를 쓰는 이유는 매번 square root를 계산하지 않기 위해서다. Unity 관례상 1 unit = 1 meter로 보면 `0.02f`는 2cm다.

### 클라이언트 재시뮬레이션

서버가 correction을 보내면 `ClientResimulate()`가 다음을 수행한다.

1. correction frame까지 ack된 history를 제거한다.
2. stance, position, velocity, stamina, 일부 life timing field를 서버 상태로 잠시 맞춘다.
3. `clientInputHistory`에 남아 있는 입력을 모두 다시 재생한다.
4. 로컬 카메라/조준 회전을 복원한다.

이게 client-side prediction + server reconciliation이다. 조작감은 즉시 반응하게 만들면서도, 최종 권위는 서버에 남긴다.

## 히트 검증

히트 검증도 비슷한 철학을 따른다. 클라이언트 편의는 허용하지만, 최종 판단은 서버가 한다.

클라이언트가 raycast hit 결과를 보낼 수는 있지만 서버가 그대로 믿지는 않는다. 서버는 타겟 identity, 거리, hit point가 현재 또는 최근 타겟 bounds 안에 들어갈 수 있는지 등을 검증한다.

`BoundsHistory`는 character-controller bounds를 50 tick rate 기준 1초 동안 저장한다.

- `CAPACITY = 50`
- `AddCharacterControllerBounds(...)`
- `ContainsPoint(...)`

이 구조는 lag compensation을 제공하면서도 클라이언트가 임의의 히트를 만들어내는 것을 막는다. 특히 플레이어 히트 검증에서 중요하다. 총을 쏜 사람이 본 상대 위치와 서버 현재 위치가 이미 달라져 있을 수 있기 때문이다.

## 데미지 시스템

관련 파일:

- `Assets/Runtime/Assembly-CSharp/Unturned/Tools/DamageTool.cs`

`DamageTool`은 여러 damageable entity를 처리하는 중앙 관문이다.

- 플레이어
- 좀비
- 동물
- 차량
- 바리케이드
- 구조물
- 자원
- 레벨 오브젝트
- 폭발

흥미로운 점:

- 플레이어 데미지는 limb와 clothing armor를 반영한다.
- 방어구가 데미지를 흡수하면 clothing quality가 감소할 수 있다.
- 폭발 방어는 장착된 옷 개수에 단순 비례하지 않고 여러 clothing slot의 평균에 가깝게 계산한다.
- PvP와 friendly-fire 체크가 중앙화되어 있다.
- 폭발 데미지는 physics overlap과 line-of-sight 체크를 함께 쓴다.
- 플러그인에서 damage request에 개입할 수 있는 hook이 있다.

작은 클래스는 아니지만, Unturned처럼 damageable entity 종류가 많고 플러그인/모드 호환성이 중요한 게임에서는 데미지를 한 관문으로 모으는 선택이 이해된다.

## 총기 런타임

관련 파일:

- `Assets/Runtime/Assembly-CSharp/Unturned/Useable/UseableGun.cs`
- `Assets/Runtime/Assembly-CSharp/Unturned/Bundles/ItemGunAsset.cs`
- 여러 attachment, magazine, projectile, item asset 클래스

`UseableGun.cs`는 총기 공통 런타임 역할을 하는 매우 큰 클래스다. 여기에는 다음 책임이 모여 있다.

- 발사 모드
- 탄약과 탄창
- 부착물
- 반동, sway, spread, 조준
- ballistic bullet
- projectile launcher
- 폭발 탄환/투사체
- hit 처리
- 서버/클라이언트 시각 효과
- UI와 부착물 메뉴
- `onBulletSpawned`, `onBulletHit`, `onProjectileSpawned` 같은 플러그인 이벤트

모든 총기가 일반적인 OOP식 개별 스크립트로 나뉘어 있는 구조는 아니다. 대신 `ItemGunAsset`, magazine asset, barrel/sight/grip/tactical attachment, item state byte 같은 데이터로 총기 차이를 표현하고, `UseableGun`이 공통 실행 엔진 역할을 한다.

### 이게 나쁜 설계인가?

비용이 큰 설계인 것은 맞다.

- 파일이 매우 크다.
- 한 클래스에 책임이 많이 몰려 있다.
- 관련 없어 보이는 수정이 다른 기능을 깨뜨릴 위험이 있다.
- 단위 테스트하기 어렵다.
- item data, networking, animation, inventory를 같이 알아야 이해된다.

하지만 맥락상 이해되는 부분도 있다.

- Unturned는 오래된 라이브 게임이다.
- legacy item과 mod가 많다.
- 플러그인 호환성이 중요하다.
- 무기는 server validation, client feel, visual effect, UI, inventory state가 강하게 엮인다.
- 데이터 기반 단일 런타임은 기존 콘텐츠를 per-gun script 없이 계속 살릴 수 있다.

그래서 "깨끗한 아키텍처"라기보다는 "오래 살아남은 라이브 게임 아키텍처"로 보는 편이 맞다. god class 냄새는 분명하지만, 동시에 수년간의 호환성, 버그 수정, 실전 제약이 누적된 결과처럼 보인다.

## 전체 감상

Unturned 런타임은 첫인상보다 훨씬 서버 권위적이고 성능을 의식한 구조다.

핵심 패턴은 다음과 같다.

- 반응성을 위해 client prediction을 쓴다.
- 권위를 위해 server re-simulation과 validation을 쓴다.
- world simulation과 network traffic을 줄이기 위해 region/nav bounds를 활용한다.
- 오래된 콘텐츠와 모드를 살리기 위해 compatibility wrapper와 data-driven asset 구조를 유지한다.
- 실제 맵과 라이브 서버에서 병목이 드러난 지점에 실용적인 최적화를 추가한다.

전체적으로는 새로 만든 깔끔한 엔진이라기보다, 수많은 멀티플레이 edge case를 겪으며 살아남은 코드베이스라는 인상이 강하다.
