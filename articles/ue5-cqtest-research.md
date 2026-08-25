---
title: "UE5: Unreal Engineの CQTestについてまとめた"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: true
---

## はじめに
本稿は`CQTest`プラグインを褒めちぎる記事です。`CQTest`は既存のUEテストフレームワークの弱点を克服しており、実用可能なレベルですごく良いものだ、という話です。


UE5 で C++ のテストを書くときは、Simpleテストを使うことが多いと思います。`IMPLEMENT_SIMPLE_AUTOMATION_TEST` これね。しかしながら、シンプルテストは、fixture がないし、フレームをまたぐテストを書こうとすると大変です。特にダメなのが、ActorやComponent のテストがほとんどできないという点です。Componentなどの標準ライフサイクルをテスト環境で再現できず、めちゃ苦労します。
実質的にpure-C++クラスやUEに依存しない論理部しかテストできず、エンジンが提供するテストフレームワークなのに全然エンジン統合がなっていませんでした。

UEには SIMPLE/COMPLEX/Spec/FunctionalTestなどなど様々なテスト系機能が搭載されているのですが、イマイチ芯を外したものばかりでございました。
UEはC++なので、`CATCH2`や`GOOGLE TEST`など巷で有名なC++テストフレームワークを使えます。実際 `CATCH2`は起動が速くて好き。とはいえ、Unreal Automationフレームワークに乗っかることが重要なので、Simpleテストが総合的に一番マシ、時点でSpecという状況でした。

そこで**CQTest** (Code Quality Tests)の登場でございます。
`CQTest`は xUnit スタイルの fixtureをサポートし、メソッドチェーンでフレーム跨ぎの処理を書けて、さらに PIE を複数立てるネットワークテストまで付いてきます。これぞ顧客が求めていたものです。最高かよ。

UE5.3 から導入され、UE5.5 でエンジンモジュールに格上げされました。本体は `Engine/Source/Developer/CQTest` にあります。プラグイン側 (`Engine/Plugins/Tests/CQTest`) に残っているのはフレームワーク自身のテスト(`CQTestTests`)だけなので不要です。罠です。

我々はテストを置くモジュールの Build.cs に `"CQTest"` を足すだけでいいです。セットアップできました。

```cs: MyGame.Build.cs
// AutomationTestFramework使うときのフラグ
if (Target.WithAutomationTests)
{
    PrivateDependencyModuleNames.Add("CQTest");

    // Editorは後述のMapTestを使うならあった方がいい
    if (Target.bBuildEditor)
    {
        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "UnrealEd", // GUnrealEd->EndPlayMap() 等を使いたいため
        });
    }
}
```
## 使い方サンプル

テストコードは.cpp単体でOKです。
```cpp: MyFeatureTest.cpp
#include "CoreMinimal.h"

// 全てのテストコードはこれで包んだほうがいい
// 意図せぬコンパイルエラー/リンクエラーを防ぐため
#if WITH_AUTOMATION_TESTS

#include "CQTest.h"

TEST_CLASS(FMyFeatureTest, "MyProject.MyFeature")
{
    // BEFORE_ALL / AFTER_ALL から使う値は static にする
    static int32 SuiteSetupCount = 0;

    int32 Value=0;
    int32 PollCount=0;

    // UCLASSじゃないので、UPROPERTYは使えないし使わない
    // UPROPERTY()TObectPtr<UObject> Obj;

    // このテストクラス全体で最初に1回だけ実行
    BEFORE_ALL()
    {
        SuiteSetupCount++;
    }

    // このテストクラス全体の最後に1回だけ実行
    AFTER_ALL()
    {
        SuiteSetupCount = 0;
    }

    // 各 TEST_METHOD の直前に実行
    BEFORE_EACH()
    {
        Value = 0;
        PollCount = 0;
    }

    // 各 TEST_METHOD と、その遅延処理が終わった後に実行
    AFTER_EACH()
    {
        Value = 0;
    }

    // TestCase0:
    TEST_METHOD(BasicAssertions)
    {
        const int32 ExistingValue = 123;
        const int32* NullValue = nullptr;

        ASSERT_THAT(IsTrue(SuiteSetupCount == 1));// BEFORE_ALLでセット済みのはず
        ASSERT_THAT(IsFalse(false));
        ASSERT_THAT(AreEqual(3, 1 + 2));
        ASSERT_THAT(AreNotEqual(3, 4));
        // float 比較は Nearを使う
        ASSERT_THAT(IsNear(1.0f, 1.0001f, /*Epsilon*/0.001f));
        ASSERT_THAT(IsNull(NullValue));
        ASSERT_THAT(IsNotNull(&ExistingValue));

        //TArray::operator==を使う
        ASSERT_THAT(AreEqual(TArray<int>{0,1,2}, TArray<int>{0,1,2})); //全要素比較
        ASSERT_THAT(AreNotEqual(TArray<int>{0,1,2}, TArray<int>{2,1,0})); //全要素比較

        // TMap の検査は OrderIndependentCompareEqualを使う
        const TMap<int32, FString> ExpectedMap =
        {
            { 1, TEXT("One") },
            { 2, TEXT("Two") },
        };
        TMap<int32, FString> ActualMap;
        ActualMap.Add(2, TEXT("Two"));
        ActualMap.Add(1, TEXT("One"));

        ASSERT_THAT(IsTrue(ExpectedMap.OrderIndependentCompareEqual(ActualMap),
            TEXT("TMapの全Keyと全Valueが一致すること")));
    }

    // TestCase1:
    TEST_METHOD(ExpectedError)
    {
        ASSERT_THAT(IsTrue(SuiteSetupCount == 1));// BEFORE_ALLでセット済みのはず

        // このエラーが1回出ること自体を期待結果とする
        Assert.ExpectError(TEXT("Value must be positive"), /*期待するエラー回数*/1);

        // サンプルでわざと自分でログエラー出す
        // 上記ExpectErrorとマッチすると、テストは成功扱いになる
        UE_LOG(LogTemp, Error, TEXT("Value must be positive"));
    }

    // TestCase2:
    TEST_METHOD(WaitsUntilConditionIsSatisfied)
    {
        ASSERT_THAT(IsTrue(SuiteSetupCount == 1));// BEFORE_ALLでセット済みのはず

        // fixtureインスタンスの寿命は保証されているので
        // thisキャプチャしてよい
        TestCommandBuilder
            .Do(TEXT("値を更新する"), [this]()
            {
                Value = 42;
            })
            .Until(TEXT("準備完了を待つ"),
                [this]()
                {
                    // 実際にはロード完了やActorの状態などを判定する
                    return ++PollCount >= 3;
                },
                FTimespan::FromSeconds(1))
            .Then(TEXT("待機後の状態を検証する"), [this]()
            {
                // 実際はここでアクターの状態などを見る
                ASSERT_THAT(AreEqual(42, Value));
            });
    }
}
#endif //WITH_AUTOMATION_TESTS
```

