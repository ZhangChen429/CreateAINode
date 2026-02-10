# UE EQS 深度定制指南

## 一、扩展点架构总览

### 可扩展组件矩阵

| 组件 | 难度 | 蓝图支持 | 典型用途 | 定制价值 |
|------|------|----------|---------|---------|
| **Generator** | ⭐⭐ | ✅ | 定义"从哪里找候选" | ⭐⭐⭐⭐⭐ |
| **Test** | ⭐⭐ | ❌ | 定义"如何评分/过滤" | ⭐⭐⭐⭐⭐ |
| **Context** | ⭐ | ✅ | 定义"相对于什么" | ⭐⭐⭐ |
| **ItemType** | ⭐⭐⭐⭐ | ❌ | 定义"存储什么数据" | ⭐⭐ |

### 扩展层次图

```
┌─────────────────────────────────────────────────────────┐
│  应用层：游戏逻辑                                         │
│  - 行为树任务                                            │
│  - AI Controller                                         │
│  - 蓝图系统                                              │
└────────────────┬────────────────────────────────────────┘
                 │ 发起查询
                 ↓
┌─────────────────────────────────────────────────────────┐
│  扩展层：自定义组件（深度定制点）                         │
│  ┌─────────────┬─────────────┬─────────────┬──────────┐ │
│  │ Generator   │ Test        │ Context     │ ItemType │ │
│  │ (生成候选)  │ (评分过滤)  │ (参考对象)  │ (数据类型)│ │
│  └─────────────┴─────────────┴─────────────┴──────────┘ │
└────────────────┬────────────────────────────────────────┘
                 │ 调用引擎
                 ↓
┌─────────────────────────────────────────────────────────┐
│  引擎层：EQS 核心系统                                     │
│  - UEnvQueryManager (调度)                               │
│  - FEnvQueryInstance (执行)                              │
└─────────────────────────────────────────────────────────┘
```

---

## 二、Generator 深度定制

### 🎯 定制目标
**"从哪里生成候选项"**

### 📋 最小化重载接口

#### C++ 扩展
```cpp
UCLASS()
class UMyCustomGenerator : public UEnvQueryGenerator
{
    GENERATED_BODY()

public:
    UMyCustomGenerator()
    {
        // 1. 指定生成的项目类型
        ItemType = UEnvQueryItemType_Point::StaticClass();
        // 或 UEnvQueryItemType_Actor::StaticClass();

        // 2. 可选：性能配置
        bAutoSortTests = true;  // 自动排序测试以优化性能
        bCanRunAsync = false;   // 是否支持异步执行
    }

    // 核心重载：必须实现
    virtual void GenerateItems(FEnvQueryInstance& QueryInstance) const override
    {
        // 1. 获取查询上下文
        TArray<FVector> ContextLocations;
        QueryInstance.PrepareContext(GenerateAround, ContextLocations);

        // 2. 生成候选项（具体逻辑）
        TArray<FVector> CandidateLocations = YourGenerationLogic(ContextLocations);

        // 3. 添加到查询实例
        for (const FVector& Location : CandidateLocations)
        {
            QueryInstance.AddItemData<UEnvQueryItemType_Point>(FNavLocation(Location));
        }
    }

    // 配置属性：暴露给编辑器
    UPROPERTY(EditDefaultsOnly, Category = Generator)
    TSubclassOf<UEnvQueryContext> GenerateAround;

    UPROPERTY(EditDefaultsOnly, Category = Generator)
    FAIDataProviderFloatValue SearchRadius;
};
```

#### 蓝图扩展（快速原型）
```cpp
// 1. 创建蓝图类，继承自 EnvQueryGenerator_BlueprintBase

// 2. 在蓝图中实现事件（选择其一）：
//    - DoItemGeneration (从位置列表生成)
//    - DoItemGenerationFromActors (从Actor列表生成)

// 3. 在事件中调用：
//    - AddGeneratedVector(Vector) 或
//    - AddGeneratedActor(Actor)
```

### 🔧 核心接口详解

#### 1. PrepareContext - 获取上下文数据
```cpp
// 获取位置列表（如：玩家位置、目标位置）
TArray<FVector> ContextLocations;
if (!QueryInstance.PrepareContext(SearchCenter, ContextLocations))
    return;  // 上下文无效

// 获取 Actor 列表（如：所有敌人）
TArray<AActor*> ContextActors;
QueryInstance.PrepareContext(TargetActorContext, ContextActors);
```

**缓存机制**：同一 Context 在单次查询中只计算一次

#### 2. AddItemData - 添加候选项
```cpp
// 添加位置候选
QueryInstance.AddItemData<UEnvQueryItemType_Point>(FNavLocation(Vector));

// 添加 Actor 候选
QueryInstance.AddItemData<UEnvQueryItemType_Actor>(ActorPtr);

// 添加方向候选
QueryInstance.AddItemData<UEnvQueryItemType_Direction>(Rotator);
```

#### 3. FAIDataProvider - 动态参数系统
```cpp
// 配置时可以绑定到黑板或常量
UPROPERTY(EditDefaultsOnly, Category = Generator)
FAIDataProviderFloatValue SearchRadius;  // 编辑器配置为黑板键或常量

// 运行时获取值
void GenerateItems(FEnvQueryInstance& QueryInstance) const override
{
    UObject* QueryOwner = QueryInstance.Owner.Get();
    SearchRadius.BindData(QueryOwner, QueryInstance.QueryID);
    float Radius = SearchRadius.GetValue();  // 从黑板或常量读取
}
```

### 🚀 深度定制案例

#### 案例 1：智能对象生成器
**功能**：查询游戏中的交互对象（椅子、门、开关等）

