---
title: "UE5:Unreal EngineのInterfaceでDelegateを上手に扱う方法についての研究"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: false
---

# はじめに
過去に`UInterface`そのものについての研究を行いました。しかしながら、`Delegate`と組み合わせて使うとき、`UInterface`にはいくつかの制限があり思うように行かないことがありました。
[UE5:Unreal EngineのInterfaceについてまとめた](https://zenn.dev/allways/articles/ue5-interface-research)

本稿では、`UInterface`で具象に依存せずに`Delegate`を扱う方法について研究しました。

特にやりたいことは、UI要素WBP向けに HPの変化デリゲートをinterface経由で公開することです。さらに、UI要素の寿命管理のためにキャラクタのスポン・デスポン・死亡・復活などのライフサイクルイベントをinterface経由で公開することです。

普通の方法ではダメでしたが、一工夫入れることで解決できました。
その工夫と奮闘についての研究成果を報告いたします。

# 忙しい人のための結論

作戦Aと作戦Bのどちらかを使うとよい。

## 作戦A: Delegateを所有するUObjectを定義してそれをinterface経由でGetする

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FUpdateHPDelegate, int, NewHP, int, NewMaxHP);

UCLASS()
class UStatusEventsObject : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintAssignable)
    FUpdateHPDelegate OnUpdateHPDelegate;
};

UINTERFACE(BlueprintType, Meta = (CannotImplementInterfaceInBlueprint))
class UStatusProviderInterface : public UInterface
{
    GENERATED_BODY()
};
class IStatusProviderInterface
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable) virtual int GetHP() const = 0;
    UFUNCTION(BlueprintCallable) virtual int GetMaxHP() const = 0;
    UFUNCTION(BlueprintCallable) virtual UStatusEventsObject* GetStatusEvents() const = 0;
};
```

## 作戦B: BindTo/UnBindFrom関数を公開する

```cpp
DECLARE_DYNAMIC_DELEGATE_OneParam(FActorOnDeadDelegate, AActor*, DeadActor);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMultiCastActorOnDeadDelegate, AActor*, DeadActor);

UINTERFACE(BlueprintType, Meta = (CannotImplementInterfaceInBlueprint))
class UCharacterLifeCycleEventSource : public UInterface
{
    GENERATED_BODY()
};
class ICharacterLifeCycleEventSource
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable) virtual void BindToOnDead(const FActorOnDeadDelegate& Handler) = 0;
    UFUNCTION(BlueprintCallable) virtual void UnbindFromOnDead(const FActorOnDeadDelegate& Handler) = 0;
};

UCLASS()
class AMyCharacter : public ACharacter, public ICharacterLifeCycleEventSource
{
    GENERATED_BODY()
public:
    virtual void BindToOnDead(const FActorOnDeadDelegate& Handler) override
    {
        OnDead.AddUnique(Handler);
    }
    virtual void UnbindFromOnDead(const FActorOnDeadDelegate& Handler) override
    {
        OnDead.Remove(Handler);
    }

protected:
    UPROPERTY()
    FMultiCastActorOnDeadDelegate OnDead;
};
```

# 背景

BPでは`interface`経由でイベントを購読することができません。

* `UInterface`は`UPROPERTY()`をもてない
* `UInterface`はUFUNCTION()`に制限がある
* `MulticastDelegate`は`UPROPERTY(BlueprintAssignable)` でないと BPノードが作成されない
* `MulticastDelegate`は`Delegate`を返すgetter関数は使えるが特になにもできない

上記問題が組み合わさることによって、uinterfaceは delegateを購読することができません。


```cpp: interface は UPROPERTY()を持てない
UINTERFACE() class UStatusProviderInterface : public UInterface { GENERATED_BODY() };
class IStatusProviderInterface
{
    GENERATED_BODY()

    // コンパイルエラー
    // interface は UPROPERTY()を持てない！
    UPROPERTY(BlueprintAssignable)
    FUpdateHPDelegate OnUpdateHPDelegate;
};
```

```cpp: interface は UFUNCTIONを持てない
UINTERFACE(BlueprintType, Meta = (CannotImplementInterfaceInBlueprint)) class UStatusProviderInterface : public UInterface { GENERATED_BODY() };
class IStatusProviderInterface
{
    GENERATED_BODY()

    // これもダメ
    // interface は UFUNCTION(BlueprintCallable)を持てない！
    UFUNCTION(BlueprintCallable)
    virtual FUpdateHPDelegate& GetOnUpdateHPDelegate();
};
```