## CQTestのAutomation Framework における立ち位置

CQTest は "Code Quality Test" という名の通り、内部品質(コード品質)を向上させるテストフレームワークです。UEのAutomation Frameworkの上に乗っており既存のテストランナー上で動きます。シンプルテストと全く変わらず、Unreal Frontendや UnrealEditor-Cmdから実行できます。

- Session Frontend の Automation タブから一覧＆実行できる
- `-ExecCmds="Automation RunTests <テスト名>; Quit"` で実行可能
- 既存の SIMPLE / COMPLEX / Spec のテストと同居できる

CQTest以外含めて、UE のテスト周りのモジュール一覧を整理しておきます。

| テスト | 特徴 | 使いどころ |
|---|---|---|
| SIMPLE | 関数 1 個 = テスト 1 個 | 関数スコープで完結する単体テスト |
| COMPLEX | `GetTests()` で実行時にケースを列挙 | パラメタライズテスト |
| Automation Spec (`DEFINE_SPEC`) | BDD スタイル (`Describe`/`It`) | `Test as a Spec`スタイルが好みなら |
| Functional Test (`AFunctionalTest`) | レベル配置の Actor + BP | レベル依存のBP統合テスト |
| Low-Level Tests (Catch2) | 別ビルドターゲット | エンジンを起動しない高速テスト |
| Gauntlet(テストではない) | テストプロセスの制御 | テストをどのビルドでどのマシンでやるかとか |
| **CQTest** | xUnit スタイルの C++ fixture | 単体テスト・結合テスト・すべてこれでいい |

たくさんありますが、CQTest 一択でいいです。特にActorやComponentやPhysicsが絡むならコレ一択です。

次点で`Spec`です。BDD好きな人は `Spec`を使ってもいいでしょう。Functional Test は BP上で組むテストがメインとなります。
Gauntletに至ってはテストフレームワークではなくて、自動TestをCIでどうやって回すかみたいな機能なので別件です。Low-Level Test は起動が高速という点で魅力的ではあるのですが、ここに書けるぐらいの純粋論理なシステムならもはやUEじゃなくてもいいんですよね。

いっぱい出てきましたが、CQTestを使えばいいです。すべてのケースで！

:::message
「すべてのケース」という強い言葉を使っていますが、私の研究によると本当に全部コレでいいです。後発機能なだけあります。
:::


## CQTestの優れているところ

ここからは、「僕から見た君の素敵だと思うところ」を述べていきます。口説き文句が続きます。

### fixture がテストごとに作り直されて嬉しい

`TEST_CLASS` で TestFixtureを作成します。この fixture インスタンスはテストメソッドの実行ごとに new され、終わったら破棄されます。前のテストの状態が次のテストに残りません。従来のテストで状態を持とうとすると その状態をきちんと自前で初期化・解放する手間がありました。`TUniquePtr`を使ってもいいのです。

```cpp: サンプル
TEST_CLASS(FMｙFeatureTest, "tag")
{
    TUniquePtr<FMyState> State;

    BEFORE_EACH()
    {
        State = MakeUnique<FMyState>();
        State->Value = 42;
    }
    AFTER_EACH()
    {
        // デストラクタで自動解放されるのでRest呼ぶ必要ないのだけど、
        // 対象性のため一応呼んでおく
        State.Reset();
    }

    TEST_METHOD(Basic)
    {
        ASSERT_THAT(IsTrue(State->Value, 42));
    }
}
```

```
C#のnUnitや UnityTestと同じような書き味で実装できるという感じです。
```

### AssertThat で失敗したらそこで止まって嬉しい

`Assert.That` が使えると言われたら嬉しいでしょ？わかるこの気持ち？

CQTest の `ASSERT_THAT` は失敗した行でテストメソッドから抜けます。展開すると `if (!Assert.IsTrue(...)) { return; }` となりテストメソッドから抜けます。asserter のメソッドは全部 `[[nodiscard]]` なので、マクロを介さず生で呼んで結果を握り潰すとコンパイル警告になります。安心。

失敗メッセージも書かなくていいです。`AreEqual(3, Count)` が失敗すると
`Expected 3 to equal 5.` のように両辺の値入りでログが出ます。`FString ToString() const` を持つ型(FVector, FNameなど) はそのまま表示されます。この`Assert`は`assert`と違いプロセスをクラッシュさせないのでエディタは停止しません。

シンプルテストを一度でも手書きしたことがある人は、ヌルチェックに失敗した次の行でデリファレンスしてクラッシュし、肝心の失敗ログがクラッシュログになる、というのをやったはず。もう全部`ASSERT_THAT`でよくね？って気持ちになっています。使い分ける認知負荷を下げたい。

```cpp: サンプル
TEST_METHOD()
{
    int* Ptr = GetFooValuePtr();
    ASSERT_THAT(IsNotNull(Ptr));

    // ここには上記アサートが成功したときだけしか来ないので非null保証
    *Ptr = 42;
}
```

全検査を実施してから最後にまとめて報告したい場合はマクロを使わなくていいです。`[[nodiscard]]`ついているのですこし面倒。
```cpp
TEST_METHOD(ObjectInitailize)
{
    FMyObject Obj;
    Obj.Initialize();

    std::ignore = Assert.AreEqual(Obj.Value, 3);
    std::ignore = Assert.AreEqual(Obj.Count, 2);
    std::ignore = Assert.IsTrue(Obj.IsValid(), TEXT("この検査も続けて実行される"));

    // 必ずここまでくる
    // TestRunnerの中に記録が保持されているので、このテストメソッドを抜けたらエラー報告される
}
```


### latent command がチェーンで書けて嬉しい