```cpp
UCLASS(meta = (DisplayName = "Smart Objects"))
class UEnvQueryGenerator_SmartObjects : public UEnvQueryGenerator
{
    GENERATED_BODY()

public:
    UEnvQueryGenerator_SmartObjects()
    {
        ItemType = UEnvQueryItemType_Actor::StaticClass();
    }

    virtual void GenerateItems(FEnvQueryInstance& QueryInstance) const override
    {
        // 1. 获取查询中心
        TArray<FVector> ContextLocations;
        QueryInstance.PrepareContext(QueryOriginContext, ContextLocations);
        if (ContextLocations.Num() == 0) return;

        // 2. 构建查询请求
        FSmartObjectRequest Request;
        Request.Filter = SmartObjectRequestFilter;
        Request.QueryBox = FBox(ContextLocations[0] - QueryBoxExtent,
                                ContextLocations[0] + QueryBoxExtent);

        // 3. 查询智能对象子系统
        USmartObjectSubsystem* Subsystem = UWorld::GetSubsystem<USmartObjectSubsystem>(QueryInstance.World);
        TArray<FSmartObjectRequestResult> Results;
        Subsystem->FindSmartObjects(Request, Results);

        // 4. 过滤可声称的对象
        if (bOnlyClaimable)
        {
            Results = Results.FilterByPredicate([&](const FSmartObjectRequestResult& Result) {
                return Subsystem->CanBeClaimed(Result.SmartObjectHandle);
            });
        }

        // 5. 添加到候选列表
        for (const auto& Result : Results)
        {
            AActor* SmartObjectActor = Result.SmartObjectHandle.GetOwnerActor();
            QueryInstance.AddItemData<UEnvQueryItemType_Actor>(SmartObjectActor);
        }
    }

    UPROPERTY(EditAnywhere, Category = Generator)
    TSubclassOf<UEnvQueryContext> QueryOriginContext;

    UPROPERTY(EditAnywhere, Category = Generator)
    FSmartObjectRequestFilter SmartObjectRequestFilter;

    UPROPERTY(EditAnywhere, Category = Generator)
    FVector QueryBoxExtent = FVector(1000.0f);

    UPROPERTY(EditAnywhere, Category = Generator)
    bool bOnlyClaimable = true;
};
```

**定制效果**：
- ✅ AI 可以通过 EQS 查询附近的可交互对象
- ✅ 支持复杂的过滤条件（类型、标签、状态）
- ✅ 与智能对象系统无缝集成
- ✅ 自动处理可声称性检查

#### 案例 2：战术位置生成器
**功能**：生成高地、掩体、侧翼等战术位置

```cpp
UCLASS(meta = (DisplayName = "Tactical Positions"))
class UEnvQueryGenerator_TacticalPositions : public UEnvQueryGenerator
{
    GENERATED_BODY()

public:
    virtual void GenerateItems(FEnvQueryInstance& QueryInstance) const override
    {
        // 1. 获取战场中心和敌人位置
        TArray<FVector> EnemyLocations;
        QueryInstance.PrepareContext(EnemyContext, EnemyLocations);

        TArray<FVector> AllyLocations;
        QueryInstance.PrepareContext(AllyContext, AllyLocations);

        // 2. 生成候选位置
        TArray<FVector> CandidatePositions;

        switch (TacticalType)
        {
            case ETacticalType::HighGround:
                CandidatePositions = GenerateHighGroundPositions(EnemyLocations);
                break;

            case ETacticalType::Flanking:
                CandidatePositions = GenerateFlankingPositions(EnemyLocations, AllyLocations);
                break;

            case ETacticalType::Cover:
                CandidatePositions = GenerateCoverPositions(EnemyLocations);
                break;
        }

        // 3. 投影到导航网格
        UNavigationSystemV1* NavSys = UNavigationSystemV1::GetCurrent(QueryInstance.World);
        for (const FVector& Position : CandidatePositions)
        {
            FNavLocation NavLoc;
            if (NavSys->ProjectPointToNavigation(Position, NavLoc, ProjectExtent))
            {
                QueryInstance.AddItemData<UEnvQueryItemType_Point>(NavLoc);
            }
        }
    }

private:
    TArray<FVector> GenerateHighGroundPositions(const TArray<FVector>& EnemyLocations) const
    {
        // 射线向上检测高地
        // 返回海拔较高的位置
    }

    TArray<FVector> GenerateFlankingPositions(const TArray<FVector>& EnemyLocs,
                                                const TArray<FVector>& AllyLocs) const
    {
        // 计算敌人朝向，生成侧后方位置
    }

    UPROPERTY(EditAnywhere, Category = Generator)
    TEnumAsByte<ETacticalType> TacticalType;

    UPROPERTY(EditAnywhere, Category = Generator)
    TSubclassOf<UEnvQueryContext> EnemyContext;

    UPROPERTY(EditAnywhere, Category = Generator)
    TSubclassOf<UEnvQueryContext> AllyContext;

    UPROPERTY(EditAnywhere, Category = Generator)
    FVector ProjectExtent = FVector(500.0f, 500.0f, 500.0f);
};
```

**定制效果**：
- ✅ 生成符合战术逻辑的位置（非简单网格）
- ✅ 考虑地形高度、敌我位置关系
- ✅ 支持多种战术模式切换
- ✅ 自动导航网格验证

#### 案例 3：动态障碍物避让生成器
**功能**：生成避开移动障碍物的路径点

```cpp
UCLASS(meta = (DisplayName = "Dynamic Obstacle Avoidance"))
class UEnvQueryGenerator_DynamicAvoidance : public UEnvQueryGenerator
{
    virtual void GenerateItems(FEnvQueryInstance& QueryInstance) const override
    {
        // 1. 获取起点和目标
        TArray<FVector> StartLocations, GoalLocations;
        QueryInstance.PrepareContext(StartContext, StartLocations);
        QueryInstance.PrepareContext(GoalContext, GoalLocations);

        // 2. 获取当前所有动态障碍物
        TArray<AActor*> DynamicObstacles;
        UGameplayStatics::GetAllActorsWithTag(QueryInstance.World,
                                               "DynamicObstacle",
                                               DynamicObstacles);

        // 3. 预测障碍物未来位置（考虑速度）
        TArray<FVector> PredictedObstaclePositions;
        for (AActor* Obstacle : DynamicObstacles)
        {
            FVector Velocity = Obstacle->GetVelocity();
            FVector PredictedPos = Obstacle->GetActorLocation() + Velocity * PredictionTime;
            PredictedObstaclePositions.Add(PredictedPos);
        }

        // 4. 生成避让路径点
        TArray<FVector> AvoidancePoints = GenerateAvoidanceWaypoints(
            StartLocations[0],
            GoalLocations[0],
            PredictedObstaclePositions
        );

        // 5. 添加候选
        for (const FVector& Point : AvoidancePoints)
        {
            QueryInstance.AddItemData<UEnvQueryItemType_Point>(FNavLocation(Point));
        }
    }

    UPROPERTY(EditAnywhere, Category = Generator)
    float PredictionTime = 2.0f;  // 预测未来 2 秒的障碍物位置
};
```