いつもの方法では interfaceで delegateを扱えないことが分かりました。
C++で使うだけならいいのですが、BPに露出しようとするとうまくいきません。

# 対策

## 作戦A: EventsHolderとなるUObjectを公開する

まずは`Delegate`を持ちし`UObject`を定義します。
```cpp
// DelegateをラップするUObject
UCLASS(BlueprintType)
class UStatusEventsObject : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintAssignable)
    FUpdateHPDelegate OnUpdateHPDelegate;
};
```

`interface`に`UStatusEventsObject*`を返す`getter`を定義します。
ついでに現在値も返すメソッドを生やしておくとよいでしょう。
```cpp
UINTERFACE(BlueprintType, Meta = (CannotImplementInterfaceInBlueprint))
class UStatusProviderInterface : public UInterface
{
    GENERATED_BODY()
};
class IStatusProviderInterface
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable) virtual int GetHP() const = 0;
    UFUNCTION(BlueprintCallable) virtual int GetMaxHP() const = 0;
    UFUNCTION(BlueprintCallable) virtual UStatusEventsObject* GetStatusEvents() const = 0;
};
```
実装してみます。

```cpp: コンクリート実装
UCLASS()
class UMyConcreteStatusModel : public UObject
    , public IStatusProviderInterface
{
    GENERATED_BODY()
public:    
    UMyConcreteStatusModel()
    {
        StatusEvents = CreateDefaultSubobject<UStatusEventsObject>(TEXT("StatusEvents"));
    };

public:
    // IStatusProviderInterface interface begin
    virtual int GetHP() const override { return HP;}; 
    virtual int GetMaxHP() const override { return MaxHP;}
    virtual UStatusEventsObject* GetStatusEvents() const override {return StatusEvents;}
    // IStatusProviderInterface interface end
    
    void SetAndTryNotifyHp(int NewHP)
    {
        float OldHP = HP;
        HP = NewHP;
        if (OldHP != NewHP)
        {
            if(StatusEvents)
            {
                StatusEvents->OnUpdateHPDelegate.Broadcast(HP, MaxHP);
            }
        }
    }
    
protected:
    UPROPERTY()    int HP = 0;
    UPROPERTY()    int MaxHP = 1;
    UPROPERTY()    TObjectPtr<UStatusEventsObject> StatusEvents;
};
```

これはうまくいきます。
BP上では`GetStatusEvents()`関数からdelegateに対して `BindEventTo`/`UnBindEventFrom`が実行できます。

![](/images/ue5-interface-and-delegate/bp_status_provider_interface.png)

ただし、完全に抽象化したかったのに`UStatusEventsObject`という具象型が残っているのは残念です。まぁしょうがない。


## 作戦B: SingaleCastDelegate を引数に取るBindTo/UnBindFrom関数を定義

作戦Bは作戦Aの発想を逆転させたものです。
作戦Bの骨子は BP上で `CreateEvent`ノードや `CustomEvent`を引数に与えることです。
C++側で用意する`MultiCastDelegate`実体をBPに渡すのではなく、BPが用意する`SingleDelegate`実体をC++が受け取るよう逆転させます。

```cpp
// 1. 引数用の SigleCast delegateを定義する
DECLARE_DYNAMIC_DELEGATE_OneParam(FActorOnDeadDelegate, AActor*, DeadActor);
// 2. 保存用の Multicast Delegateを定義する
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMultiCastActorOnDeadDelegate, AActor*, DeadActor);

// 3. interfaceを定義する
UINTERFACE(BlueprintType, Meta = (CannotImplementInterfaceInBlueprint))
class UCharacterLifeCycleEventSource : public UInterface
{
    GENERATED_BODY()
};
class ICharacterLifeCycleEventSource
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable) virtual void BindToOnDead(const FActorOnDeadDelegate& Handler) = 0;
    UFUNCTION(BlueprintCallable) virtual void UnbindFromOnDead(const FActorOnDeadDelegate& Handler) = 0;

    //UnbindAllはお好みで用意してください
    //UFUNCTION(BlueprintCallable) virtual void UnbindFromOnDeadAll(const UObject* WorldContextObject) = 0;
};
```
実装します。
```cpp
// 4. interfaceを実装する
UCLASS()
class AMyCharacter : public ACharacter, public ICharacterLifeCycleEventSource
{
    GENERATED_BODY()
    AMyCharacter();
public:
    virtual void BindToOnDead(const FActorOnDeadDelegate& Handler) override
    {
        OnDead.AddUnique(Handler);
    }
    virtual void UnbindFromOnDead(const FActorOnDeadDelegate& Handler) override
    {
        OnDead.Remove(Handler);
    }

protected:
    // BlueprintAssignableはなくていい. BindTo/UnBindFromを自前で用意したから
    UPROPERTY()
    FMultiCastActorOnDeadDelegate OnDead;
};
```