CQTest では fixture が最初から `TestCommandBuilder` を持っていて、こう書けます。とってもフルーエント！

```cpp
TestCommandBuilder
    .Do(TEXT("リクエスト発行"), [this]() { Loader->RequestLoad(AssetPath); })
    .Until(TEXT("完了待ち"), [this]() { return Loader->IsComplete(); })
    .Then(TEXT("検証"), [this]() { ASSERT_THAT(IsTrue(Loader->GetAsset() != nullptr)); });
```

このチェーンには便利な挙動が仕込まれています。

- `Until` は毎フレーム述語を評価し、**タイムアウトすると `AddError` で失敗になる**(既定 10 秒、後述の CVar で変更可)。条件が満たされないままテストがハングして CI を巻き込む、という事故が起きない
- どこかのステップでエラーが出ると、**残りのステップは自動でスキップ**される
- `OnTearDown` に積んだ後始末は **テストが失敗していても LIFO 順で実行**される

従来はフレームをまたぐたびに `DEFINE_LATENT_AUTOMATION_COMMAND_ONE_PARAMETER` でクラスを 1 個定義して `ADD_LATENT_AUTOMATION_COMMAND` で積んでいました。

### 後始末が保証されて嬉しい

デストラクタ呼ばれます。Actor/Component もちゃんと UnRegister/ Destroyされます。
UObjectはGC回収されます。RAIIが使えます。

各テストの前に `GEngine->DelayGarbageCollection()`、終了後に fixture 破棄と `ForceGarbageCollection()` が自動で走ります。後述の FActorTestSpawnerは、生成した Actor / World をデストラクタで破棄します。
テスト実行のたびにメモリがリークする心配は大きく減りました。

### World/Spawnerが用意されて嬉しい

素の Automation Framework には「Test用のUWorld をどう用意するか」「マップをどうロードするか」が用意されておりませんでした？（なんでなかったん？）
どこのご家庭も独自のヘルパーを実装してTestWorldを構築してきたことでしょう。CQTest でようやく搭載されました。

| コンポーネント | 用途 |
|---|---|
| `FActorTestSpawner` | PIE なしで最小の `UWorld` を作り Actor / UObject をスポーン |
| `FMapTestSpawner` | 指定マップをロード (エディタでは PIE) してテスト |
| `FPIENetworkComponent` | dedicated server + 任意数の client を PIE で立てて replication をテスト |
| `FCQTestSlateComponent` | Slate の tick 経過を待つ |
| `CQTestAssetHelper` | AssetRegistry 検索、Blueprint クラスの取得 |

特にネットワークテストは、FPIENetworkComponentを使った方がいいでしょう。`NetTestHelpers.h`自体はCQTest以外でも使えるのでしょうがね。

## 基本的な使い方

ここからは正直AIに聞けよ感が否めませんが、温かみのあるハンドリトゥンで記載しておきます。

### 最小形

1 ケースだけなら `TEST` マクロで済みます。第 2 引数が Session Frontend 上のツリー位置になります。

```cpp
#include "CQTest.h"

TEST(LerpHalf_ReturnsMidpoint, "Game.Math")
{
    ASSERT_THAT(IsNear(50.f, FMath::Lerp(0.f, 100.f, 0.5f), UE_KINDA_SMALL_NUMBER));
}
```

関数スコープで完結するテストならこれでいいと思います。SIMPLEテスト使ってもいいんですが、使い分ける認知負荷を下げたい。

### fixture

本命。`BEFORE_EACH` / `AFTER_EACH` は各メソッドの前後で呼ばれ、メンバ変数はメソッドごとに独立します。

例として"インベントリシステム"を題材にします。インベントリの実装はどうでもいいので割愛。解説のためにコメントを多数書いています。実際のプロダクトコードではこんな細かいコメントは書きません。あしからず。

```cpp
#include "CQTest.h"
#include "Inventory/Inventory.h"

// インベントリシステムをテストするFixtureクラス
TEST_CLASS(FInventoryTest, "Game.Inventory")
{
    FInventory Inventory{};   // メンバ変数として保持

    // 初期状態としてリンゴが3つあることを前提にする
    BEFORE_EACH()
    {
        Inventory.Add(TEXT("Apple"), 3);
    }

    // 同じアイテムを追加したら3+2で5個になるはずだ
    TEST_METHOD(Add_ExistingItem_StacksCount)
    {
        Inventory.Add(TEXT("Apple"), 2);
        ASSERT_THAT(AreEqual(5, Inventory.Count(TEXT("Apple"))));
    }

    // 3個の状態から4個をRemoveしたとき failしてnop扱いになり3個のままのはずだ、という仕様
    TEST_METHOD(Remove_MoreThanHeld_FailsAndKeepsCount)
    {
        // 在庫以上のRemove の返り値は falseのはず
        ASSERT_THAT(IsFalse(Inventory.Remove(TEXT("Apple"), 4)));
        // 在庫は3個のままのはず
        ASSERT_THAT(AreEqual(3, Inventory.Count(TEXT("Apple"))));
    }
};
```
`FInventryTest::Inventory` はメンバ変数として定義されます。`TEST_METHOD`単位でインスタンスはdeleteされますから、`FInventory`のデストラクタが呼ばれます。セットアップはコンストラクタで与えるか、`BEFORE_EACH`で行います。`TEST_METHOD`それぞれで初期状態を確定できます。


Session Frontend には `Game > Inventory > FInventoryTest` の下に 2 メソッドがぶら下がります。コマンドラインからは

```sh: cmd
UnrealEditor-Cmd.exe MyProject.uproject -ExecCmds="Automation RunTests Game.Inventory; Quit" -unattended -nullrhi -log
```

↑はcoding AI が自動実行できます。テスト実行はskill化しておくといいでしょう。

### assert 一覧

`ASSERT_THAT(...)` の中に書けるのは `FNoDiscardAsserter` のメソッドです。

| メソッド | 内容 |
|---|---|
| `IsTrue(Cond)` / `IsFalse(Cond)` | 真偽 |
| `IsNull(Ptr)` / `IsNotNull(Ptr)` | ヌル判定 |
| `AreEqual(Expected, Actual)` / `AreNotEqual` | `==` / `!=` 比較。失敗時は両辺の値を表示 |
| `AreEqualIgnoreCase` / `AreNotEqualIgnoreCase` | FString の大文字小文字無視比較 |
| `IsNear(Expected, Actual, Epsilon)` | 浮動小数の近似比較 |