**定制效果**：
- ✅ 考虑动态障碍物的运动趋势
- ✅ 生成时间上最优的避让路径
- ✅ 适用于人群、载具、飞行物等场景

---

## 三、Test 深度定制

### 🎯 定制目标
**"如何评分和过滤候选项"**

### 📋 最小化重载接口

```cpp
UCLASS()
class UMyCustomTest : public UEnvQueryTest
{
    GENERATED_BODY()

public:
    UMyCustomTest()
    {
        // 1. 指定兼容的项目类型
        ValidItemType = UEnvQueryItemType_Point::StaticClass();

        // 2. 设置默认测试目的
        TestPurpose = EEnvTestPurpose::Score;  // Filter / Score / FilterAndScore

        // 3. 设置成本（影响排序优化）
        Cost = EEnvTestCost::Low;  // Low / Medium / High
    }

    // 核心重载：必须实现
    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        // 1. 获取查询所有者（用于黑板绑定）
        UObject* QueryOwner = QueryInstance.Owner.Get();
        if (!QueryOwner) return;

        // 2. 绑定数据提供者参数
        FloatValueMin.BindData(QueryOwner, QueryInstance.QueryID);
        float MinThreshold = FloatValueMin.GetValue();

        // 3. 获取上下文数据（如果需要）
        TArray<FVector> ContextLocations;
        QueryInstance.PrepareContext(TestContext, ContextLocations);

        // 4. 遍历所有候选项并评分
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            // 获取当前项的数据
            const FVector ItemLocation = GetItemLocation(QueryInstance, It.GetIndex());

            // 计算测试值
            float TestValue = CalculateYourMetric(ItemLocation, ContextLocations);

            // 设置分数（自动处理过滤和评分逻辑）
            It.SetScore(TestPurpose, FilterType, TestValue, MinThreshold, MaxThreshold);
        }
    }

    // 可选重载：编辑器显示
    virtual FText GetDescriptionTitle() const override
    {
        return FText::FromString("My Custom Test");
    }

    virtual FText GetDescriptionDetails() const override
    {
        return FText::Format(
            FText::FromString("Tests {0} against threshold {1}"),
            FText::FromName(TestContext->GetFName()),
            FText::AsNumber(FloatValueMin.DefaultValue)
        );
    }

    // 配置属性
    UPROPERTY(EditDefaultsOnly, Category = Test)
    TSubclassOf<UEnvQueryContext> TestContext;
};
```

### 🔧 核心接口详解

#### 1. ItemIterator - 遍历候选项
```cpp
for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
{
    // 自动跳过已被过滤掉的项目
    int32 ItemIndex = It.GetIndex();

    // 获取项目数据
    FVector Location = GetItemLocation(QueryInstance, ItemIndex);
    AActor* Actor = GetItemActor(QueryInstance, ItemIndex);

    // 评分
    It.SetScore(...);
}
```

#### 2. SetScore - 评分和过滤
```cpp
// 自动处理过滤和评分逻辑
It.SetScore(
    TestPurpose,        // Filter / Score / FilterAndScore
    FilterType,         // Match / Range
    TestValue,          // 计算出的值
    MinThreshold,       // 最小阈值
    MaxThreshold        // 最大阈值
);

// 内部逻辑：
// 1. 如果是 Filter 模式：
//    - 值不在 [Min, Max] 范围内 → 标记为 Discarded
// 2. 如果是 Score 模式：
//    - 根据 ScoringEquation 计算分数
//    - Score = f(TestValue, Min, Max) * ScoringFactor
// 3. 如果是 FilterAndScore：
//    - 先过滤，再评分
```

#### 3. ScoringEquation - 评分公式
```cpp
// 在配置中设置
UPROPERTY(EditDefaultsOnly, Category = Score)
TEnumAsByte<EEnvTestScoreEquation::Type> ScoringEquation;

// 可选值：
// - Linear: (Value - Min) / (Max - Min)
// - Square: ((Value - Min) / (Max - Min))^2
// - InverseLinear: (Max - Value) / (Max - Min)
// - SquareRoot: sqrt((Value - Min) / (Max - Min))
// - Constant: 固定值

// 最终分数 = Equation(Value) * ScoringFactor
```

### 🚀 深度定制案例

#### 案例 1：可见性测试（考虑遮挡和 FOV）
**功能**：测试候选位置是否在视野内且不被遮挡

```cpp
UCLASS(meta = (DisplayName = "Advanced Visibility"))
class UEnvQueryTest_AdvancedVisibility : public UEnvQueryTest
{
    GENERATED_BODY()

public:
    UEnvQueryTest_AdvancedVisibility()
    {
        ValidItemType = UEnvQueryItemType_Point::StaticClass();
        Cost = EEnvTestCost::High;  // 射线检测成本高
    }

    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        UObject* QueryOwner = QueryInstance.Owner.Get();
        AActor* QuerierActor = Cast<AActor>(QueryOwner);
        if (!QuerierActor) return;

        // 获取观察者数据
        TArray<AActor*> ObserverActors;
        QueryInstance.PrepareContext(ObserverContext, ObserverActors);

        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            const FVector ItemLocation = GetItemLocation(QueryInstance, It.GetIndex());

            float BestVisibilityScore = 0.0f;

            for (AActor* Observer : ObserverActors)
            {
                FVector ObserverLocation = Observer->GetActorLocation();
                FVector ObserverForward = Observer->GetActorForwardVector();

                // 1. 检查是否在视野角度内
                FVector ToItem = (ItemLocation - ObserverLocation).GetSafeNormal();
                float DotProduct = FVector::DotProduct(ObserverForward, ToItem);
                float AngleDegrees = FMath::RadiansToDegrees(FMath::Acos(DotProduct));

                if (AngleDegrees > FOVAngle)
                    continue;  // 超出视野角度

                // 2. 射线检测遮挡
                FHitResult HitResult;
                FCollisionQueryParams Params;
                Params.AddIgnoredActor(Observer);

                bool bHit = QueryInstance.World->LineTraceSingleByChannel(
                    HitResult,
                    ObserverLocation,
                    ItemLocation,
                    ECC_Visibility,
                    Params
                );

                // 3. 计算可见性分数
                float VisibilityScore = 0.0f;
                if (!bHit || HitResult.Distance > (ItemLocation - ObserverLocation).Size() - 10.0f)
                {
                    // 未遮挡，根据角度和距离计算分数
                    float AngleScore = 1.0f - (AngleDegrees / FOVAngle);
                    float DistanceScore = 1.0f - FMath::Clamp(HitResult.Distance / MaxVisibleDistance, 0.0f, 1.0f);
                    VisibilityScore = AngleScore * 0.6f + DistanceScore * 0.4f;
                }

                BestVisibilityScore = FMath::Max(BestVisibilityScore, VisibilityScore);
            }

            // 设置分数
            It.SetScore(TestPurpose, FilterType, BestVisibilityScore, 0.0f, 1.0f);
        }
    }

    UPROPERTY(EditAnywhere, Category = Test)
    TSubclassOf<UEnvQueryContext> ObserverContext;

    UPROPERTY(EditAnywhere, Category = Test)
    float FOVAngle = 90.0f;

    UPROPERTY(EditAnywhere, Category = Test)
    float MaxVisibleDistance = 2000.0f;
};
```