実装できました。
BPノード上ではちゃんとEventPinとして公開されています。

![](/images/ue5-interface-and-delegate/bp_character_lifecycle_event_interface.png)

# 作戦A vs 作戦B
研究の結果２通りの方法でBP上からinterface のdelegateを扱えるようになりました。

どちらがいいか比較検討します。

## 作戦Aの特徴: Observer付きDataModel
作戦Aの最大の特徴は`EventSourceObject` という共通の具象型を引き回すことです。

具象型であるがゆえに、
1. 直接`UPROPERTY() FMulticastDelegate`を配置できる
2. Modelとしてデータの一元化ができる
3. `Replicate`や`SaveGame`も割と簡単
4. 一元化する故、神クラスになりうる

ロジックを持たずあくまで値を保持・同報することを目的としたコンテナに突っ込みます。
UI同期用にするのか、制御・通信・保存全般も含めたModelにするのかは設計次第です。

## 作戦Bの特徴: Event Source Interface
作戦Bの特徴は `interface`に完全に隠蔽できていることです。抽象にのみ依存しているため、可塑性が高いです。

1. `UPROPERTY()`を持たないからこそ、ユーザー側で`UPROPERTY()`の属性を制御できる。
2. イベント種の増加にはスケールできない

`ReplicatedUsing`や`SaveGame`や`VisibleAnywhere`など`UPROPERTY`属性を実装者側で決定できるというのは大変すばらしいことです。

ただし、イベント種が増えてくると実装が非常に大変になります。`HP`だけで2メソッド必要ということは、`MP`,`SP`,`Coin`,`Score`,`Gold`,`Equipment`,... ありとあらゆる属性パラメータを突っ込んだときその分だけメソッド量が増えます。`interface`を実装するすべての具象型でその対応が必要となってしまいます。メンテナンスは非現実的です。

ソフトウェア工学的には`interface`の細分化が良いのだろうと思います。
ゲーム仕様をうまくモデリングできれば、適切な粒度での`interface`定義が行えるのでしょう。ただし、ゲーム仕様そのものが固まりづらいため難しいです。

`interface`の細分化はBP実装者側の複雑度が増えていることにも注意が必要です。
BP編集時にはどの`interface`に`Cast`するの？どのメンバ`interface`を`get`するの？を考える必要があり、モノリシックな`interface`に全部収まっていた方が楽ちんです。
(BP編集性は`BlueprintFunctionLibrary`を充実させて頑張るという作戦もありではありますがね)

## パラメータ系は 作戦Aが良さげ
より注意深く観察すると、パラメータ系と純粋なイベント系でニーズが異なることに気付きます。

パラメータ系はその変化を`Delegate`経由で飛ばすのですが、現在値のそばに配置したいです。
このときは、真値を持つ`ViewModel`や純粋な`Model`としてモデリングした方がいいでしょう。

```cpp
// [現在値,最大値]のタプルで扱うことが適切なint要素に使うdelegate (HPなど)
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FMaxedIntDelegate, int, Value, int, MaxValue);

// パラメータ全部入りモデル
UCLASS(BlueprintType)
class UParamModelForUI: public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(Transient) int HP{0};
    UPROPERTY(Transient) int MaxHP{1};
    UPROPERTY(Transient) FMaxedIntDelegate OnUpdateHP;

    UPROPERTY(Transient) int MP{0};
    UPROPERTY(Transient) int MaxMP{1};
    UPROPERTY(Transient) FMaxedIntDelegate OnUpdateMP;

    //...その他いろいろ
};
```
本例ではUIへの値同期用という明確な目的があるので、UI用ViewModelと割り切ってみました。

セーブ、通信、制御用には別の専用Modelに変換するのがいい気がしています。
`ViewModel`に無理に`SaveGame` や `Replicated`属性を付けると拡張が大変かと思います。
UIにつかう表示値は保存しないという意図を表すため`Transient`を付与してみました。

分けたら分けたで、Modelが複数存在して同期や転送が面倒なときもあります。全部入りモノリシックModelもありです。
この辺りは本当にゲームドメイン次第なので、毎回悩ましいところです。