全オーバーロードに失敗メッセージを追加で渡せます。無条件に失敗させたいときは `ASSERT_FAIL(TEXT("..."))` です。

### 異常系をテストするとき
エラーログが出ること自体を検証したい、あるいは想定内のエラーでテストを落としたくないときは `Assert.ExpectError` / `ExpectErrorRegex` を先に呼んでおきます。

```cpp
// エラーログがでることの検証
TEST_METHOD(Equip_UnknownItemId_LogsError)
{
    // 存在しないアイテムIDをEquip()に渡したとき、エラーログが出る仕様
    Assert.ExpectError(TEXT("Unknown item id"));   // 部分一致、出現回数も指定可
    Equipment.Equip(TEXT("no_such_item"));
}
```
### UObjectを使ったテスト

```cpp
TEST_CLASS(FUObjectMemberTest, "MyProject.UObjectMember")
{
    // フレームを跨がないなら弱参照でOK
    TWeakObjectPtr<UObject> TestObject = nullptr;

    BEFORE_EACH()
    {
        TestObject = NewObject<UObject>(GetTransientPackage(), TEXT("TestObject"));
    }

    AFTER_EACH()
    {
        TestObject = nullptr;
    }

    TEST_METHOD(UObjectを検査する)
    {
        // 同一フレーム内で完結し、明示的なGCも実行しない
        ASSERT_THAT(IsNotNull(TestObject));
        ASSERT_THAT(AreEqual(
            FName(TEXT("CQTestObject")),
            TestObject->GetFName()));
    }
};
```

CQTest側で `DelayGarbageCollection`を呼び出してくれますので、すぐ完了するテストなら弱参照を使っていいです。ただし、1フレーム遅延させるだけです。数フレームにわたって延期するわけではないので、その場合は`TStrongObjectPtr`を使うか、`AddToRoot`する必要があります。

なお`NewObject`は使わずに基本的に`FSpawnHelper`や`FTestActorSpawner`を使うべきです。

:::message
`FTestActorSpawner::SpawnActor`したアクターはTestWorldから参照されるのでGCされません。弱参照でkeepしていいです。
`FTestActorSpawner::SpawnObject`はGCされえます。Actorに紐づけるか、TStrongObjectPtrを使いましょう。
:::

### Actor / Component を使ったテスト

`FActorTestSpawner` はテスト用の`UWorld`を1個作ります。このWorldは物理シーンまで含めて初期化済みで、テストが終われば生成したActorごと破棄します。テスト用 World の用意は私も過去に挑戦したことがあるのですが、かなりUnrealEngineに精通していないとできないめちゃムズ案件です。Spawner はそれを fixture のメンバ 1 個にカプセル化してくれます。マップロードが走らないので速いです。

```cpp
#include "CQTest.h"
#include "Components/ActorTestSpawner.h"

// 自動タレットキャラのテスト
// タレットは敵キャラをタゲにとる
TEST_CLASS(FTurretTest, "Game.Turret")
{
    FActorTestSpawner Spawner;

    // 捕捉範囲内にいる敵をタゲにとる
    TEST_METHOD(AcquireTarget_EnemyInRange_TargetsIt)
    {
        // タレットのレンジが10mだとして
        // タレットを原点、敵を5mの位置にスポンさせる
        ATurret& Turret = Spawner.SpawnActor<ATurret>();
        AEnemy&  Enemy  = Spawner.SpawnActorAt<AEnemy>(FVector(500, 0, 0), FRotator::ZeroRotator);

        Turret.UpdateTargeting();

        // UpdateTargetingで捕捉しているはず
        ASSERT_THAT(IsTrue(Turret.GetTarget() == &Enemy));
    }
};
```

`SpawnActor` は失敗時に check で落ちるので戻り値は参照です。生成した Actor / World は fixture 破棄時に逆順で破棄されます。AActor 以外は `SpawnObject<UMyObject>()`、GameInstanceSubsystem が要るなら `Spawner.InitializeGameSubsystems()` を呼んでおくと `UTestGameInstance` 経由で初期化されます。

Component はランタイムと同じ手順で足せます。TestWorld が生きているので `RegisterComponent()` が動きます。

```cpp
TEST_METHOD(RegisterComponent_OnSpawnedActor_Works)
{
    AActor& Owner = Spawner.SpawnActor<AActor>();
    auto* Health = NewObject<UHealthComponent>(&Owner);
    Health->RegisterComponent();   // TestWorld に登録される

    ASSERT_THAT(IsTrue(Health->GetOwner() == &Owner));
}
```

Registerは手動なのかよ、と思うかもしれませんがRegister前後で挙動を変更したいためこれはありがたい仕様です。ComponentやActorをPoolingするときはUnRegisterしたりしますよね？Registerタイミングをいじりたいことがあるのでかえって好都合です。


物理シーンが最初からあるので、コリジョンを持つ Actor をスポーンすれば物理クエリができます。コリジョン絡みのロジック (射線判定、設置判定、到達可能性) がこの粒度でテストできてラクチンですね。

```cpp
TEST_METHOD(LineTrace_HitsSpawnedBlocker)
{
    ABlockerActor& Blocker = Spawner.SpawnActorAt<ABlockerActor>(
        FVector(500, 0, 0), FRotator::ZeroRotator);

    FHitResult Hit;
    const bool bHit = Spawner.GetWorld().LineTraceSingleByChannel(
        Hit, FVector::ZeroVector, FVector(1000, 0, 0), ECC_Visibility);

    ASSERT_THAT(IsTrue(bHit));
    ASSERT_THAT(IsTrue(Hit.GetActor() == &Blocker));
}
```

:::message
物理シムは動きません。後述。
:::
:::message
GameMode 起点のゲームフローや tick 進行までは対応しません。スポーン済みオブジェクトへの同期的な操作と物理クエリまでがこの Spawner の担当範囲で、BeginPlay やシミュレーション進行に依存するテストは後述の`FMapTestSpawner` を使います。
:::

### 物理ワールドでのテスト