**定制效果**：
- ✅ 综合考虑视野角度、遮挡、距离
- ✅ 支持多观察者（取最佳可见性）
- ✅ 可用于潜行游戏的隐蔽位置查询
- ✅ 可用于敌人视野分析

#### 案例 2：声音传播测试
**功能**：考虑墙壁和材质的声音衰减

```cpp
UCLASS(meta = (DisplayName = "Sound Propagation"))
class UEnvQueryTest_SoundPropagation : public UEnvQueryTest
{
    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        TArray<FVector> SoundSourceLocations;
        QueryInstance.PrepareContext(SoundSourceContext, SoundSourceLocations);

        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            const FVector ItemLocation = GetItemLocation(QueryInstance, It.GetIndex());

            float MaxAudibilityScore = 0.0f;

            for (const FVector& SourceLocation : SoundSourceLocations)
            {
                // 1. 计算直线距离衰减
                float Distance = FVector::Dist(ItemLocation, SourceLocation);
                float DistanceAttenuation = FMath::Exp(-Distance / AttenuationDistance);

                // 2. 射线检测穿过的墙壁数量
                TArray<FHitResult> HitResults;
                FCollisionQueryParams Params;
                QueryInstance.World->LineTraceMultiByChannel(
                    HitResults,
                    SourceLocation,
                    ItemLocation,
                    ECC_Visibility,
                    Params
                );

                // 3. 计算墙壁衰减
                float WallAttenuation = 1.0f;
                for (const FHitResult& Hit : HitResults)
                {
                    if (Hit.GetActor() && Hit.GetActor()->ActorHasTag("Wall"))
                    {
                        // 不同材质不同衰减
                        UPhysicalMaterial* PhysMat = Hit.PhysMaterial.Get();
                        float MaterialAttenuation = GetMaterialAttenuation(PhysMat);
                        WallAttenuation *= MaterialAttenuation;
                    }
                }

                // 4. 综合评分
                float AudibilityScore = DistanceAttenuation * WallAttenuation;
                MaxAudibilityScore = FMath::Max(MaxAudibilityScore, AudibilityScore);
            }

            It.SetScore(TestPurpose, FilterType, MaxAudibilityScore, 0.0f, 1.0f);
        }
    }

private:
    float GetMaterialAttenuation(UPhysicalMaterial* Material) const
    {
        if (!Material) return 0.5f;
        // 根据材质类型返回不同衰减系数
        // 混凝土: 0.3, 木头: 0.6, 玻璃: 0.8
    }

    UPROPERTY(EditAnywhere, Category = Test)
    float AttenuationDistance = 1000.0f;
};
```

**定制效果**：
- ✅ AI 可以找到"听不到声音"的安全位置
- ✅ 考虑物理材质对声音的影响
- ✅ 适用于潜行和声音捕食者游戏

#### 案例 3：团队协作测试
**功能**：评估位置对整个团队的战术价值

```cpp
UCLASS(meta = (DisplayName = "Team Tactical Value"))
class UEnvQueryTest_TeamTacticalValue : public UEnvQueryTest
{
    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        // 获取队友位置
        TArray<AActor*> TeamMates;
        QueryInstance.PrepareContext(TeamMatesContext, TeamMates);

        // 获取敌人位置
        TArray<AActor*> Enemies;
        QueryInstance.PrepareContext(EnemiesContext, Enemies);

        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            const FVector ItemLocation = GetItemLocation(QueryInstance, It.GetIndex());

            // 1. 计算支援价值（能支援多少队友）
            int32 SupportableTeamMates = 0;
            for (AActor* TeamMate : TeamMates)
            {
                float Distance = FVector::Dist(ItemLocation, TeamMate->GetActorLocation());
                if (Distance <= SupportRange)
                    SupportableTeamMates++;
            }
            float SupportScore = float(SupportableTeamMates) / FMath::Max(TeamMates.Num(), 1);

            // 2. 计算交叉火力价值（与队友形成交叉角度）
            float CrossfireScore = 0.0f;
            if (Enemies.Num() > 0 && TeamMates.Num() > 0)
            {
                FVector ToEnemy = (Enemies[0]->GetActorLocation() - ItemLocation).GetSafeNormal();
                for (AActor* TeamMate : TeamMates)
                {
                    FVector TeamMateToEnemy = (Enemies[0]->GetActorLocation() - TeamMate->GetActorLocation()).GetSafeNormal();
                    float Angle = FMath::RadiansToDegrees(FMath::Acos(FVector::DotProduct(ToEnemy, TeamMateToEnemy)));
                    if (Angle > 45.0f && Angle < 135.0f)  // 理想交叉角度
                        CrossfireScore += 1.0f;
                }
                CrossfireScore /= TeamMates.Num();
            }

            // 3. 计算分散价值（避免扎堆）
            float DispersionScore = 1.0f;
            for (AActor* TeamMate : TeamMates)
            {
                float Distance = FVector::Dist(ItemLocation, TeamMate->GetActorLocation());
                if (Distance < MinDispersionDistance)
                    DispersionScore *= 0.5f;
            }

            // 4. 综合评分
            float TacticalValue = SupportScore * 0.4f + CrossfireScore * 0.4f + DispersionScore * 0.2f;

            It.SetScore(TestPurpose, FilterType, TacticalValue, 0.0f, 1.0f);
        }
    }

    UPROPERTY(EditAnywhere, Category = Test)
    float SupportRange = 1000.0f;

    UPROPERTY(EditAnywhere, Category = Test)
    float MinDispersionDistance = 500.0f;
};
```

