# A Week

<img width="960" height="540" alt="배경" src="https://github.com/user-attachments/assets/df913484-3069-4b0e-983e-296c546b76f3" />


> **Unreal Engine 5** 기반의 TPS 서바이벌 게임 프로젝트입니다. 낮 시간의 자원 탐색/수집 파밍 단계와 밤 시간의 TPS 슈팅/방어 단계를 분리하여 긴장감 있는 장르 전환을 목표로 제작했습니다.

- **프로젝트 구분**: 팀 프로젝트 (5명)
- **장르**: TPS 서바이벌 슈팅
- **담당 역할**: 클라이언트 개발 (플레이어 메카닉, 전투 시스템, HUD UI)
- **개발 환경**: Unreal Engine 5.5.4, C++, Visual Studio
- **개발 기간**: 4개월 (2025.08.05~2025.12.08)
- **주요 링크**:
  - 🎬 [YouTube 데모 플레이 영상](https://youtu.be/aVBd1-4LbnU?si=xALBAkW1rRmQv8Vv)

---

## 게임 소개
- 낮과 밤이라는 시간대를 설정하여 낮에는 수집 및 제작, 밤에는 생존, 슈팅으로 장르가 변하는 게임입니다.
- 무기 뿐 아니라 식량과 포탑 등 다양한 아이템을 제작하고 사용할 수 있도록 제작했습니다.
- 좀비는 시야와 소리 모두 탐지가 가능하고, 밤에는 탐지 범위가 더 증폭되어 긴장감 넘치는 게임플레이 경험을 주도록 설계했습니다.

---

## 핵심 구현 컨텐츠 및 아키텍처

### 1. TPS 플레이어 로코모션 & 무기별 애니메이션 오버라이드
- **BlendSpace & AimOffset**: 이동 방향, 속도 및 시선 처리를 위한 조준 앵글 로직을 구현했습니다.
- **런타임 애니메이션 오버라이드**: 무기 종류(권총, 소총, 근접 무기 등)에 따라 적절한 애니메이션이 재생되도록 `AnimInstance` 가 데이터테이블을 캐싱하여 간편한 애니메이션 오버라이드 구조를 설계했습니다.
- **신체 부위에 따른 애니메이션 블렌딩**: 무기 종류(권총, 소총, 근접 무기 등)에 따라 상체 애니메이션이 유연하게 전환되도록 `AnimInstance` 내 Layered blend per bone 및 Animation Montage 오버라이드 구조를 설계했습니다.

<img width="887" height="377" alt="데이터테이블&애님인스턴스" src="https://github.com/user-attachments/assets/2540bf9c-164b-48ec-876a-57641949a9b5" />

<br>

### 2. `Motion Warping` 기반 파쿠르 시스템
- **Trace 기반 지형 탐지**: Line/Capsule Trace를 사용해 장애물의 높이, 두께, 벽면 법선(Normal)을 실시간으로 산출했습니다.
- **Motion Warping Plugin 활용**: 계산된 지점(Vault Point, Climb Point)에 플레이어 몽타주 루트 모션을 정확히 동기화하여 자연스러운 장애물 넘기 및 벽 오르기 동작을 구현했습니다.
  
<img width="359" height="279" alt="image" src="https://github.com/user-attachments/assets/06d2511b-a518-49f2-b059-10dcc0dc9a0e" />

<br>

### 3. `UGameEventMessageSubsystem` 기반 HUD 및 UMG UI
- **객체와 UI간의 의존성 최소화**: 플레이어 캐릭터나 스태미너/체력 컴포넌트가 UI 클래스를 직접 참조하지 않도록 옵저버 패턴을 활용했습니다.
- **이벤트 기반 데이터 동기화**: 체력, 허기, 스태미너, 탄약 수치 변경 시 `GameplayTag`로 이벤트를 분류하여 상응하는 위젯의 수치 및 외형(애니메이션 등)이 갱신되도록 제작했습니다.

다음은 주요 코드 요약 (위젯, 객체) 입니다.
> AWeekPlayerStateWidget.cpp
```cpp
void UAWeekPlayerStateWidget::NativeConstruct()
{
	Super::NativeConstruct();

	/*--------------INIT--------------*/
	HealthBar = Cast<UProgressBar>(GetWidgetFromName(TEXT("HealthBar")));
	HungerBar = Cast<UProgressBar>(GetWidgetFromName(TEXT("HungerBar")));

	/*--------------EVENTMSSAGE--------------*/
	HPChangedHandle = UGameEventMessageSubsystem::Get(this).RegisterListener<FHPChangedHandle>(
		FGameplayTag::RequestGameplayTag(FName("Event.HPChanged")),
		[this](FGameplayTag Channel, const FHPChangedHandle& Msg)
		{
			HealthBar->SetPercent(Msg.HP / Msg.MaxHP);
		}
	);

	HungerChangedHandle = UGameEventMessageSubsystem::Get(this).RegisterListener<FHungerChangedMessage>(
		FGameplayTag::RequestGameplayTag(FName("Event.HungerChanged")),
		[this](FGameplayTag Channel, const FHungerChangedMessage& Msg)
		{
			HungerBar->SetPercent(Msg.Hunger / Msg.MaxHunger);
		}
	);
}
```

> AWeekPlayerCharacter.cpp
```cpp
void AAWeekPlayerCharacter::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	FHPChangedHandle Msg;
	Msg.HP = mDamageSystem->Execute_GetCurrentHealth(this);
	Msg.MaxHP = mDamageSystem->Execute_GetMaxHealth(this);
	UGameEventMessageSubsystem::Get(this).BroadcastMessage(
		FGameplayTag::RequestGameplayTag(FName("Event.HPChanged")),
		Msg);
}
```

---

## 트러블슈팅

### 1. 총기 프레임/Notify 의존성으로 인한 연사 속도 오류
- **문제**: 애니메이션 몽타주의 AnimNotify에 격발 로직을 바인딩해 두었으나, 프레임 드랍이나 몽타주 재생 속도 조절 시 실제 프레임에 맞춰 연사 속도가 비정상적으로 빨라지거나 느려지는 현상이 발생했습니다. [YouTube 문제 상황 1](https://www.youtube.com/watch?v=-_sebh7_O0Q)
- **해결**:
  - 무기 데이터(DataTable)에 `FireRate` 항목을 추가하고, `TickComponent` 내에서 DeltaTime 누적 수치(`mTimeSinceLastShot`)를 계산하는 타이머 기반 로직으로 전환했습니다.
  - 애니메이션 재생 타이밍과 실제 격발 타이밍을 분리하여 안정적인 사격 주기를 확보했습니다.

### 2. 높이가 다른 벽 오르기 시 Motion Warping 위치 어긋남
- **문제**: Motion Warping으로 착지/잡기 지점을 지정했으나, 장애물 높이에 따라 캡슐 콜리전과 벽 상단의 연산 지점이 달라지면서 파쿠르 종료 후 플레이어가 공중에 뜨거나 벽 내부로 파묻히는 문제가 있었습니다. [YouTube 문제 상황 2](https://www.youtube.com/watch?v=SwFqHIYX0_8)
- **해결**:
  - Trace로 측정한 실제 벽 높이 오프셋(`mWallHeight`)을 계산하여, 파쿠르 시작 시 플레이어의 캡슐 콜리전 위치와 이동 모드(`MOVE_Flying`)를 수동으로 1차 보정한 뒤 Motion Warping을 수행하도록 수정했습니다.

---

## 개발 회고 및 성찰

- **언리얼 엔진 5 핵심 플러그인 및 시스템 활용**: `EnhancedInput`, `MotionWarping`, `AssetManager` 등 엔진 내장 플러그인과 프레임워크를 프로젝트에 직접 적용하며, 각 기능의 내부 동작 원리와 확장 가능성을 명확히 이해할 수 있었습니다. 언리얼 엔진 개발은 OOP에 대한 깊은 이해도가 필수적이라는 것을 몸소 느낄 수 있었고, 엔진의 잠재능력을 더 끌어올리기 위해서는 C++의 class에 대한 높은 이해도가 있어야 한다는 깨달음을 얻었습니다.
- **데이터 및 이벤트를 통한 구조 개선**: UGameEventMessageSubsystem과 Delegate를 활용한 이벤트 기반 설계를 통해 시스템 간 결합도를 최소화하고, DataTable 기반의 데이터 중심 방식을 적용하여 유지보수성과 확장성이 높고 안정적인 클라이언트 아키텍처의 중요성을 체감했습니다. 팀과 개발하면서 팀원이 내 코드를 유의깊게 볼 수 있고, 내 코드로 컨텐츠를 확장할 수 있다는 점을 깊게 느꼈습니다. 이러한 맥락에서 왜 유지보수성과 가독성이 중요한지 알 수 있었고, 단순히 높은 기술력 뿐만 아니라 팀원의 스타일을 고려하여 코드를 작성할 줄 아는 유연함 역시 개발자에게 필요한 역량이라는 것을 깨달았습니다.
