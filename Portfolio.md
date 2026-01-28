# K 데몬헌터 키우기 - 기술 포트폴리오

## 프로젝트 요약

| 항목 | 내용 |
|------|------|
| **장르** | 모바일 방치형 액션 RPG (라이브 서비스) |
| **역할** | 클라이언트 메인 프로그래머 |
| **개발 환경** | Unity 6000.0.63, C# .NET |
| **플랫폼** | Android / iOS |

### 프로젝트 규모
- C# 스크립트 **936개**
- 서버 데이터 테이블 **80+**
- 스탯 ID **100+**, 연동 시스템 **15개**

### 핵심 기여
| 영역 | 성과 |
|------|------|
| **스탯 시스템** | Dirty Flag 캐싱으로 프레임당 재계산 **수백 회 → 0~10회** |
| **전투 아키텍처** | FSM 기반 상태 관리로 10+ 상태 간 전이 체계화 |
| **서버 동기화** | 변경 데이터 자동 감지/선별 전송으로 네트워크 트래픽 **90%+ 감소** |
| **GC 최적화** | LinqGen, ZString 적용으로 Zero Allocation 달성 |

---

## 목차

1. [복잡한 스탯 시스템 설계와 성능 최적화](#1-복잡한-스탯-시스템-설계와-성능-최적화)
2. [FSM 기반 전투 시스템 아키텍처](#2-fsm-기반-전투-시스템-아키텍처)
3. [네트워크 통신 안정성 및 보안 강화](#3-네트워크-통신-안정성-및-보안-강화)
4. [대규모 코드베이스 관리 전략](#4-대규모-코드베이스-관리-전략)

---

## 1. 복잡한 스탯 시스템 설계와 성능 최적화

### 결과
- 프레임당 스탯 재계산: **수백 회 → 0~10회**
- 100+ 스탯 ID, 15개 시스템 통합 관리
- 신규 시스템 추가 시 기존 코드 수정 최소화

### 핵심 설계
- **Dirty Flag 기반 StatCalculator 패턴**: 이벤트 발생 시에만 재계산
- **단일 Dictionary 기반 다층 스탯 집계**: 고정값 → 비율 → 승수 순서 적용
- **이벤트 기반 트리거**: 스탯 변경 이벤트 구독으로 불필요한 계산 제거

### 설계 배경
15개 시스템(장비, 스킬, 펫, 코스튬, 길드버프, 도감, 영약 등)에서 100+ 스탯을 실시간으로 계산해야 했습니다. 단순 구현 시 프레임마다 전체 재계산이 발생하여 성능 저하가 불가피했고, 이를 해결하기 위해 캐싱 기반 지연 계산 패턴을 도입했습니다.

### 구현 상세

#### 1.1. StatCalculator 패턴

```csharp
public class StatCalculator
{
    private Func<double> computeFunc;
    private List<EventName> refreshEvents;
    private double cachedValue;
    private bool isDirty = true;

    public StatCalculator(Func<double> compute, List<EventName> events)
    {
        this.computeFunc = compute;
        this.refreshEvents = events;

        // 이벤트 발생 시에만 더티 플래그 설정
        foreach (var eventName in refreshEvents)
        {
            EventUtils.AddEventListener(eventName, () => isDirty = true);
        }
    }

    public double ComputeValue()
    {
        // 캐시된 값이 유효하면 재계산 없이 반환
        if (!isDirty)
            return cachedValue;

        // 더티 상태일 때만 재계산
        cachedValue = computeFunc();
        isDirty = false;
        return cachedValue;
    }
}

// 사용 예시
public static StatCalculator Attack = new( () => ComputeAttack(), new List<EventName> { EventName.RefreshStat } );
public static StatCalculator HP = new( () => ComputeHP(), new List<EventName> { EventName.RefreshStat } );
```

#### 1.2. 다층 스탯 집계 시스템

```csharp
public static void ApplyTotalStats()
{
    // 1단계: 모든 스탯 초기화
    foreach (var metaStatKeyInfo in StatKeyUtils.MetaStatKeyDictionary.Values)
    {
        _statDic[metaStatKeyInfo.statId] = 0;
    }

    ProfileUtils.ApplyOwnValuesToPlayerStat();           // 프로필
    CharacterUtils.ApplyStatValuesToPlayerStat();        // 캐릭터 레벨
    ContentsOptionUtils.ApplyOptionValuesToPlayerStat(); // 기혈 시스템
    SkillUtils.ApplyOwnValuesToPlayerStat();             // 술법
    PassiveSkillUtils.ApplyOwnValuesToPlayerStat();      // 부적
    PetUtils.ApplyOwnValuesToPlayerStat();               // 영물
    CostumeUtils.ApplyOwnValuesToPlayerStat();           // 외형
    TacticsBuildUtils.ApplyOwnValuesToPlayerStat();      // 추천 빌드
    PromotionUtils.ApplyOwnValuesToPlayerStat();         // 승급
    GuildBuffUtils.ApplyGuildBuffActiveStatValuesToPlayerStat(); // 길드 버프
    AdBuffUtils.ApplyBuffValuesToPlayerStat();           // 광고 버프
    EquipUtils.ApplyEquipValuesToPlayerStat();           // 장비
    ArtifactUtils.ApplyEquipValuesToPlayerStat();        // 법구
    BookUtils.ApplyBookValuesToPlayerStat();             // 도감
    AltarUtils.ApplyAltarValuesToPlayerStat();           // 영약
    GuardianStatUtils.ApplyStatValuesToPlayerStat();     // 신수
    NeidanUtils.ApplyStatValuesToPlayerStat();           // 내단
}

// 3단계: 최종 계산 (고정값 + 비율 + 승수)
private static double ComputeAttack()
{
    // 기본 공격력 (고정값)
    double baseAtk = _statDic[_statId_atk]
                   + _statDic[_statId_atk_skin]
                   + _statDic[_statId_atk_blood]
                   + _statDic[_statId_atk_equip]
                   + _statDic[_statId_atk_neidan];

    // 공격력 증가율 (비율 합산)
    double atkRate = 1.0
                   + _statDic[_statId_atkRate]
                   + _statDic[_statId_atkRate_skin]
                   + _statDic[_statId_atkRateEquip]
                   + _statDic[_statId_atkRate_artifact]
                   + _statDic[_statId_atkRate_altar]
                   + _statDic[_statId_atkRate_blood]
                   + _statDic[_statId_atkRate_neidan];

    // 광고 버프 (승수)
    double atkRateAds = 1.0 + _statDic[_statId_atkRateAds];

    // 최종 계산: (기본값 × 비율) × 승수
    return baseAtk * atkRate * atkRateAds;
}
```

**트레이드오프:**
- 이벤트 기반 설계로 즉시 계산 대비 약간의 지연 발생 가능 → 프레임 끝에 일괄 처리로 해결
- Dictionary 기반으로 타입 안전성 일부 포기 → IDE 자동완성과 상수 정의로 보완

---

## 2. FSM 기반 전투 시스템 아키텍처

### 결과
- 10+ 상태(Idle, Move, Attack, Skill, Dash, Death 등) 명확히 분리
- 상태 전이 우선순위 체계화: 조이스틱 > 강제이동 > 스킬 > 전투
- 신규 상태 추가 시 기존 코드 수정 없이 확장 가능

### 핵심 설계
- **FSM 패턴**: 상태별 Enter/Update/Exit로 책임 분리
- **우선순위 기반 전이**: 입력 → 강제이동 → 스킬 → 자동 전투 순서
- **싱글톤 상태 인스턴스**: 메모리 효율성 확보

### 설계 배경
플레이어 캐릭터의 상태(Idle, Move, Attack, Skill, Dash, Death, Spawn, ForceMove 등)가 10개 이상이고, 각 상태 간 전이 조건이 복잡했습니다. if-else 체인으로 구현 시 스파게티 코드가 되어 유지보수가 어려워지는 문제를 FSM 패턴으로 해결했습니다.

### 구현 상세

```csharp
public class PlayerController : CharacterController
{
    // FSM 인스턴스
    private StateMachine<PlayerController> stateMachine;

    protected override void Awake()
    {
        base.Awake();

        // FSM 초기화
        stateMachine = new(this);
    }

    private void Update()
    {
        // FSM 업데이트 (현재 상태의 Update 호출)
        stateMachine.OnUpdate();
    }
}
```

#### 2.2. 상태별 명확한 책임 분리

각 상태는 Enter, Update, Exit 메서드를 통해 독립적으로 동작합니다.

```csharp
public void OnIdleEnter()
{
    // 애니메이션 속도 0으로 설정
    if (animator)
    {
        animator.SetFloat(AnimKeySpeed, 0f);
    }

    // 조이스틱 입력 즉시 반응
    if (IsJoystickActivated())
    {
        stateMachine.ChangeState(PlayerManualMove.Instance);
    }
}

public void OnIdleUpdate()
{
    // 정지 상태 체크
    if (isStop == true)
    {
        return;
    }

    // 1순위: 조이스틱 입력 → 수동 이동
    if (IsJoystickActivated())
    {
        stateMachine.ChangeState(PlayerManualMove.Instance);
        return;
    }

    // 2순위: 강제 이동
    if (forceMoveRemainTime > 0.0f)
    {
        stateMachine.ChangeState(PlayerForceMove.Instance);
        return;
    }

    // 3순위: 스킬 사용 가능 체크
    if (useMetaSkillInfo != null && isShowSkillMotion == false)
    {
        stateMachine.ChangeState(PlayerUseSkill.Instance);
        return;
    }

    // 4순위: 적 타겟 발견 → 이동/공격
    var nearEnemy = FindNearestEnemy();
    if (nearEnemy != null)
    {
        atkTarget = nearEnemy;

        // 대시 범위 내면 대시, 아니면 자동 이동
        if (IsInDashRange(atkTarget))
        {
            stateMachine.ChangeState(PlayerDash.Instance);
        }
        else
        {
            stateMachine.ChangeState(PlayerAutoMove.Instance);
        }
    }
}
```

**트레이드오프:**
- 상태 클래스 증가로 파일 수 증가 → 명확한 네이밍 규칙으로 관리
- 상태 간 공유 데이터 접근 복잡도 → Controller에 공유 데이터 집중

---

## 3. 네트워크 통신 안정성 및 보안 강화

### 결과
- 재시도 메커니즘으로 일시적 네트워크 끊김 자동 복구
- 암호화 통신 + 메모리 난독화로 보안 강화
- 사용자 경험 개선 (수동 재시도 불필요)

### 핵심 설계
- **3회 자동 재시도**: 타임아웃/에러 시 30초 대기 후 재시도
- **DhCrypto 암호화**: 요청/응답 데이터 암호화
- **ObscuredTypes**: 메모리상 민감 데이터 난독화

### 설계 배경
모바일 환경(지하철, 엘리베이터, 와이파이↔데이터 전환)에서 네트워크 불안정이 빈번했습니다. 또한 패킷 스니핑과 메모리 해킹에 대한 보안 요구사항이 있었습니다.

### 구현 상세

#### 3.1. 재시도 메커니즘

```csharp
public static async UniTask<bool> SendRequest(
    string url,
    object sendData = null,
    Action<NetworkReturnObject> onFinished = null,
    Action onConnectFail = null,
    bool debugLogUrl = true,
    bool isShowIndicator = true,
    int tryCount = 0,           // 현재 시도 횟수
    float connectTrySecond = 30f // 재시도 대기 시간
)
{
    tryCount++;
    bool isHideIndicator = false;

    var httpMethods = sendData != null ? HTTPMethods.Post : HTTPMethods.Get;
    var baseUrl = GetServerUrl();
    var request = new HTTPRequest(new Uri(baseUrl + url), httpMethods);

    if (httpMethods == HTTPMethods.Post)
    {
        // 데이터 암호화 후 전송
        var data = EncodeServerData(sendData);
        request.AddField("data", data);
    }

    request.Timeout = TimeSpan.FromSeconds(60f);

    try
    {
        // 로딩 인디케이터 표시
        if (isShowIndicator)
        {
            await IndicatorUtils.Create();
        }

        // 요청 전송
        await request.Send();

        // 인디케이터 숨김
        if (isShowIndicator)
        {
            await IndicatorUtils.HideIndicator();
            isHideIndicator = true;
        }

        // 타임아웃 체크
        if (request.IsTimedOut)
        {
            throw new Exception("TimeOut");
        }

        // 요청 상태 체크
        if (request.State == HTTPRequestStates.Aborted)
        {
            throw new Exception("Aborted");
        }

        if (request.State == HTTPRequestStates.Error)
        {
            throw new Exception("Error");
        }

        // 응답 데이터 복호화
        string result = request.Response.DataAsText;
        request.Dispose();

        if (!string.IsNullOrEmpty(result))
        {
            var jsonData = DecodeServerData(result);  // 암호화 해제
            var NRO = new NetworkReturnObject(jsonData);
            onFinished?.Invoke(NRO);
            return NRO.isSuccess;
        }

        return true;
    }
    catch (Exception e)
    {
        request?.Dispose();

        if (!isHideIndicator)
        {
            await IndicatorUtils.HideIndicator();
        }

        // 최대 재시도 횟수 도달 시
        if (MAX_TRY_COUNT <= tryCount)  // MAX_TRY_COUNT = 3
        {
            DebugUtils.LogError($"requestUrl : {url}\nErrorMessage : {e}");

            if (onConnectFail == null)
            {
                // 기본 실패 처리: 연결 실패 팝업
                Time.timeScale = 0;
                ConfirmPopup.CreatePopup(
                    "commonConnectFailDesc",
                    Application.Quit,
                    null,
                    UIUtils.DEPTH_70
                );
            }
            else
            {
                onConnectFail?.Invoke();
            }

            return false;
        }

        // 재시도 대기
        if (isShowIndicator)
            await IndicatorUtils.Create();

        await UniTask.Delay(TimeSpan.FromSeconds(connectTrySecond), true);

        if (isShowIndicator)
            await IndicatorUtils.HideIndicator();

        // 재귀 호출로 재시도 (tryCount 증가)
        return await SendRequest(
            url,
            sendData,
            onFinished,
            onConnectFail,
            debugLogUrl,
            true,
            tryCount,  // 증가된 tryCount 전달
            connectTrySecond
        );
    }
}
```

#### 3.2. 데이터 암호화 및 보안

```csharp
public static JsonData DecodeServerData(string result)
{
    // 커스텀 암호화 라이브러리 사용
    return JsonUtils.JsonDecode(
        DhCrypto.DecryptData(result, false)
    );
}

public static string EncodeServerData(object value)
{
    // JSON 직렬화 후 암호화
    return DhCrypto.EncryptData(
        JsonUtils.JsonEncode(value),
        false
    );
}
```

```csharp
public static class NetworkAPI
{
    public static JsonData getTableData;
    public static string playerId;

    // ObscuredInt로 메모리 해킹 방지
    public static ObscuredInt token;

    public static string GetDeviceId()
    {
        // 디바이스 고유 ID로 검증
        return SystemInfo.deviceUniqueIdentifier;
    }
}
```

**트레이드오프:**
- 재시도 대기 시간(30초)이 UX에 영향 → 로딩 인디케이터로 사용자에게 상태 전달
- 암호화 오버헤드 → 모바일 환경에서 무시할 수준의 지연

---

## 4. 대규모 코드베이스 관리 전략

### 결과
- 936개 스크립트, 370+ Utils/Manager/Info 클래스 체계적 관리
- 80+ 서버 테이블 자동 동기화로 네트워크 트래픽 **90%+ 감소**
- GC Zero Allocation 달성 (LinqGen, ZString)

### 핵심 설계
- **Manager-Utils-Info 3계층**: 단방향 의존성으로 복잡도 관리
- **ReactiveProperty 변경 감지**: 수동 추적 없이 자동 Dirty Flag
- **LinqGen/ZString**: LINQ, 문자열 연산의 GC 제거

### 설계 배경
936개 스크립트와 80+ 서버 테이블을 관리하면서 코드 중복, 의존성 복잡도, 네트워크 비효율 문제가 발생했습니다. 명확한 계층 분리와 자동화된 변경 감지 시스템으로 해결했습니다.

### 구현 상세

#### 4.1. Manager-Utils-Info 3계층 아키텍처

**아키텍처 구조:**

```
┌─────────────────────────────────────────────┐
│           Manager Layer (싱글톤)              │
│  - 게임 전역 상태 관리                         │
│  - 생명주기 관리 (Init, Update, Destroy)      │
│  - 예: StageManager, IAPManager              │
└─────────────────────────────────────────────┘
                    ↓ 호출
┌─────────────────────────────────────────────┐
│           Utils Layer (정적 메서드)            │
│  - 비즈니스 로직 구현                          │
│  - 순수 함수 중심 (상태 없음)                  │
│  - 예: SkillUtils, EquipUtils                │
└─────────────────────────────────────────────┘
                    ↓ 참조
┌─────────────────────────────────────────────┐
│           Info Layer (데이터)                 │
│  - MetaInfo: 게임 설정 데이터 (읽기 전용)      │
│  - ServerInfo: 플레이어 진행 데이터 (서버 동기화)│
│  - 예: MetaSkillInfo, SkillInfoManager       │
└─────────────────────────────────────────────┘
```

**코드 예제:** `SkillUtils.cs:10-19`

```csharp
public static class SkillUtils
{
    // MetaInfo 참조 (게임 설정 데이터)
    public static Dictionary<int, MetaSkillInfo> MetaInfos => MetaInfoManager.metaSkillInfos;

    // ServerInfo 참조 (플레이어 데이터)
    private static Dictionary<int, SkillInfo> ServerInfos => SkillInfoManager.Instance.CurrServerInfos;
    private static Dictionary<int, SkillEquipInfo> ServerEquipInfos => SkillEquipInfoManager.Instance.CurrServerInfos;

    // 비즈니스 로직: 스킬 추가
    public static void AddAmount(string dataId, double amount)
    {
        var metaInfo = FindMetaInfo(dataId);
        if (metaInfo == null)
        {
            DebugUtils.LogError($"metaInfo 찾기 실패, dataId = {dataId}");
            return;
        }

        if (IsOwn(metaInfo.id) == false)
        {
            // 새 스킬 생성
            var newSkillInfo = new SkillInfo(metaInfo.id);
            ServerInfos.Add(metaInfo.id, newSkillInfo);
            newSkillInfo.amount = amount - 1;
            newSkillInfo.level = 1;
            newSkillInfo.awakenLevel = 0;
        }
        else
        {
            // 기존 스킬 수량 증가
            ServerInfos[metaInfo.id].amount += amount;
        }

        // 이벤트 발행 (UI 갱신)
        EventUtils.TriggerEventListenerEndOfFrame(
            EventName.RefreshSkillInfo,
            metaInfo.id
        );
        EventUtils.TriggerEventListenerEndOfFrame(
            EventName.RefreshSkillRedDots
        );
    }
}
```

**설계 장점:**
- 단일 책임 원칙 준수, 의존성 명확화 (Info ← Utils ← Manager)
- Utils는 순수 함수로 단위 테스트 용이

#### 4.2. LinqGen/ZString GC 최적화

```csharp
using Cathei.LinqGen;  // LinqGen 라이브러리

public static List<MetaEquipmentInfo> FindEquipmentInfos(EquipType equipType)
{
    if (MetaEquipmentInfoDictionary.TryGetValue(equipType, out var value))
    {
        return value;
    }

    // 기존 LINQ: 할당 발생
    // var result = MetaEquipmentInfos.Where(x => x.type == equipType).ToList();

    // LinqGen: Zero Allocation
    MetaEquipmentInfoDictionary.TryAdd(
        equipType,
        MetaEquipmentInfos.Gen()
            .Where(x => x.type == equipType)
            .ToList()
    );

    return MetaEquipmentInfoDictionary[equipType];
}
```

```csharp
// ZString: UI 텍스트 Zero Allocation
public static string GetSkillLevelText(int skillId)
{
    return ZString.Format("Lv.{0}", GetLevel(skillId));
}
```

#### 4.3. 변경 데이터 자동 감지 및 동기화 시스템

80+ 테이블에서 변경된 데이터만 자동으로 감지하여 서버에 전송하는 시스템입니다.

```csharp
// ServerInfo 베이스 클래스
public abstract class ServerInfo
{
    public bool isChangedValue = false;  // 변경 플래그
    public abstract Dictionary<string, object> GetServerData();
}

// SkillInfo: ReactiveProperty로 자동 변경 감지
public class SkillInfo : ServerInfo
{
    private ReactiveProperty<ObscuredDouble> Amount = new(0);
    private ReactiveProperty<ObscuredInt> Level = new(0);
    private ReactiveProperty<ObscuredInt> AwakenLevel = new(0);

    public SkillInfo(JsonData data) : base(data)
    {
        skillId = data.GetInt(nameof(skillId));
        amount = data.GetDouble(nameof(amount));
        level = data.GetInt(nameof(level));
        awakenLevel = data.GetInt(nameof(awakenLevel));

        // 값 변경 시 자동으로 isChangedValue = true
        Amount.Skip(1).Subscribe(_ => isChangedValue = true);
        Level.Skip(1).Subscribe(_ => isChangedValue = true);
        AwakenLevel.Skip(1).Subscribe(_ => isChangedValue = true);
    }

    public override Dictionary<string, object> GetServerData()
    {
        isChangedValue = false;  // 전송 후 플래그 리셋
        return new()
        {
            { nameof(skillId), skillId },
            { nameof(amount), amount.GetDecrypted() },
            { nameof(level), level.GetDecrypted() },
            { nameof(awakenLevel), awakenLevel.GetDecrypted() },
        };
    }
}

// SkillInfoManager: 변경된 것만 필터링
public class SkillInfoManager : ServerManager<SkillInfoManager, SkillInfo>
{
    protected override string GetTableName => "User_SkillInfo";

    protected override object GetServerDatas()
    {
        // isChangedValue가 true인 것만 필터링
        var serverData = currServerInfos
            .Where(x => x.Value.isChangedValue)
            .Select(x => x.Value.GetServerData()).ToList();

        if (serverData.Count <= 0)
            return null;  // 변경 없으면 null 반환 (서버 전송 안 함)

        return new Dictionary<object, object>()
        {
            { "playerId", NetworkAPI.playerId },
            { "serverData", serverData }
        };
    }
}

// ServerManager: 자동 등록
public abstract class ServerManager<TServerManager, TServerInfo> : SingletonMonoBehaviour<TServerManager>
{
    protected override void Awake()
    {
        base.Awake();

        // 초기화 시 자동으로 ServerSaveUtils에 등록
        ServerSaveUtils.AddTableName(GetTableName);
        ServerSaveUtils.AddGetServerData(GetServerDatas);
    }
}

// ServerSaveUtils: 중앙 관리
public static class ServerSaveUtils
{
    private static List<Func<object>> callbackGetServerData = new();

    public static void AddGetServerData(Func<object> data)
    {
        if (callbackGetServerData.Contains(data))
            return;
        callbackGetServerData.Add(data);
    }

    public static void UpdateServerInfos(string url_name = url_common,
        Action<NetworkReturnObject> onSuccess = null)
    {
        var serverDataDictionary = new Dictionary<string, object>();

        // 등록된 모든 Manager의 GetServerDatas 호출
        for (int i = 0; i < callbackGetServerData.Count; i++)
        {
            var serverData = callbackGetServerData[i]?.Invoke();

            // null이 아닌 것만 추가 (변경된 데이터만)
            if (serverData != null)
            {
                serverDataDictionary.Add(tableNames[i], serverData);
            }
        }

        // 변경된 테이블만 서버로 전송
        if (serverDataDictionary.Count > 0)
        {
            await NetworkAPI.UpdateServerInfos(url_name, serverDataDictionary, onSuccess);
        }
    }
}

// 사용 예시
public static void AddAmount(string dataId, double amount)
{
    var metaInfo = FindMetaInfo(dataId);

    if (IsOwn(metaInfo.id) == false)
    {
        var newSkillInfo = new SkillInfo(metaInfo.id);
        ServerInfos.Add(metaInfo.id, newSkillInfo);
        newSkillInfo.amount = amount - 1;  // 이 시점에 isChangedValue = true
    }
    else
    {
        ServerInfos[metaInfo.id].amount += amount;  // 이 시점에 isChangedValue = true
    }

    // 변경된 SkillInfo만 자동으로 서버에 전송됨
    ServerSaveUtils.UpdateServerInfos();
}
```

**설계 장점:**
- **자동 변경 감지**: ReactiveProperty + Subscribe 패턴
- **선택적 전송**: isChangedValue가 true인 데이터만 전송
- **확장 용이**: ServerManager 상속만으로 자동 등록

**트레이드오프:**
- ReactiveProperty 오버헤드 → 변경 빈도가 낮아 무시할 수준
- 모든 필드를 ReactiveProperty로 감싸야 함 → 코드 생성 도구로 자동화 가능