**定制效果**：
- ✅ AI 团队能够自动形成战术阵型
- ✅ 避免 AI 扎堆
- ✅ 自动形成交叉火力
- ✅ 支持团队协同游戏机制

#### 案例 4：路径成本测试（A* 启发式）
**功能**：评估到达候选位置的实际路径成本

```cpp
UCLASS(meta = (DisplayName = "Path Cost"))
class UEnvQueryTest_PathCost : public UEnvQueryTest
{
    UEnvQueryTest_PathCost()
    {
        Cost = EEnvTestCost::High;  // 寻路成本高
        bCanRunAsync = true;        // 支持异步执行
    }

    virtual void RunTest(FEnvQueryInstance& QueryInstance) const override
    {
        TArray<FVector> StartLocations;
        QueryInstance.PrepareContext(PathStartContext, StartLocations);
        if (StartLocations.Num() == 0) return;

        UNavigationSystemV1* NavSys = UNavigationSystemV1::GetCurrent(QueryInstance.World);
        if (!NavSys) return;

        // 批量寻路请求（异步）
        TArray<FPathFindingQuery> Queries;
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            const FVector ItemLocation = GetItemLocation(QueryInstance, It.GetIndex());

            FPathFindingQuery Query;
            Query.StartLocation = StartLocations[0];
            Query.EndLocation = ItemLocation;
            Query.NavAgentProperties = NavAgentProperties;

            Queries.Add(Query);
        }

        // 批量执行（可异步）
        TArray<FPathFindingResult> Results;
        NavSys->FindPathBatch(Queries, Results);

        // 评分
        int32 ResultIndex = 0;
        for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
        {
            if (Results.IsValidIndex(ResultIndex))
            {
                const FPathFindingResult& Result = Results[ResultIndex];

                float PathCost = 0.0f;
                if (Result.IsSuccessful())
                {
                    PathCost = Result.Path->GetLength();

                    // 可选：考虑路径危险度
                    if (bConsiderPathDanger)
                    {
                        float DangerPenalty = CalculatePathDanger(Result.Path);
                        PathCost += DangerPenalty;
                    }
                }
                else
                {
                    PathCost = BIG_NUMBER;  // 不可达
                }

                It.SetScore(TestPurpose, FilterType, PathCost, 0.0f, MaxAcceptablePathCost);
            }
            ResultIndex++;
        }
    }

private:
    float CalculatePathDanger(const FNavigationPath* Path) const
    {
        // 检测路径是否经过危险区域（火焰、毒气等）
        float Danger = 0.0f;
        for (const FNavPathPoint& Point : Path->GetPathPoints())
        {
            // 检测危险区域重叠
        }
        return Danger;
    }

    UPROPERTY(EditAnywhere, Category = Test)
    bool bConsiderPathDanger = false;

    UPROPERTY(EditAnywhere, Category = Test)
    float MaxAcceptablePathCost = 5000.0f;

    UPROPERTY(EditAnywhere, Category = Test)
    FNavAgentProperties NavAgentProperties;
};
```

**定制效果**：
- ✅ 选择实际最短路径而非直线距离
- ✅ 考虑导航网格的实际连通性
- ✅ 可扩展到考虑路径危险度
- ✅ 支持异步批量寻路

---

## 四、Context 深度定制

### 🎯 定制目标
**"提供查询的参考对象"**

### 📋 最小化重载接口

#### C++ 扩展
```cpp
UCLASS()
class UMyCustomContext : public UEnvQueryContext
{
    GENERATED_BODY()

public:
    // 核心重载：必须实现
    virtual void ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const override
    {
        // 1. 获取查询发起者
        UObject* QueryOwner = QueryInstance.Owner.Get();
        AActor* QuerierActor = Cast<AActor>(QueryOwner);
        if (!QuerierActor) return;

        // 2. 计算/查询上下文数据
        AActor* TargetActor = FindTargetActor(QuerierActor);

        // 3. 填充上下文数据（选择其一）
        // 单个 Actor
        UEnvQueryItemType_Actor::SetContextHelper(ContextData, TargetActor);

        // 或多个 Actor
        TArray<AActor*> Actors = FindMultipleActors(QuerierActor);
        UEnvQueryItemType_Actor::SetContextHelper(ContextData, Actors);

        // 或单个位置
        FVector Location = CalculateLocation(QuerierActor);
        UEnvQueryItemType_Point::SetContextHelper(ContextData, Location);

        // 或多个位置
        TArray<FVector> Locations = CalculateLocations(QuerierActor);
        UEnvQueryItemType_Point::SetContextHelper(ContextData, Locations);
    }
};
```

#### 蓝图扩展
```cpp
// 1. 创建蓝图类，继承自 EnvQueryContext_BlueprintBase

// 2. 实现事件（选择其一）：
//    - ProvideSingleActor
//    - ProvideSingleLocation
//    - ProvideActorsSet
//    - ProvideLocationsSet

// 3. 返回上下文数据
```

### 🚀 深度定制案例

#### 案例 1：预测未来位置上下文
**功能**：提供目标 N 秒后的预测位置