上記の通り、物理クエリはできるのですが、素のままでは Chaosの物理シムは動きません。そこで動かせるように拡張します。SpawnerのもつWorldの`UWorld::bShouldSimulatePhysics=true` が必要になります。`UWorld::Tick(..., DeltaTime)`を自前で行い、ワールド時間を動かすことで物理シムも動きます。

```cpp: 疑似コード
// helper関数
static void StepFrame(UWorld& World, int NumFrames, float DeltaTime = 1.0f/60.0f)
{
    for(NumFrames)
    {
        World.Tick(LEVELTICK_ALL, DeltaTime);
        // 罠. GFrameCounterを進めないと chaosソルバが動いてくれない!
        // 泣く泣くグローバル変数をいじる!
        ++GFrameCounter;
    }
}

BEFORE_EACH()
{
    Spawner.GetWorld().bShouldSimulatePhysics=true;
    Spawner.GetWorld().BeginPlay();
}

AFTER_EACH()
{
    // Worldの畳み方この順番であってますか？？？自信ない。
    auto&& World = Spawner.GetWorld();
    GEngine->ShutdownWorldNetDriver(...);
    World.DestroyWorld(...);
    World.ReleasePhysicsScene(...);
    GEngine->DestoryWorldContext(...);
}

TEST_METHOD(...)
{
    //スポンして、初速与える

    StepFrame(Spawner.GetWorld(), 10); // 例えば 10F進める

    // 座標が移動していることを確認する
}
```

このように初期条件を整えることができれば、テスト世界でも安定した物理解を得ることができます。当たり判定系のバグ調査やなぞの物理挙動の解析に役立ちますね！というか、ACharacterの移動処理は必須なのでほとんどのテストで必要になると思います。

イケてないところ2点ありました。`GFrameCounter`をいじる悪いコードがありますね。エンジン改造以外のいい方法が思いつかなったので、やっつけでこうしてます。エディタでしか動かさないテストだしいっかなって。おそらく実機でこのテストを動かすとやばいことになりそうな危険の予感がします。また、物理を動かすためのWorldの初期化・破棄方法がよくわかりません。誰か何とかして。

### フラグ

`TEST_CLASS` の既定は「エディタでもスタンドアロンでも実行可、フィルタは Product」です。Editor/Client/Server/Commandlet/Programで動きます。

明示的に変えたいときは `TEST_CLASS_WITH_FLAGS` を使います。

```cpp
TEST_CLASS_WITH_FLAGS(FEditorOnlyTest, "Game.Editor",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)
{
    // ...
};
```

フィルタ (Smoke / Engine / Product / Perf / Stress / Negative) をちょうど 1 個指定しないと static_assert でコンパイルが止まります。
テストディレクトリを手で管理したくなければ、第 2 引数に `GenerateTestDirectory` を渡すとソースファイルのパスから自動生成されます。`"Game.[GenerateTestDirectory]"` のように部分埋め込みもできます。

## 応用的な使い方

### TestCommandBuilderを使ってフレーム跨ぎにする

ステップ間で共有する状態は fixture のメンバ変数にして、ラムダ式で`this`キャプチャするのが基本形です。ステップ実行時にはメソッドの関数スタックはもう存在しないので、ローカル変数のキャプチャはいけません。

`BEFORE_EACH` でも latent を積めます (マップロード待ちはこれ)。latent を一切積まなければテスト全体が 1 フレームで同期完結します。

```cpp
TEST_CLASS(FQuestTest, "Game.Quest")
{
    FActorTestSpawner Spawner;
    UQuestSubsystem* Quest = nullptr;   // ステップ間の共有状態はメンバへ

    BEFORE_EACH()
    {
        Spawner.InitializeGameSubsystems();
        Quest = Spawner.GetGameInstance()->GetSubsystem<UQuestSubsystem>();
    }

    TEST_METHOD(AcceptQuest_AsyncTableLoad_Completes)
    {
        TestCommandBuilder
            .Do(TEXT("受注"), [this]() { Quest->Accept(TEXT("Q001")); })
            .Until(TEXT("ロード完了待ち"), [this]() { return Quest->IsReady(TEXT("Q001")); })
            .Then(TEXT("検証"), [this]() {
                ASSERT_THAT(AreEqual(EQuestState::Active, Quest->GetState(TEXT("Q001"))));
            });
    }
};
```

`WaitDelay` (指定時間待つ) もありますが、ヘッダのコメントに`Until`を使えと書いてあります。最後の手段です。

### マップを開いてテストする (FMapTestSpawner)

ゲームフローごと動かす結合テストです。`BEFORE_EACH` でロード待ちコマンドを積むのが決まった形です。
CQTest を使いたい理由の2番目ぐらいの機能です。

```cpp
#include "CQTest.h"
#include "Components/MapTestSpawner.h"

#if WITH_AUTOMATION_TESTS

// 大前提として、Testマップには Bossキャラが直接配置してある
// マップロード時点で自動的にBossキャラはスポンする想定
TEST_CLASS_WITH_FLAGS(FBossArenaTest, "Game.Boss",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)
{
    TUniquePtr<FMapTestSpawner> Spawner;
    TWeakObjectPtr<APawn> PlayerPawn = nullptr;

    BEFORE_EACH()
    {
        Spawner = MakeUnique<FMapTestSpawner>(TEXT("/Game/Maps/Test"), TEXT("BossArena"));
        Spawner->AddWaitUntilLoadedCommand(TestRunner);   // PIE 起動+ロード完了までの latent を積む
    }

    TEST_METHOD(BossDies_DoorOpens)
    {
        TestCommandBuilder
            .StartWhen([this]() {
                // MapのWorldSettigns>DefaultPawnが勝手にスポンするはずだよね
                PlayerPawn = Spawner->FindFirstPlayerPawn();
                return PlayerPawn != nullptr;
            })
            .Do(TEXT("ボスを倒す"), [this]() {
                // コード上でボススポンは実装されていないが、マップ配置ずみであるためすでに存在するはず
                ABoss* Boss = Cast<ABoss>(UGameplayStatics::GetActorOfClass(&Spawner->GetWorld(), ABoss::StaticClass()));
                Boss->Kill();
            })
            .Until(TEXT("扉が開くのを待つ"), [this]() {
                AArenaDoor* Door = Cast<AArenaDoor>(UGameplayStatics::GetActorOfClass(&Spawner->GetWorld(), AArenaDoor::StaticClass()));
                return Door && Door->IsOpen();
            });
    }
};

#endif // WITH_AUTOMATION_TESTS
```
マップを使ったテストはかなりフレーキーになります。なぜならマップ内に何を配置しているか？に大きく依存するからです。ソースコードとテストマップを1:1対応させるのがいいと思います。