```cpp: 全機能入りモデル
// すべてのパラメータを握る神クラス
UCLASS(BlueprintType)
class UParamModelAllInOne: public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(SaveGame, Replicated) int HP{0};
    UPROPERTY(SaveGame, Replicated) int MaxHP{1};
    UPROPERTY() FMaxedIntDelegate OnUpdateHP;

    UPROPERTY(SaveGame, Replicated) int MP{0};
    UPROPERTY(SaveGame, Replicated) int MaxMP{1};
    UPROPERTY() FMaxedIntDelegate OnUpdateMP;

    //...その他いろいろ

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};
```

### 論理イベントは作戦Bが良さげ
例えば、キャラクタのライフサイクルイベントです。

* Spawn/Despawn
* Active/Deactive
* Dead/Revive
* Sleep/Awake
* Pause/Play
* Setup/TearDown
* LateBeginPlay/EarlyEndPlay

この辺りは比較的数が少なくて済み、様々なクラスで実装したいでしょうから作戦Bがいい気がします。ゲーム次第ですので、ドメインルールをよく観察すると見えてくるでしょう。

値や状態enumそのものではなく、状態遷移イベントを購読したいことから真値を保持しなくて済みます。真値を保持しなくて済むなら、上記例と異なり`SaveGame`や`Replicate`のことを考えなくて済みます。`MulticastDelegate`を保持するだけでいいですね。
もし状態遷移のPrev,Nextが重要になる場合はdelegateの引数に入れるか、`Actor`からgetしたら良いです。

```cpp
DECLARE_DYNAMIC_DELEGATE_OneParam(FActorOnDeadDelegate, AActor*, DeadActor);
DECLARE_DYNAMIC_DELEGATE_OneParam(FActorOnReviveDelegate, AActor*, ReviveActor);


UINTERFACE(BlueprintType, Meta = (CannotImplementInterfaceInBlueprint))
class UCharacterLifeCycleEventSource : public UInterface
{
    GENERATED_BODY()
};
class ICharacterLifeCycleEventSource
{
    GENERATED_BODY()
public:
    // 死んだとき
    UFUNCTION(BlueprintCallable) virtual void BindToOnDead(const FActorOnDeadDelegate& Handler) = 0;
    UFUNCTION(BlueprintCallable) virtual void UnbindFromOnDead(const FActorOnDeadDelegate& Handler) = 0;

    // 蘇生したとき
    UFUNCTION(BlueprintCallable) virtual void BindToOnRevive(const FActorOnReviveDelegate& Handler) = 0;
    UFUNCTION(BlueprintCallable) virtual void UnbindFromOnRevive(const FActorOnReviveDelegate& Handler) = 0;

    // などなど
};
```

状態やパラメータはActor経由で取得したらいいので、DelegateにAActor*を渡しておきましょう。
実装者は素直にMulticastDelegateを持つだけで済みました。
```cpp
UCLASS(BlueprintType)
class AMyCharacter : public ACharacter, public ICharacterLifeCycleEventSource
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable) virtual void BindToOnDead(const FActorOnDeadDelegate& Handler) override;
    UFUNCTION(BlueprintCallable) virtual void UnbindFromOnDead(const FActorOnDeadDelegate& Handler) override;
    UFUNCTION(BlueprintCallable) virtual void BindToOnRevive(const FActorOnReviveDelegate& Handler) override;
    UFUNCTION(BlueprintCallable) virtual void UnbindFromOnRevive(const FActorOnReviveDelegate& Handler) override;

protected:
    UPROPERTY() FMultiCastActorOnDeadDelegate OnDead;
    UPROPERTY() FMultiCastActorOnReviveDelegate OnRevive;
};
```

設計を頑張るなら Actorが直接`MulticastDelegate`を持つのではなく、`UHealthComponent`や`ULifeComponent`を用意するかもしれません。

```cpp: コンポーネントに委譲する例
void AMyCharacter::BindToOnDead(const FActorOnDeadDelegate& Handler)
{
    HealthComponent->OnDead.AddUnique(Handler);
}
```
どう実装するかは本稿の主題ではないので触れません。

# まとめ
`UInterface` を使って `Delegate`をBP上でうまく扱う方法について調べました。
以下の2つの方法がよさそうだとわかりました。

* イベントオブジェクトを導入する
* `BindTo`/`UnBindFrom` 関数を導入する

# オチ

UI用は[MVVMプラグイン](https://dev.epicgames.com/community/learning/tutorials/jDXO/unreal-engine-mvvm-ui)を使ったら全て解決しませんか？