```cpp
UCLASS(meta = (DisplayName = "Predicted Future Location"))
class UEnvQueryContext_PredictedLocation : public UEnvQueryContext
{
    GENERATED_BODY()

public:
    virtual void ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const override
    {
        UObject* QueryOwner = QueryInstance.Owner.Get();
        AAIController* AIController = Cast<AAIController>(QueryOwner);
        if (!AIController) return;

        // 获取黑板中的目标
        UBlackboardComponent* Blackboard = AIController->GetBlackboardComponent();
        AActor* Target = Cast<AActor>(Blackboard->GetValueAsObject(TargetKey.SelectedKeyName));
        if (!Target) return;

        // 获取目标当前位置和速度
        FVector CurrentLocation = Target->GetActorLocation();
        FVector Velocity = Target->GetVelocity();

        // 预测未来位置
        FVector PredictedLocation = CurrentLocation + Velocity * PredictionTime;

        // 可选：投影到导航网格
        if (bProjectToNavigation)
        {
            UNavigationSystemV1* NavSys = UNavigationSystemV1::GetCurrent(QueryInstance.World);
            FNavLocation NavLoc;
            if (NavSys->ProjectPointToNavigation(PredictedLocation, NavLoc, ProjectExtent))
            {
                PredictedLocation = NavLoc.Location;
            }
        }

        UEnvQueryItemType_Point::SetContextHelper(ContextData, PredictedLocation);
    }

    UPROPERTY(EditAnywhere, Category = Context)
    FBlackboardKeySelector TargetKey;

    UPROPERTY(EditAnywhere, Category = Context)
    float PredictionTime = 2.0f;

    UPROPERTY(EditAnywhere, Category = Context)
    bool bProjectToNavigation = true;

    UPROPERTY(EditAnywhere, Category = Context)
    FVector ProjectExtent = FVector(500.0f);
};
```

**定制效果**：
- ✅ AI 可以拦截移动目标
- ✅ 提前占据目标的逃跑路线
- ✅ 适用于追击和伏击

#### 案例 2：团队重心上下文
**功能**：提供队友的几何中心或加权中心

```cpp
UCLASS(meta = (DisplayName = "Team Centroid"))
class UEnvQueryContext_TeamCentroid : public UEnvQueryContext
{
    virtual void ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const override
    {
        UObject* QueryOwner = QueryInstance.Owner.Get();
        AAIController* AIController = Cast<AAIController>(QueryOwner);
        if (!AIController) return;

        // 获取所有队友
        TArray<AActor*> TeamMates = GetTeamMates(AIController);
        if (TeamMates.Num() == 0) return;

        FVector Centroid = FVector::ZeroVector;
        float TotalWeight = 0.0f;

        for (AActor* TeamMate : TeamMates)
        {
            float Weight = 1.0f;

            // 可选：根据角色重要性加权
            if (bWeightByImportance)
            {
                if (ICommanderInterface* Commander = Cast<ICommanderInterface>(TeamMate))
                {
                    Weight = 2.0f;  // 指挥官权重更高
                }
            }

            // 可选：根据距离加权（近的队友权重更高）
            if (bWeightByDistance)
            {
                float Distance = FVector::Dist(AIController->GetPawn()->GetActorLocation(),
                                                 TeamMate->GetActorLocation());
                Weight *= FMath::Exp(-Distance / DistanceDecay);
            }

            Centroid += TeamMate->GetActorLocation() * Weight;
            TotalWeight += Weight;
        }

        if (TotalWeight > 0.0f)
        {
            Centroid /= TotalWeight;
        }

        UEnvQueryItemType_Point::SetContextHelper(ContextData, Centroid);
    }

    UPROPERTY(EditAnywhere, Category = Context)
    bool bWeightByImportance = false;

    UPROPERTY(EditAnywhere, Category = Context)
    bool bWeightByDistance = false;

    UPROPERTY(EditAnywhere, Category = Context)
    float DistanceDecay = 1000.0f;
};
```

**定制效果**：
- ✅ AI 可以围绕团队重心行动
- ✅ 支持指挥官、医疗兵等角色的重要性权重
- ✅ 适用于团队协同战术

#### 案例 3：威胁区域上下文
**功能**：提供所有危险区域的中心点

```cpp
UCLASS(meta = (DisplayName = "Threat Zones"))
class UEnvQueryContext_ThreatZones : public UEnvQueryContext
{
    virtual void ProvideContext(FEnvQueryInstance& QueryInstance,
                                 FEnvQueryContextData& ContextData) const override
    {
        TArray<FVector> ThreatLocations;

        // 1. 收集所有敌人位置
        TArray<AActor*> Enemies;
        UGameplayStatics::GetAllActorsWithTag(QueryInstance.World, "Enemy", Enemies);
        for (AActor* Enemy : Enemies)
        {
            ThreatLocations.Add(Enemy->GetActorLocation());
        }

        // 2. 收集环境危险区域（火焰、毒气等）
        TArray<AActor*> HazardZones;
        UGameplayStatics::GetAllActorsOfClass(QueryInstance.World, AHazardZone::StaticClass(), HazardZones);
        for (AActor* Hazard : HazardZones)
        {
            ThreatLocations.Add(Hazard->GetActorLocation());
        }

        // 3. 收集投掷物（手榴弹、炸弹）
        TArray<AActor*> Projectiles;
        UGameplayStatics::GetAllActorsOfClass(QueryInstance.World, AProjectile::StaticClass(), Projectiles);
        for (AActor* Projectile : Projectiles)
        {
            // 预测落点
            if (AProjectile* Proj = Cast<AProjectile>(Projectile))
            {
                FVector ImpactPoint = Proj->GetPredictedImpactPoint();
                ThreatLocations.Add(ImpactPoint);
            }
        }

        UEnvQueryItemType_Point::SetContextHelper(ContextData, ThreatLocations);
    }
};
```

**定制效果**：
- ✅ AI 可以规避多种威胁
- ✅ 自动识别环境危险
- ✅ 适用于生存和逃生场景

---

## 五、ItemType 深度定制（高级）

### 🎯 定制目标
**"存储自定义数据类型"**

### ⚠️ 难度说明
ItemType 扩展难度最高，需要深入理解类型擦除和内存管理，通常只在以下场景需要：
- 存储复杂的自定义数据结构
- 需要特殊的黑板集成
- 需要自定义的序列化逻辑

### 📋 最小化重载接口