製品用マップをテストするには向いていません。なぜならば、すぐ仕様変更されるからです。Bossとしては1つだけど、双子のボスでBossキャラが2体いたら上のテストはfailするよね、とか。
仕様追加や変更のたびにソースコードも一緒に修正しないといけないとなると面倒くさいです。
そちらは自動テスト側の検討事項となるでしょう。

Coding AIにUE MCP使って操作させるなら問題にならないかもしれません。


テスト専用の空マップでよければ `FMapTestSpawner::CreateFromTempLevel(TestCommandBuilder)` が一時レベルを作ってくれます (エディタ限定)。

PIE テストを CI で連続実行するときの注意。

**PIE は自前で畳む。** `FMapTestSpawner` を破棄しても PIE セッション自体は終わりません(!?)。放置すると world とレンダーリソースが後続のテストへ持ち越されてしまうので、`AFTER_EACH` で明示的に終了させます。
あまりに罠すぎるので気を付けてください。
```cpp
AFTER_EACH()
{
    // Spawner の破棄だけでは PIE が継続し、リソースが次のテストへ残る
    if (Spawner && GUnrealEd && IsValid(GUnrealEd->PlayWorld))
    {
        GUnrealEd->EndPlayMap();
    }
    Spawner.Reset();
}
```

### GPUを使うテスト
レンダリング依存のテストは実行環境を選べるようにします。

GPU が要るテストはフラグに `EAutomationTestFlags::NonNullRHI` を足したうえで、テスト冒頭で `FApp::CanEverRender()` を見て、 `-nullrhi` の インスタンスでは早期 return しておくとよいです。 ヘッドレス CI と GPU 付き CI で同じテストバイナリを共有できます。これがないと、ヘッドレストでテストスイートを全件実行したときにRedになってしまいます。

ComputeShader とか、Nanite周りとか、RHIが必須のテストは面倒くさいってことです。


### ネットワークテスト (FPIENetworkComponent)

CQTest の価値のコアにして大トロ、またの名を白眉。ネットワーク疑似貫通テストです。dedicated server + 任意数の client を PIE で立ち上げ、server / client それぞれの視点でステップを書けます。状態は `FBasePIENetworkComponentState` を派生させて持ち回ります。

```cpp
#include "CQTest.h"
#include "Components/PIENetworkComponent.h"

#if ENABLE_PIE_NETWORK_TEST

NETWORK_TEST_CLASS(FHealthReplicationTest, "Game.Network")
{
    struct FState : public FBasePIENetworkComponentState
    {
        AMyCharacter* Character = nullptr; // UPROPERTY(Replicated)じゃない!
    };

    FPIENetworkComponent<FState> Network{ TestRunner, TestCommandBuilder, bInitializing };

    BEFORE_EACH()
    {
        FNetworkComponentBuilder<FState>()
            .WithClients(2)
            .WithGameMode(AMyGameMode::StaticClass())
            .Build(Network);
    }

    TEST_METHOD(Damage_OnServer_ReplicatesHealthToClients)
    {
        Network.SpawnAndReplicate<AMyCharacter, &FState::Character>() // 格納先のメンバフィールドをアドレス指定する
            .ThenServer(TEXT("サーバーでダメージ適用"), [](FState& Server) {
                Server.Character->ApplyDamage(30.f);
            })
            .UntilClients(TEXT("全クライアントへの複製を待つ"), [](FState& Client) {
                return Client.Character && Client.Character->GetHealth() == 70.f;
            });
    }
};

#endif // ENABLE_PIE_NETWORK_TEST
```

こんなコードが書けて、本当に動くんですか！？そんな都合のいいシステムがあるんでしょうか。私はまだ疑っています。

`SpawnAndReplicate` はサーバーで Actor をスポーンし、全クライアントに複製が行き渡って `FState` の指定メンバに格納されるまで面倒を見てくれます。`UntilClients` は全クライアントで述語が真になるまで待ち、タイムアウトすれば失敗です。Replication のリグレッションテストがこの行数で書けるのは、すごいですね。

FStateは USTRUCTではないので、`UPROPERTY(Replicated)`は付与できません。スポンしたActor自体は管理領域で保持されておりGCされません。

生ポなのは気になりますが、テスト中の借用ポインタなので良いです。TObjectPtrにすると却って厄介です。
というのも、`SpawnAndReplicate<T1, T2>` の`T2`では格納先アドレスを指定しているからです。T2はポインタである必要があり、TObjectPtrはポインタとしてふるまうオブジェクトであってポインタじゃないから不適切なのです。

### fixture の共通化 (TEST_CLASS_WITH_BASE)

複数のテストクラスで同じセットアップを使い回すなら、`TTest` 派生の中間 fixture を作ります。

```cpp
template <typename Derived, typename AsserterType>
struct FCombatFixture : public TTest<Derived, AsserterType>
{
    FActorTestSpawner Spawner;
    AArena* Arena = nullptr;

    // BEFORE_EACH の実体は virtual void Setup() override
    BEFORE_EACH()
    {
        Arena = &Spawner.SpawnActor<AArena>();
    }

    AMyCharacter& SpawnCharacterWith(FName WeaponId)
    {
        AMyCharacter& Character = Spawner.SpawnActor<AMyCharacter>();
        Character.Equip(WeaponId);
        return Character;
    }
};

TEST_CLASS_WITH_BASE(FMeleeTest, "Game.Combat", FCombatFixture)
{
    AMyCharacter* Attacker = nullptr;

    // 派生側の BEFORE_EACH は基底の Setup() を上書きしてしまう
    BEFORE_EACH()
    {
        FCombatFixture::Setup();   // 基底のセットアップを明示的に呼ぶ。忘れると Arena が nullptr のまま
        Attacker = &SpawnCharacterWith(TEXT("Sword"));
    }

    TEST_METHOD(SwordHit_DealsBaseDamage)
    {
        ASSERT_THAT(IsNotNull(Arena));   // 基底の Setup() が呼ばれていれば通る
        // ...
    }
};
```