```cpp
UCLASS()
class UEnvQueryItemType_CustomData : public UEnvQueryItemType
{
    GENERATED_BODY()

public:
    // 1. 定义存储的数据类型
    struct FCustomData
    {
        FVector Location;
        float Priority;
        FGameplayTag Tag;
        // ... 自定义字段
    };

    typedef const FCustomData& FValueType;

    // 2. 构造器中设置大小
    UEnvQueryItemType_CustomData()
    {
        ValueSize = sizeof(FCustomData);
    }

    // 3. 实现静态访问方法
    static FCustomData GetValue(const uint8* RawData)
    {
        return *((FCustomData*)RawData);
    }

    static void SetValue(uint8* RawData, const FCustomData& Value)
    {
        *((FCustomData*)RawData) = Value;
    }

    // 4. 实现上下文辅助方法
    static void SetContextHelper(FEnvQueryContextData& ContextData, const FCustomData& SingleData)
    {
        ContextData.ValueType = UEnvQueryItemType_CustomData::StaticClass();
        ContextData.NumValues = 1;
        ContextData.RawData.SetNumUninitialized(sizeof(FCustomData));
        SetValue(ContextData.RawData.GetData(), SingleData);
    }

    static void SetContextHelper(FEnvQueryContextData& ContextData, const TArray<FCustomData>& MultipleData)
    {
        ContextData.ValueType = UEnvQueryItemType_CustomData::StaticClass();
        ContextData.NumValues = MultipleData.Num();
        ContextData.RawData.SetNumUninitialized(sizeof(FCustomData) * MultipleData.Num());

        for (int32 i = 0; i < MultipleData.Num(); i++)
        {
            SetValue(ContextData.RawData.GetData() + i * sizeof(FCustomData), MultipleData[i]);
        }
    }

    // 5. 可选：黑板集成
    virtual void AddBlackboardFilters(FBlackboardKeySelector& KeySelector, UObject* FilterOwner) const override
    {
        // 限制可选的黑板键类型
        KeySelector.AddVectorFilter(FilterOwner, FName("Location"));
    }

    virtual bool StoreInBlackboard(FBlackboardKeySelector& KeySelector,
                                     UBlackboardComponent* Blackboard,
                                     const uint8* RawData) const override
    {
        const FCustomData& Data = GetValue(RawData);
        Blackboard->SetValueAsVector(KeySelector.SelectedKeyName, Data.Location);
        return true;
    }

    // 6. 可选：调试显示
    virtual FString GetDescription(const uint8* RawData) const override
    {
        const FCustomData& Data = GetValue(RawData);
        return FString::Printf(TEXT("Loc: %s, Priority: %.2f, Tag: %s"),
                               *Data.Location.ToString(),
                               Data.Priority,
                               *Data.Tag.ToString());
    }
};
```

### 🚀 深度定制案例

#### 案例：复合战术数据类型
**功能**：存储位置 + 战术属性的组合数据

```cpp
UCLASS()
class UEnvQueryItemType_TacticalPosition : public UEnvQueryItemType
{
    struct FTacticalPosition
    {
        FNavLocation Location;          // 位置
        float CoverQuality;             // 掩体质量 [0-1]
        float ElevationAdvantage;       // 高度优势 [米]
        TArray<FVector> FiringAngles;   // 可射击方向
        bool bHasEscapeRoute;           // 是否有逃生路线
        float TeamSupportValue;         // 团队支援价值
    };

    typedef const FTacticalPosition& FValueType;

    UEnvQueryItemType_TacticalPosition()
    {
        ValueSize = sizeof(FTacticalPosition);
    }

    static FTacticalPosition GetValue(const uint8* RawData)
    {
        return *((FTacticalPosition*)RawData);
    }

    static void SetValue(uint8* RawData, const FTacticalPosition& Value)
    {
        *((FTacticalPosition*)RawData) = Value;
    }

    // ... SetContextHelper 实现

    // 重载位置访问（用于内置 Test）
    virtual FVector GetItemLocation(const uint8* RawData) const override
    {
        const FTacticalPosition& Data = GetValue(RawData);
        return Data.Location.Location;
    }
};
```

**使用示例**：
```cpp
// Generator 中生成
FTacticalPosition TacPos;
TacPos.Location = FNavLocation(Position);
TacPos.CoverQuality = CalculateCoverQuality(Position);
TacPos.ElevationAdvantage = CalculateElevation(Position);
// ...
QueryInstance.AddItemData<UEnvQueryItemType_TacticalPosition>(TacPos);

// Test 中访问
for (FEnvQueryInstance::ItemIterator It(this, QueryInstance); It; ++It)
{
    const uint8* RawData = QueryInstance.RawData.GetData() + QueryInstance.Items[It.GetIndex()].DataOffset;
    const FTacticalPosition& TacPos = UEnvQueryItemType_TacticalPosition::GetValue(RawData);

    float Score = TacPos.CoverQuality * 0.5f + TacPos.TeamSupportValue * 0.5f;
    It.SetScore(...);
}
```

**定制效果**：
- ✅ Generator 可以预计算复杂属性
- ✅ Test 直接使用预计算数据，提升性能
- ✅ 避免重复计算相同的指标
- ✅ 支持多维度的战术评估

---

## 六、深度定制效果总结

### 按游戏类型分类

#### 🎮 潜行游戏
| 定制组件 | 功能 | 效果 |
|---------|------|------|
| **Generator** | 阴影区域生成器 | 生成视线盲区 |
| **Test** | 噪音传播测试 | 评估声音暴露风险 |
| **Test** | 视野锥检测测试 | 避开守卫视线 |
| **Context** | 巡逻路线上下文 | 预判守卫移动 |

#### 🔫 战术射击游戏
| 定制组件 | 功能 | 效果 |
|---------|------|------|
| **Generator** | 战术位置生成器 | 生成高地、掩体、侧翼位置 |
| **Test** | 交叉火力测试 | 团队形成火力网 |
| **Test** | 弹药补给距离测试 | 考虑补给便利性 |
| **Context** | 火力覆盖区上下文 | 规避敌方火力区 |

#### 🧟 生存游戏
| 定制组件 | 功能 | 效果 |
|---------|------|------|
| **Generator** | 资源点生成器 | 查询食物、水源、物资 |
| **Test** | 多威胁评估测试 | 综合评估僵尸、辐射、饥饿风险 |
| **Test** | 庇护所质量测试 | 评估避难所安全性 |
| **Context** | 威胁密度上下文 | 提供危险区域分布 |