`BEFORE_EACH` の実体は `virtual void Setup() override` なので、基底と派生の両方で書くと普通の仮想関数オーバーライドとして基底側が上書きされます。基底のセットアップも走らせたいときは、上の例のように派生側の先頭で `FCombatFixture::Setup()` を明示的に呼んでください。呼び忘れてもコンパイルは通ってしまいます。

### BEFORE_ALL / AFTER_ALL

罠です。気を付けて使うように。
xUnitの OnetimeSetup/OnetimeTeardownに該当する機能かとおもったらそんなことは全然ありませんでした。

static 関数として定義され、同一テストクラスの実行区間に入るとき/出るときに 1 回ずつ呼ばれます。共有リソースの構築に使えますが、static メンバを使う羽目になります。
:::message
static メンバを使った時点でfixture がテストごとに独立しなくなるため注意して使ってください。TEST_METHODの実行順に依存しないように気を付ける。
:::


### タイムアウトの調整

`Until` 系の既定タイムアウトは CVar で決まります。

| CVar | 既定値 | 対象 |
|---|---|---|
| `TestFramework.CQTest.CommandTimeout` | 10 秒 | 通常の latent command |
| `TestFramework.CQTest.CommandTimeout.Network` | 30 秒 | `FPIENetworkComponent` |
| `TestFramework.CQTest.CommandTimeout.MapTest` | 30 秒 | `FMapTestSpawner` のマップロード |

Project Settings (CQ Test Settings) からも設定できます。個別のコマンドには `Until(Query, FTimespan::FromSeconds(5))` のように直接渡せるほか、テストクラス単位でまとめて変えたいときはこう書きます。

```cpp
TSharedPtr<FScopedTestEnvironment> Env;

BEFORE_EACH()
{
    // fixture 破棄時に Env が解放され、CVar は元の値に戻る
    Env = UCQTestSettings::SetTestClassTimeouts(FTimespan::FromSeconds(30));
}
```

**注意**
:::message
恐らくバグなのですが, `SetTestClassTimeouts`に与える`FTimespan`は 60秒以上を与えられません。内部で使用している`FTimespan::GetSeconds()`は `mod 60` を返すために 例えば120秒を指定すると`120%60=0` となり タイムアウト0秒になります。`FTimespan::GetTotalSeconds` を使うべきなんです。
:::

エンジン修正するか、59秒以下にしてください。(UE5.8.1で確認)

```cpp
    // タイムアウトが0秒になっちゃう!!!
    Env = UCQTestSettings::SetTestClassTimeouts(FTimespan::FromSeconds(120));
```

(そんな長いテストを作るんじゃねぇ、というのはその通り）

### テストタグ

UE5.6 以降のエンジンならテストにタグを付けてフィルタ実行できます。書式は `[Tag]` の連結です。

```cpp
TEST_CLASS_WITH_TAGS(FNightlyInventoryTest, "Game.Inventory", "[Nightly][Inventory]")
{
    TEST_METHOD_WITH_TAGS(FullStressCase, "[Slow]")
    {
        // ...
    }
};
```

コンソールから `Automation SetTagFilter <式>` でグローバルフィルタを設定してから `RunTests` / `RunAll` すると、タグに一致するテストだけが対象になります。テスト名を網羅するのは大変なのでタグが整備されているなら楽です。CIのナイトリーテストが楽になるのかな。

---
## CQTestの罠

### fixture のコンストラクタはエンジン起動時にも走る
テスト登録のために、起動時に各 fixture が 1 回インスタンス化されます。このときは `bInitializing` が true になっているので、コンストラクタで重い処理やアセットアクセスをするなら必ずこれを見て分岐してください。メンバ初期化子で重い処理を書くと、エディタ起動が遅くなる恐れがあります。初期化は素直に `BEFORE_EACH` に書くのが安全です。

```cpp
struct FSuperHeavyValue
{
    FSuperHeavyValue()
    {
        // めっちゃ重たい処理
        Sleep(1000);
    }
}

TEST_CLASS(FHeavyTest, "...")
{
    FHeavyTest()
    {
        if(bInitializing)
        {
            //Test Frameworkの初期化中らしい
            return;
        }

        // 重たい初期化
        Sleep(1000);
    }

    // このFSuperHeavyValueのコンストラクタ内でとんでもない処理していると
    // 結局めっちゃ遅くなるので注意!
    FSuperHeavyValue Value{};
}
```

### TestCommandBuilderのOnTearDown怪しい
TestCommandBuilderのOnTearDownはエラー発生前の登録済みの後始末だけ実行されます。
コマンドビルド前に`UE_Log(ERROR)`などのエラーが発生していたら呼ばれません。

```cpp
if(Cond){ UE_LOG(ERROR,...);}

TestCommandBuilder.OnTearDown([]()
{
    //Error出てると、このteardown実行されない
    //teardownなんだから実行してくれー
});
```
:::message
AFTER_EACHやデストラクタの方が信頼できます。
:::

### FActorTestSpawnerは World::BeginPlay呼ばない
AActor::DispatchBeginPlayを呼ぶ必要ありそう？

### SmokeTestに気を付けろ
:::message
CQTestに限りませんが、Smoke フィルタにするとスモークテストになります。スモークテストはUEエディタ起動時に毎回必ず実行されるという仕様（罠）があります。
:::

スモークテストの定義からして一見良さそうですが、これはエディタの起動時間を延ばしかねません。アーティストや非プログラマの起動する環境においてもスモークテストが起動します。
CIでスモークテストを実行するのは賛成ですが、普段の業務でスモークテストを実施するのは過剰でしょう。
かえって反復時間を減らすことになりかねません。スモークテストは本当に超高速で実行できるもの、このまま起動したら支障が出るフェータルなシステム、などに限定するべきです。
本当にエディタの起動ごとに必要なのか？それはCIによる定期実行でだめなのか？という点をよく考えてください。


逆に実機でのスモークテストは既定で一切走りません。DevelopmentのPackageを作ったうえ、.iniに`bForceSmokeTests=true`または `-bForceSmokeTests`を追加する必要があります。
TestConfigurationならば、さらにbForceCompileDevelopmentAutomationTests を指定する必要があります。
こんなに苦労するぐらいならスモークテストやっぱ使わない方がよくないですか？

### マクロに気を付けろ
C++マクロの悪いところ出てます。
※UE5.8.1で確認。

```cpp
if( bCondition) ASSERT_FAIL(...);

// ここには来ない
DoFooBar();
```

これは展開すると以下のようになります。
```cpp
if( bCondition)
    this->Assert.Fail(...);
return;

//ここには来ない
DoFooBar();
```
古典的なブレースなしif文とマクロのバグなので、ちゃんと{}を使いましょう。

```cpp
if(bCondition){ ASSERT_FAIL(...); }

// OK
DoFooBar();
```

釈迦に説法ですが、補足します。

`{}`なしのif文は次の1つのstatementだけをtrue-branchとして扱います。
よって、`if() GetInstance().GetFoo().DoBar(); return;` は
```cpp
if(){ GetInstance().GetFoo().DoBar(); }
return;
```

として解釈されます。`if`とマクロを組み合わせると意図せぬコードになってしまうというバグです。このバグは多くのプログラマを苦しめたため、if文には`{}`をつけろという[コーディング規約](https://www.jpcert.or.jp/sc-rules/c-exp19-c.html)が普及しました。Epicのコーディング規約は`{}`を付けるようになっているため、この罠は発動しないはずです。

[PRE10-C. 複数の文からなるマクロは do-while ループで包む](https://www.jpcert.or.jp/sc-rules/c-pre10-c.html)


## CQTestのイケてないところ

### 日本語テスト名が文字化けする
cppはマルチバイト文字に対応しているので合法です。コンパイルできます。ただし、UnrealFrontend上で文字化けします。ログもターミナルの環境によっては化けるかもしれません。英語がいんじゃないかな。

テスト名には機能や仕様を書け、と言われますね。日本語が使えるんだけどログから読めなくなる危険があるのであんまり使えません。
```cpp
// 合法だけど文字化けするかも
TEST_CLASS(Fリクエストキューイング機能Test,"Tag")
{
    TEST_METHOD(バッファ溢れはfalseを返す){}
    TEST_METHOD(MPSCでスレッドセーフ){}
    TEST_METHOD(リクエストのキャンセル){}
}
```

### パラメタライズテストがない
`TEST_METHOD` はコンパイル時列挙で、COMPLEX テストの `GetTests()` のような実行時のケース生成に相当する仕組みがありません。データ駆動でケースを増やしたければ 1 メソッド内でループを回すことになり、Session Frontend 上では 1 件として扱われます。テストデータを自前で用意して、どのデータで落ちたかは失敗メッセージに自分で入れることになります。

### latent の中から latent を積めない
未サポート (「Adding latent actions from within latent actions is currently unsupported」)

### ASSERT_THAT は関数スコープを越えない
実体が `return` なので、ヘルパー関数やラムダ式に切り出すとヘルパーから抜けるだけになります。そのラムダの残りは中断されますが同じ意味論ではありません (エラー自体は記録されるので、後続の latent ステップはスキップされます)。GoogleTest の `EXPECT_*` に相当する「失敗を記録して続行」する機能もありません。

### ネットワークテストはエディタ限定
`ENABLE_PIE_NETWORK_TEST` は `WITH_EDITOR` を含むので、`NETWORK_TEST_CLASS` はパッケージ済みビルドでは存在しません。実機・実プロセス間のネットワークテストは Gauntlet の領分です。

### エンジン起動なしの高速実行はできない
テストはエディタまたはゲームプロセスの中で走ります。エンジン起動が遅くて高速ユニットテストは不可能です。コミットフックなども危険でしょう。

Catch2 ベースの Low-Level Tests はエンジン起動しないので速いです。こちらな高速ユニットテストも可能かもしれません。

### C++ 専用
Blueprint からは書けません。BPerは Functional Test 使え。

## 実際に使った所感

凄くいい。ぜんぶこれでいい。Automation Specとかいらんかった。
人間が書く際の書き味もいいです。Coding AIに書かせるのもいいです。

テスト実行時間の面で、Canon-TDDは少し苦しいかもしれません。CQTestが悪いんじゃないんです。UEが悪いんです。

Canon-TDDにおけるテストリストの作成はもちろんできますが、その後のテストサイクルがかなり苦しいです。UEのビルド時間と起動時間が問題でして、1ショット数十秒かかっちゃうので、Red-Green-Refactorという悠長なサイクルは取れません。CQTestじゃなくて、UEそのものの起動問題です。

起動時間に目をつむれば、Coding AI に自律駆動させてTDDをぶん回させる分には都合がいいです。テスト実行は(頑張れば)並列化できるので、複数セッション扱えます。


## まとめ
Uneral Automation は実機テストをサポートしています。つまりShippingからは除外したけど、Developmentや最適化ありのTestコンフィグで実機上の自動テストや機能テスト、単体テストをぶん回せるのです。グラボ固有のバグや、コンソール限定のバグを追いかけるときに動かせます。

CQTestはマップをロードしActorを操作できるため、かなり凝ったテストをエディタや実機で動かせるかと思います。テストのための差し込み口を想定した設計を行えば、単体テスト、統合テスト、自動プレー、パフォーマンス測定、バグ再現、といった色んなテストを実装できるはずです。

実際、私はパフォーマンス改善をコーディングAIに指示して行ったことがあります。
「推測するな、計測せよ」の鉄則のもと、特定シナリオのデータをハードコードで用意させ、PIEさせて、Statsやログを仕込ませ、それをAIが認識して自動で最適化をかけていきます。高速化ループを人間が介在せずに勝手に進めていく、というのは見ていて楽しいものであります。とあるタスクでは90msの処理を10msまで縮めてくれました。

バグ再現とバグ修正も回帰テスト含め、ちゃんとCanon-テスト駆動を実行できたこともあります。Coding AI時代になりテストを実装するための負担がなくなったことで、CQTestの価値は非常に高まったと言えるでしょう。

### 参考

- [Automation System User Guide](https://dev.epicgames.com/documentation/unreal-engine/automation-system-user-guide-in-unreal-engine)
- `Engine/Source/Developer/CQTest/`　ヘッダ先頭にボイラープレート例のコメントがある。`CQTest.h`, `Components/*.h` を読むのが一番早い
- `Engine/Plugins/Tests/CQTest/Source/CQTestTests/` 使い方のサンプル集としても読める