#### 🏎️ 竞速/追逐游戏
| 定制组件 | 功能 | 效果 |
|---------|------|------|
| **Generator** | 最优路线生成器 | 生成赛道最优走线 |
| **Test** | 弯道速度测试 | 评估过弯极限速度 |
| **Test** | 超车机会测试 | 识别超车窗口期 |
| **Context** | 前车预测位置 | 预判对手行动 |

#### 🐉 开放世界 RPG
| 定制组件 | 功能 | 效果 |
|---------|------|------|
| **Generator** | 兴趣点生成器 | 查询任务、商店、事件 |
| **Test** | 等级适配测试 | 推荐适合玩家等级的内容 |
| **Test** | 任务链相关性测试 | 推荐相关任务 |
| **Context** | 玩家兴趣上下文 | 基于玩家行为推荐 |

### 性能优化效果

| 优化技术 | 实现方式 | 效果 |
|---------|---------|------|
| **异步执行** | `bCanRunAsync = true` | 避免主线程阻塞 |
| **测试排序** | `bAutoSortTests = true` | 减少 50%+ 计算量 |
| **上下文缓存** | 自动缓存机制 | 避免重复计算 |
| **ItemType预计算** | 自定义 ItemType | Generator 中预计算，Test 直接使用 |
| **批量寻路** | `PathfindingBatch` | 10x+ 寻路性能提升 |

### 设计模式应用

| 设计模式 | 应用场景 | 优势 |
|---------|---------|------|
| **策略模式** | Generator/Test 可互换 | 运行时组合 |
| **模板方法** | Test 基类定义流程 | 标准化扩展 |
| **数据提供者** | 参数动态绑定 | 黑板集成 |
| **迭代器模式** | ItemIterator 遍历 | 隐藏存储细节 |
| **类型擦除** | ItemType 系统 | 统一内存管理 |

---

## 七、最佳实践建议

### ✅ 推荐做法

1. **优先使用蓝图扩展 Generator/Context**
   - 快速原型开发
   - 设计师友好
   - 易于迭代

2. **C++ 实现性能关键的 Test**
   - 射线检测、寻路等昂贵操作
   - 复杂数学计算
   - 需要异步执行的逻辑

3. **合理设置 Test Cost**
   - `Low`: 简单计算（距离、点积）
   - `Medium`: 单次射线、碰撞检测
   - `High`: 寻路、批量射线

4. **使用 FAIDataProvider 参数化配置**
   - 支持黑板动态值
   - 易于调试和调优
   - 提升复用性

5. **利用 Context 缓存机制**
   - 相同 Context 只计算一次
   - 避免在 Test 中重复查询

### ❌ 避免做法

1. **不要在 Generator 中执行昂贵操作**
   - Generator 应该快速生成候选
   - 复杂评估留给 Test

2. **不要在 Test 中修改候选列表**
   - 只能评分和过滤
   - 不能添加新候选

3. **不要忘记设置 ValidItemType**
   - Test 必须声明兼容的 ItemType
   - 避免类型不匹配

4. **不要滥用自定义 ItemType**
   - 优先使用内置类型（Point/Actor）
   - 只在必要时创建自定义类型

### 🔍 调试技巧

1. **使用 EQS Testing Pawn**
   - 实时可视化查询结果
   - 逐步回放测试过程
   - 查看每个 Test 的评分

2. **重载 GetDescriptionTitle/Details**
   - 在编辑器中显示友好名称
   - 帮助设计师理解配置

3. **使用 Draw Debug 辅助**
   ```cpp
   // 在 Test 中绘制调试信息
   DrawDebugLine(QueryInstance.World, Start, End, FColor::Red, false, 2.0f);
   DrawDebugSphere(QueryInstance.World, Location, 50.0f, 12, FColor::Green, false, 2.0f);
   ```

4. **添加统计日志**
   ```cpp
   UE_LOG(LogEQS, Log, TEXT("MyTest: Evaluated %d items, %d passed"), TotalItems, PassedItems);
   ```

---

## 八、扩展点能力矩阵

### 终极对比表

| 特性 | Generator | Test | Context | ItemType |
|-----|----------|------|---------|----------|
| **扩展难度** | ⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐⭐ |
| **蓝图支持** | ✅ | ❌ | ✅ | ❌ |
| **性能影响** | 中 | 高 | 低 | 低 |
| **使用频率** | 高 | 极高 | 中 | 低 |
| **定制价值** | 极高 | 极高 | 高 | 中 |
| **黑板集成** | ✅ | ✅ | ✅ | ✅ |
| **异步支持** | ✅ | ✅ | ❌ | ❌ |
| **可复用性** | 高 | 高 | 中 | 低 |

### 学习路径建议

```
入门 (1-2周)
    ├── 使用蓝图 Context
    ├── 使用蓝图 Generator
    └── 理解内置 Test

进阶 (2-4周)
    ├── C++ 实现简单 Generator
    ├── C++ 实现简单 Test
    └── 理解 FAIDataProvider 系统

高级 (1-2月)
    ├── 实现复杂的 Test (射线、寻路)
    ├── 优化性能 (异步、缓存)
    └── 理解 ItemType 系统

专家 (3-6月)
    ├── 自定义 ItemType
    ├── 设计游戏专用的 EQS 组件库
    └── 深度集成游戏系统
```

---

## 九、总结

### 核心价值

EQS 的深度定制能力让你可以：

1. **扩展空间推理能力** - 从简单网格到复杂战术分析
2. **适配任何游戏类型** - 潜行、射击、生存、竞速等
3. **优化 AI 表现** - 从"能用"到"智能"的质的飞跃
4. **提升开发效率** - 模块化、可复用、易于迭代

### 设计哲学

```
内置组件：覆盖 80% 常见需求
扩展接口：解决 20% 特殊需求
蓝图支持：快速原型和设计师友好
C++ 扩展：性能关键和复杂逻辑
```

### 最终建议

1. **从简单开始** - 先用内置组件，确认需要再扩展
2. **蓝图优先** - Generator/Context 优先蓝图实现
3. **性能为王** - Test 和 ItemType 用 C++ 实现
4. **模块化设计** - 创建项目级的 EQS 组件库
5. **持续优化** - 使用 Profiler 分析，数据驱动优化

掌握 EQS 深度定制，你就掌握了 UE AI 系统的核心竞争力！
