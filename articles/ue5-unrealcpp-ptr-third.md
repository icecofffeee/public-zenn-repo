---
title: "UE5:Unreal Engineのポインタについてまとめた 後編"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: true
---
# はじめに

[UE5:Unreal Engineのポインタについてまとめた 前編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr)
[UE5:Unreal Engineのポインタについてまとめた 中編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr-second) 
の続きです。後編では、マネージドポインタについて記載します。

# ポインタ一覧再掲

### アンマネージドポインタ

| アンマネージド | 名前 | 補足 |
| ---- | ---- | ---- |
| `FCppStruct* Object;` | 生ポインタ | 使うべきでない |
| `TUniquePtr<FCppStruct> Pointer;`| ユニークポインタ | `std::unique_ptr`に該当 |
| `TSharedPtr<FCppStruct> Pointer;` | シェアードポインタ | `std::shared_ptr`に該当 |
| `TWeakPtr<FCppStruct> Pointer;` | ウィークポインタ | `std::weak_ptr`に該当 |

### マネージドポインタ

| マネージド | 名前 | 補足 |
| ---- | ---- | ---- |
| `UPROPERTY() Ubject* Pointer{};` | ハードオブジェクトポインタ | 古い書き方で今どき使わない。`TObjectPtr`使え |
| `UPROPERTY() TObjectPtr<UObject> Pointer;` | オブジェクトポインタ | ハードオブジェクトポインタの進化版。基本はこれ。 |
| `UPROPERTY() TSoftObjectPtr<UObject> Pointer;` | ソフトオブジェクトポインタ | UE5独自のやわらかい参照(後述) |
| `UPROPERTY() TWeakObjectPtr<UObject> Pointer;` | ウィークブジェクトポインタ | `TObjectPtr`の所有権を持たない版 |
| `UPROPERTY() TStrongObjectPtr<UObject> Pointer;` | ストロングブジェクトポインタ | `TObjectPtr`の所有権を持つ版 |
| `Ubject* Pointer{};` | ワイルドポインタ | UPROPERTY()としてリフレクションシステムに辿られない野生のポインタ。ダングリングポインタになる。バグなので直せ。`TObjectPtr`にしろ |

### 特別なマネージドポインタ

| マネージド | 名前 | 補足 |
| ---- | ---- | ---- |
| `UPROPERTY() TScriptInterface<IMyInterface> Pointer;` | Nativeインターフェースへのポインタ | pure C++なinteface型を指すポインタ(後述) |
| `UPROPERTY() TSubClassOf<UMyBase> Pointer;` | UCLASSへのポインタ | BP含め任意の`UMyBase`派生型を指すポインタ(後述) |
| `UPROPERTY() TObjectPtr<AActor> Actor;` | `AActor`へのポインタ | BP含め任意の`AActor`派生型を指すポインタ(後述) |
| `UPROPERTY() TObjectPtr<UActorComponent> Component;` | `UActorComponent`へのポインタ | BP含め任意の`UActorComponent`派生型を指すポインタ(後述) |
| `UPROPERTY() TArray<TObjectPtr<UObject>> PointerArray;` | `TArray`の中のポインタ | コンテナの中のオブジェクトポインタ |
| `UPROPERTY() TMap<TObjectPtr<UObject>, TObjectPtr<UObject>> PointerMap;` | `TMap`の中のポインタ | コンテナの中のオブジェクトポインタ |
| `UPROPERTY() TSet<TObjectPtr<UObject>> PointerSet;` | `TSet`の中のポインタ | コンテナの中のオブジェクトポインタ |

# 詳解

## `TScriptInterface` ポインタ

`TScriptInterface` は インターフェースを実装した`UObject`への参照を保持するポインタです。
`UObject` が `UInterface` インターフェースを実装する際に使用します。

簡潔に述べると、`TScriptInterface` は `TObjectPtr` の `interface`に特化した版です。

公式ドキュメントはこちら
[Unreal Engineのインターフェース](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/interfaces-in-unreal-engine)

前置きですが、C++にインターフェースという機能は存在しません。存在しませんが純粋仮想関数を宣言することで実質インターフェースクラスを定義することができます。以降は 純粋仮想関数のみを持ち一切フィールドをもたないclassのことを、C++界におけるinterface とみなして説明します。

UEでそのインターフェースを実装する場合 `UINTERFACE`を使用します。
説明のために簡単なインターフェースを宣言します。事例としてサウンド機能を取り上げます。

```cpp
namespace MyModule
{
    class MYMODULE_API ISoundService
    {
    public:
        virtual ~ISoundService() = default;
        virtual int32 PlaySound(const FName& SoundLabel) = 0;
    }
}
```
上記は、pure C++な native インターフェースですね。
このようなインターフェースを定義して、サウンドライブラリ側とゲームライブラリ側を疎結合にしつつどの実装を使用するかを選択可能にするかと思います。依存性逆転や依存性注入でよく使われる手法です。
しかしながら、案の定Unreal C++ではこのままでは使えません。
下記のように`UInterface`の作法に則ります。


```cpp
// 型リフレクションに認識させるためのUInterface型
UINTERFACE(MinimalAPI, Blueprintable)
class MYMODULE_API USoundService : public UInterface
{
    GENERATED_BODY();
}

// 実際のC++ Nativeな インターフェース型
class MYMODULE_API ISoundService
{
    GENERATED_BODY();

public:
    virtual ~ISoundService() = default;
    virtual int32 PlaySound(const FName& SoundLabel) = 0;
}
```

`UINTERFACE`は UHTにより解析されて `ISoundService`を認識します。
そのため Nativeインターフェースは `namespace` には入れられません。

UHTにより`ISoundService::UClassType`が自動定義されます。これが `USoundService`を指しておりますので、interface型から型情報を使って逆引きすることができるのです。
```cpp
ISoundService::UClassType::StaticClass()->Interfaces;
```

最初のPrefixが 「U」のものと「I」のものがあります。以降、この2つを使い分けます。

### TScriptInterfaceの実装
インターフェースの実装は、「I」付きのインターフェースを 任意のUObject派生型で実装します。

まずはテスト用にNullObjectパターンな実装を行います。
```cpp
UCLASS()
class MYMODULE_API UNullSoundService : public UObject, public ISoundService
{
    GENERATED_BODY();

public:
    // ISoundService interface
    virtual int32 PlaySound(const FName& SoundLabel) override
    {
        return 0;
    };
}
```

実際にはMetasoundやらWwiseやらADXやらのバックエンドにあった実装を行うことでしょう。
```cpp
UCLASS()
class MYMODULE_API UWwiseSoundService : public UObject, public ISoundService
{
    GENERATED_BODY();

public:
    // ISoundService interface
    virtual int32 PlaySound(const FName& SoundLabel) override
    {
        // AkSoundEngine系のAPI叩く...
        return Handle;
    };
}

UCLASS()
class MYMODULE_API UADXSoundService : public UObject, public ISoundService
{
    GENERATED_BODY();

public:
    // ISoundService interface
    virtual int32 PlaySound(const FName& SoundLabel) override
    {
        // ADX系のAPI叩く...
        return Handle;
    };
}
```
実装は本題と関係ないので省略します。これで`UINTERFACE`を実装できました。

#### Abstractクラスの実装
`UInterface`は Dynamic Multicast Delegateを持てません。戻り値で返せませんし、引数で受け取れません。
そこで、Dynamic Multicast Delegateをインターフェースで扱いたい場合、Abstractクラスを挟みます。

```cpp
UCLASS(Abstract)
class MYMODULE_API UAbstractSoundService : public UObject, public ISoundService
{
    GENERATED_BODY();

public:
    // ISoundService interface
    virtual int32 PlaySound(const FName& SoundLabel) override PURE_VIRTUAL(UAbstractSoundService::PlaySound, return 0;)

protected:
    // なんかプロパティ増やしたりする
    UPROPERTY(BlueprintAssignable)
    FMyDelegate OnSoundCalled;
}

UCLASS()
class MYMODULE_API UConcreteSoundService : public UAbstractSoundService
{
    GENERATED_BODY();
public:
    // ISoundService interface
    virtual int32 PlaySound(const FName& SoundLabel) override;
}
```

`interface`を継承した `abstract`クラスは、大変キモイのですが `override PURE_VIRTUAL`をつけて純粋仮想関数を仮実装しなければなりません。
これはUnreal Engineが Class Default Object:CDOをインスタンス化したいからです。
C++では純粋仮想関数をもつクラスはそれを実装しない限りインスタンス化できません。
通常の`abstract`クラスならば実装せず宣言するだけでよかったのですが、UEではCDOのために仮実装してFatalログを仕込んだうえで無理矢理インスタンス化できるように対策しています。

### TScriptInterfaceの生成

`UInterface`型を参照したいときは`UPROPERTY()`を使います。
C++およびBP両方で設定できます。

```cpp
UCLASS()
class UMyComponent : public UActorComponent
{
public:
    virtual void BeginPlay() override;

private:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TScriptInterface<ISoundService> SoundSerivce;
}
```

C++で設定する場合は、UObject派生型を `operator=`で設定するだけです。
```cpp
void UMyComponent::BeginPlay()
{
    UConcreteSoundService* Service = NewObject<UConcreteSoundService>();
    SoundService = Service;
}
```

明示的に `SetObject` + `SetInterface` を使うこともできますが、滅多に利用しないでしょう。

```cpp
    UConcreteSoundService* Service = NewObject<UConcreteSoundService>();
    SoundService.SetObject(Service);
    SoundService.SetInterface(Cast<ISoundService>(Service));
```
`TScriptInterface`は `IInterface`を実装した `UObject`を保持するポインタなので`UObject`ではないpure C++な native インスタンスはセットできません。

```cpp
struct FImpl : public ISoundService
{
    //...実装省略...
}
FImpl* pIpml =  = new FImpl();
SoundService.SetInterface(pIpm);
SoundService.SetObject(pIpml); // コンパイルエラー

SoundService = pImpl // コンパイルエラー
```

要するに`TScriptInterface` は `TObjectPtr`の`UInterface`版と言えるでしょう。参照している`UObject`が`IsValid`でなくなれば、`TScriptInterface`は`nullptr`として振る舞います。
```cpp
UConcreteSoundService* Service = NewObject<UConcreteSoundService>();
SoundService.SetObject(nullptr);
SoundService.SetInterface(Cast<ISoundService>(Service));

// c++ interface参照は活きているけど、UObject参照がないのでnot valid
check(SoundService.IsValid());

// 参照は正しいけどUObjectが死んでいるのでnot valid
SoundService.SetObject(Service);
Service->MarkAsGarbage();
check(SoundService.IsValid());

```

### TScriptInterfaceの破棄
参照を外すときは `nullptr`をセットします。
```cpp
void UMyComponent::EndPlay(...)
{
    SoundService = nullptr;

    // もしくは 空オブジェクトを再割り当てでもいい
    SoundService = {};

    // これでもいいけど冗長かと思う
    SoundService.SetObject(nullptr);
    SoundService.SetInterface(nullptr);
}
```
`TScriptInterface` は 特別なテンプレート型です。内部の`TObjectPtr`に対してちゃんと参照チェインを貼ります。そのため `UPROPERTY() TScriptInterface` として親から保持される限り、自身は子への参照を維持します。使い終わったら明示的に参照を外すことでGCを早めることができます。（所有者と寿命が同じなら別に参照を外さなくてもいいです）

### TScriptInterfaceの使用
カジュアルな nullチェックなら、`operator bool` と `operator->` でそのまま使えます。
まるでポインタのようです。
```cpp
void Func()
{
    if(SoundService)
    {
        SoundService->PlaySound(TEXT("SE.Attack01"));
    }
}
```

Garbageマーク済みな死んだ`UObject`にアクセスしたくないなら `IsValid`を使います。
`TScriptInterface`は直接`IsValid`をサポートしていないので中身の`TObjectPtr`を触ります。
```cpp
void Func()
{
    if(IsValid(SoundService.GetObject()))
    {
        SoundService->PlaySound(TEXT("SE.Attack01"));
    }
}
```

面倒くさいですね。オレオレでtemplate関数を用意するにしても、共通ヘッダをインクルードするかエンジン改造する羽目になるのでやはり面倒です。素直に`operator bool`を使えばいいんじゃないでしょうか。

```cpp
template<typename T>
bool IsValid(const TScriptInterface<T>& Interface)
{
    // IsValid(UObject*) へ移譲する
    return IsValid(Interface.GetObject());
}
```

### TScriptInterface の検索

`AActor`から `interface`を実装したコンポーネントを得るには `FindComponentByInterface<T>`を使います。
`Uinterface`を使う一番の理由がここにあると思います。 `UActorComponent`をinterface化すること`Actor`-`Component`間を疎結合にできます。
`OtherActor`から interface経由でアクセスできるということは、 `Actor`-`Actor`間も疎結合にできるということです。
Componentからは`GetOwner()`経由で`FindComponentByInterface` を使えば隣のinterfaceを触れますから、`Component`-`Component`間も疎結合になりました。やったー。

```cpp
void AProjectile::OnOverlapBegin(AActor* Other)
{
    // 弾がDamageableにHitしたらメッセージ飛ばす
    if(IDamageable* Damageable = Other->FindComponentByInterface<IDamageable>())
    {
        Damageable->HandleDamage(this, /*ダメージ情報*/);
    }
}
```
複数の実装を得たい場合は、 `GetComponentsByInterface`を使用します。

```cpp
void AProjectile::OnOverlapBegin(AActor* Other)
{
    // 弾がDamageableにHitしたらメッセージ飛ばす
    TArray<UActorComponent*> Interfaces = Other->GetComponentsByInterface(UDamageable::StaticClass());
    for(auto&& ActorComponent : Interfaces)
    {
        IDamageable pImpl = Cast<IDaamgeable>(ActorComponent);
        pImpl->HandleDamage(this, /*ダメージ情報*/);
    }
}
```
単数形が `FindComponentByInterface`, 複数形が `GetComponentsByInterface`とAPIが対称形じゃないので覚えるのが面倒くさいです。

`ISoundService`のように共通サービスなら`WorldSubsystem`に`ServiceProvider`パターンで保持するのがしていくのが楽ちんでしょう。`UGameInstance`でも `GameStatics`でもなんでもいいです。
```cpp
UCLASS()
class UMyServiceProviderSubsystem : public UWorldSubsystem
{
	GENERATED_BODY()
	
public:
	template<typename T>
	T* GetImpl(){ return InstanceMap.Find(T::StaticClass()); }

	template<typename T>
	void Register(TObjectPtr<UObject> Impl){ InstanceMap.Add(T::StaticClass(), Impl); }

	TMap<UClass*, TObjectPtr<UObject>> InstanceMap;
};


void AProjectile::OnOverlapBegin(AActor* Other)
{
    // 弾があたった音鳴らす
    auto* Subsystem = GetWorld()->GetSubsystem<UMyServiceProviderSubsystem>();
    ISoundService* SoundService = Subsystem->GetImpl<ISoundService>();
    SoundService->PlaySound(TEXT("SE.Projectile.Hit01"));
}
```

C++ネイティブからならこれでよいのですが、BPからは`tempalte`関数が呼べないません。`UFUNCTION`にして`FName`をキーにするとか、`TSubclassOf<T>`を引数に渡すとか工夫の余地はあると思います。

いずれにせよ、上記のように特定の実装に依存することなくinterface経由で疎結合にしたまま別クラスからアクセスできるようになりました。


### TScriptInterfaceを BPで扱う

正直[公式の説明](https://dev.epicgames.com/documentation/en-us/unreal-engine/interfaces-in-unreal-engine)が完璧なのでいうことありません。どう使うかよりも、どれを使うべきかの参考として下記をごらんください。

* `BlueprintCallable` - Interfaceの関数をBPから呼びたいときに使う
* `BlueprintImplementableEvent` - C++からInvokeするイベントで、その反応をBPで実装したいときに使う
* `BlueprintNativeEvent` - C++からInvokeするイベントで、その反応をC++でデフォルト実装しつつ、BP側でさらにオーバーライドしたいときに使う

具体例：
#### BlueprintCallable
ダメージ計算とかパス検索とかマスターデータ参照とかC++側で実装したい関数で、BPから呼び出したいものです。

#### BlueprintImplementableEvent
複雑なイベント発行をC++で実装しつつ、BP側でイベント駆動を実装したいときです。
OnDamageReact, OnPlayerJoin, OnLevelUp, OnItemPickupなどなど、C++で飛んでくるイベントに反応してBP側でUIやVFXやアニメ制御したいー、というときに使います。

#### BlueprintNativeEvent
BlueprintImplementableEvent の拡張版です。重めなので乱用しない方がいいです。
C++側でデフォルト実装を持ちつつもBPでオーバーライドしたいケースで利用します。
OnDead や OnDamage などサーバー同期が重要な局面で有効です。
ダメージ計算や通信といった複雑な処理をC++で実装しつつ、VFXやアニメの再生をBP側で差し込めます。

`BlueprintImplementableEvent`で良くない？という気がしますが、プロトのうちは全部コレでもいい気もします。

### TScriptInterfaceの詳解
`TScriptInterface`は`UCLASS`ではなく、テンプレート型です。
なので普通は`UPROEPRTY`になれないはずなのですが、どうして大丈夫なのでしょうか？
それはUHTことUnreal Header Tool にて `"TScriptInterface"`という型名がハードコードでサポートされているからです。
また、エンジン側に `TScriptInterface`に対応する `FInterfaceProperty` というプロパティ型が特別に実装されています。
UHTによる名指しのサポート + `FInterfaceProperty`によるシリアライズと `AddReferencedObject` がサポートされているおかげで、`TObjectPtr`同様に扱えるのです。

余談ですが、UHT側で名指しされているということはエイリアス型はダメだということです。
```cpp
using FInterfacePtr1 = TScriptInterface<ISoundService>;
typedef TScriptInterface<ISoundService> FInterfacePtr2;

UPROPERTY()
FInterfacePtr1 Ptr; // コンパイルエラー!
UPROPERTY()
FInterfacePtr2 Ptr2; // コンパイルエラー!
```
まったく同じ型なのに、UHTが解析できないため、エイリアス型を宣言することができません。悪しからず。


### 参考資料

* https://isaratech.com/ue4-declaring-and-using-interfaces-in-c/
* https://dev.epicgames.com/documentation/en-us/unreal-engine/interfaces-in-unreal-engine
* https://dev.epicgames.com/community/snippets/003/unreal-engine-interfaces-for-c-and-blueprint

## UPROPERTY() TSubClassOf<UObject> Pointer;

任意の`UCLASS`クラス派生型への参照を保持するポインタです。
SubClassという名の示す通り、型`T`のサブクラス(派生型)にマッチします。
サブクラス型以外を参照できないようになっているため、型安全に設定できます。

このポインタはインスタンスを指すのではなく、クラスを指します。つまり型情報を指します。
`T::StaticClass()` が入っています。

C# でいうところの`Type`インスタンスに該当します。

```cs
Type type = typeof(T);
```

クラス自体を参照できるため、どの実装を使うか何をインスタンス化して使うか、ということをEditor上で設定できます。依存性の注入をエディタ上で行えるのです。


### TSubClassOf<T> の設定
もっぱら `UPROPERTY()` で設定します。

C++実装の`UCLASS`クラスのみならずBPクラスも設定できるという点が強いです。
C++のデフォルトコンストラクタで初期化した値ではなく、BP上で設定したデフォルト値を使用してインスタンス化できるという点も便利です。


```cpp
ULCASS()
class AMyActor : public AActor
{
    virtual void BeginPlay() override;
    virtual void OnBeginOverlap(...) override;

    void FireProjectile(...);
protected:
    // こっちはクラス
    UPROEPRTY(EditDefaultsOnly)
    TSubClassOf<UMyDamageCalculator> DamageCalculatorClass;

    // こっちはインスタンス
    UPROEPRTY(Transient)
    TObjectPtr<UMyDamageCalculator> DamageCalculator{};

    // 発射する弾アクターのクラス
    UPROEPRTY(EditDefaultsOnly)
    TSubClassOf<AMyProjectile> ProjectileClass;
}
```

```cpp
AMyActor::BeginPlay()
{
    // ダメージ計算式は外部クラスに任せる
    // クラスからインスタンスを複製して生成する
    DamageCalculator = NewObjectFromTemplate(DamageCalculatorClass);
}

AMyActor::OnBeginOverlap(...)
{
    // 生成したクラスを使う
    auto DamageInfo = DamageCalculator.Calculate(...);
    this->TakeDamage(DamageInfo);
}

AMyActor::FireProjectile()
{
    // 弾丸アクターを発射する
    // 発射する弾丸クラスは外部から設定してもらう
    // Enumや FName ではなく直接 BPクラスを渡すことができる
    SpawnActor(ProjectilClass);
}
```

型安全なので全然違うクラスが入力されてアクセス違反するということはありません。

※ `TSubClassOf`はハード参照です。アセットが自動ロードされる点に注意してください。
普通は`SoftObjectPtr`にした方がいいと思います。今回は説明のために弾クラスを例に挙げましたが、装備品の切り替えなどがある場合はEquipmentマネージャーに移譲するなど、本例を鵜呑みにせずにちゃんと設計してください。あくまでこういう使い方ができるよ、という一例です。

`TSubClassOf` は データやアセットを持たない純粋な論理クラスを参照するとき向いていると思います。データを持つなら`TObjectPtr<UDataAsset>`, 重いアセットなら `TSoftObjectPtr<T>` がロード面で自由が利くでしょう。

### 参考
https://dev.epicgames.com/documentation/ja-jp/unreal-engine/typed-object-pointer-properties-in-unreal-engine

## `UPROPERTY() TObjectPtr<AActor> Actor;`

`AActor`派生型へのポインタです。

基本的には `UPROPERTY() TObjectPtr<UObject>` と同じ振る舞いを見せるのですが、`AActor::Destroy`を持ちます。
`AActor`は特別なclassです。Levelによってリスト化されて参照されており、基本的にGC回収されることはありません。そのため、明示的に`Destroy()`を呼び出す必要があるのです。

さて、この`Destroy()`は一体何を破棄するのでしょうか？

### `AActor::Destroy` は何をするか

中編で触れたとおり`UObject*`はマネージドポインタであり明示的に破棄することはできませんでした。しかしながら、`AActor`には`Destroy()`が存在します。
ユーザー側で明示的に破棄ができる気がしましたが、そんなことはありません。やっぱり`Destroy()`という名の破棄予約です。
そしてその実態は `TryDestroy()`と名づけるべきメソッドです。`Destroy()`関数にはいくつか例外が存在します。

1. `AWorldSettings`アクターは`Destroy()`できない
1. ネット同期されたアクターで自クライアントがAuthorityを所有していないもの(Replicatedなやつ)は`Destroy`できない

普通の開発者が気にするほどではないですね。破棄してはいけないものを破棄しないようにしてくれている、気軽に使える関数ということです。


`AActor::Destroy`は、自身から直接辿れる参照を外しまくります。

* LevelのActorListからの除去
* 親からのDettach
* 子のDettach
* 子コンポーネントの`UnRegister()` & `OnComponentsDestroyed()`
* `OverlapEnd`の実行(`PrimitiveComponent`経由で)
* ネットワークへの`Destroy`通知

`AActor`は`UActorComponent`の所有者ですから、自身の所有する全ての`UActorComponent`を`MarkAsGarbage`した上で、最終的に自身を`MarkAsGarbage`します。

### AActorはいつデストラクトされるか

他の`UObject*` と全く同様のタイミングです。ガベージコレクションに回収されたら初めて実行されます。

公式ドキュメントでは、`AActor`や`UActorComponent`は遅延破棄されるよ、という、さもこの2種の`派生型`だけ遅延破棄されるかのような誤読を招く表現で記載されていました。実際には全ての`UObject`派生型で適用されるマーク＆スイープ方式でGC回収されます。デストラクタが呼ばれるのはGC回収されたときです。

とはいうものの、`AActor`はワールドのアクターリスト内に参照されているため、到達不可能と判断されてGC回収されることはまぁありません。明示的に`AActor::Destroy`(もしくは`UWorld::DestroyActor`)によって破棄予約されて、`MarkAsGarbage`になった次のGCタイミングで回収されます。そして、そこから`BeginDestroy()-FinishDestroy()` という破棄シーケンスが開始されます。

詳しくは [Life Cycle Breakdown](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-actor-lifecycle#lifecyclebreakdown)をご覧ください。


### Actor Documents

* https://dev.epicgames.com/documentation/en-us/unreal-engine/actors-in-unreal-engine
* https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/GameFramework/AActor#remarks

## `UPROPERTY() TObjectPtr<UActorComponent> ActorComponent;`

`UActorComponent`派生型へのポインタです。`TObjectPtr<AActor>`と大体同じです。

`AActor`の項で既に説明済みですので、言及することがないのですが、改めて。

`UActorComponent`は `AActor` に所有されるため、寿命を同じくします。`AActor`は Levelから参照され続けている限り存在するため、`AActor`に所有される`UActorComponent`もまた存在します。`AActor:Destroy()`によって、`AActor`に所有される`UActorComponent`も破棄予約されてしまうため、参照を保持し続けていたとしても、破棄されます。

例えば`ACharacter`が破棄されたときそいつの`MovementComponent`だけが生き残って存在するのはおかしいので、この挙動は納得の仕様でしょう。


## `UPROEPRTY() TArray<UObject*> Objects;`

`TArray<T>` は`UPROPERTY`対応されている特別なコンテナです。

`TArray<T>` は`USTRUCT`ではない C++ nativeなテンプレートクラスなのですが、それにも関わらず`UPROPERTY`になれます。`TArray<UObject*>`に限らず 他のポインタ型も格納できます。

* `UPROPERTY() TArray<UObject*> Objects;`
* `UPROPERTY() TArray<const UObject*> ConstObjects;`
* `UPROPERTY() TArray<TObjectPtr<UObject>> Objects;`
* `UPROPERTY() TArray<TWeakObjectPtr<UObject>> Objects;`
* ... などなど

一体なぜでしょうか？
それは `TArray`に対する特別なテンプレート関数が実装されており、`GENERATED_BODY()`経由でその特別なテンプレート関数を呼び出す関数が自クラスに埋め込まれるからです。
つまり、Unreal Header Toolで名指しでサポートされており、リフレクションシステムの対象となっております。更に、`FArrayProperty`によって直接Propertyサポートがなされているからです。

### `EliminateReference` サポート

先に用語解説しておきます。UObjectが GC回収されたとき `UPROPERTY() TObjectPtr` や `UPROPERTY() TArray<TObjectPtr<T>>` の中身が nullptrに書き戻されます。
この機能には名前がないのですが、本稿では `Eliminate Reference`と呼称します。UEのエンジンソースコード上でそういう文言があったからです。

では、`EliminateReference`が `TArray`でどのようにサポートされているのか、その一端を読み解いていきましょう。

以下は本質だけに注目できるように略記した疑似コードです。(UE5はPrivateリポジトリなので、ソース公開するのも微妙ですから)
```cpp: UObjectGlobals.h
template<class UObjectType>
void AddReferencedObjects(TArray<UObjectType*>& ObjectArray, ...)
{
    EliminateReference(reinterpret_cast<UObject**>(ObjectArray.GetData()), ObjectArray.Num(), ...);
}

    template<class UObjectType>
void AddReferencedObjects(TArray<const UObjectType*>& ObjectArray, ...)
{
    EliminateReference(reinterpret_cast<UObject**>(const_cast<UObjectType**>(ObjectArray.GetData())), ObjectArray.Num(),...);
}

    template<class UObjectType>
void AddReferencedObjects(TArray<TObjectPtr<UObjectType>>& ObjectArray, ...)
{
    EliminateReference((FObjectPtr*)(ObjectArray.GetData()), ObjectArray.Num(), ...);
}

```

このtemplate関数は GerbageCollecter経由で呼び出されます。内部の`EliminateReference`という関数で`Object=nullptr;`という処理が走ります。※`EliminateReference`という名前の関数は存在しませんよ。あくまで疑似コード上での関数です。

めちゃくちゃ読みにくいですが、上から順に次の通りです。
* `TArray<T*>`版
* `TArray<const T*>`版
* `TArray<TObjectPtr<T>>`版

解説していきます。

#### `TArray<UObjectType*>` 版
`TArray<UObjectType*>` は `reinterpret_cast`で無理矢理`UObject**`に変換しています。`TArray<UObjectType*>` は連続したメモリ領域に確保されたポインタ配列であるため配列の先頭は`UObjectType**`です。`UObjectType`は`UObject`派生型なので、`UObjectType**`を`UObject**`にキャストすることは合法です。

#### `TArray<const UObjectType*>` 版
`TArray<const UObjectType*>` は `reinterpret_cast`と`const_cast`で無理やり`UObject**`に変換しています。先ほどのバージョンから`const_cast`で無理矢理constを取り除いただけで合法です。（dirtyではありますが、合法です）

今回やりたいことは`UPROEPRTY()`が付与された`TArray`の中身のポインタを`nullptr`にセットすることです。`const UObject**` は `UObject* const *`ではないので、`const_cast`が必要ないように思えます。しかし、忘れてはいけません。`UObjectType`がもつ`UPROPERTY`もまた再帰的にnullptrに書き戻したいのです。`const UObject**` では書き換えることができないので、しゃあなしで `const_cast` を使って `const`を外すしかありません。


#### `TArray<TObjectPtr>` 版
`TArray<TObjectPtr>` はもうむちゃくちゃなことしてます。 一度`TObjectPtr`配列の先頭アドレスを`FObjectPtr*`に変換して、`FObjectPtr*`用のオーバーロード関数に移譲しております。`TObjectPtr`のポインタ型は`TObjectPtr*` であって、`FObjectPtr*`ではありません。Cスタイルキャストでやっちゃってるけど安全なんでしょうか？

`TObjectPtr`の疑似コードを載せます。
```cpp
template<typename T>
struct TObjectPtr
{
    FObjectPtr ObjectPtr;
}
```
上記のように、`TObjectPtr`は `FObjectPtr`だけをメンバとして所有しているテンプレクラスです。継承もしていませんし、`virtual`関数もありません。メモリアライメントが合っているかぎり、`sizeof<TObjectPtr>`と`sizeof<FObjectPtr>`は同じになるはずです。`FObjectPtr`は8byteであり、64bit環境における生ポインタと同じサイズです。

構造体が構造体を所有するときそのメモリレイアウトはメンバの宣言順に並べられますから、`TObjectPtr`インスタンスのアドレスと`ObjectPtr`のアドレスは一致します。このときメモリアラインメントに合うようにパディングされますがフィールドを1つかしか持たない、かつ暗黙的に8byteアラインメントだと思うので、パティングはありません。

```cpp
TObjectPtr Instance{};
TObjectPtr* ObjPtr= &Instance;
FObjectPtr* InnerPtr = &Instance.ObjectPtr; // privateだからこんなコードは書けないが説明用の疑似コードだ

ensure((void*)ObjPtr == (void*)InnerPtr); // 同じアドレスを指している
```
となります。なので、`ObjectArray[0].ObjectPtr`を直接指すならばまだ納得がいきます。(いくか？)

```cpp
// privateだからこんなコードは書けないが説明用の疑似コードだ
FObjectPtr* InnerPtr = &ObjectArray[0].ObjectPtr;
```
上記は確かに`FObjectPtr*`です。

ここで`TObjectPtr[N]`という長さNの配列を考えましょう。この配列の先頭アドレスは配列そのものです。お忘れかもしれませんが、配列とポインタはコンパイラレベルでは大体同一です。配列はポインタとして扱われます

```cpp
TObjectPtr<T> ObjectPtrArray[N]{};
TObjectPtr<T>* Head = ObjectPtrArray; //添え字なしだと先頭アドレス
TObjectPtr<T>* Head = &ObjectPtrArray[0]; // [0]は (Head + 0) という操作で表現されている
```

次に、`TArray<TObjectPtr<T>>`について考えます。`TArray`はヒープの先頭アドレスと要素長と長さで表現されています。
(※`FDefaultAllocator`の場合)
```cpp
// 疑似コード. めっちゃ省略するとこんな感じ. 本当はTAllocator型がいて様々なデータ表現がなされている
struct TArray<TObjectPtr<T>>
{
    TObjectPtr<T>* Memory;
    int32 Size; // Memory確保サイズ
    int32 Num; //現在使用中の要素数
}
```

`TArray::GetData()`は確保したメモリ領域の先頭アドレスである `TObjectPtr* Memory;`を返します。
先述のように`TObjectPtr<T>`は`FObjectPtr`を1つのみ持つ構造体であるので、`TObjectPtr*` を `FObjectPtr*`に無理矢理キャストしても先頭部分は大丈夫です。
`GetData()`で得たポインタを`FObjectPtr*`にキャストしても先頭は大丈夫です。先頭は。

2番目以降の要素についても考えます。
`sizeof(TObjectPtr)`と`sizeof(FObjectPtr)`が一致していることからポインタ型のストライド量も一致しています。
よって、それぞれのポインタをインクリメントしたときに指すアドレスは一致します。セーフ。

```cpp
TArray<TObjectPtr> Array;
TObjectPtr* THead = Array.GetData();
FObjectPtr* FHead = &Array.GetData()->ObjectPtr;

ensure((uint8*) THead == (uint8*)FHead); // アドレス一致


TObjectPtr* TNext = THead + 1; // 8byte動くはず
FObjectPtr* FNext = FHeat + 1; // 8byte動くはず
ensure((uint8*) TNext == (uint8*)FNext); // アドレス一致

ensure((uint8*) TNext - (uint8*)THead == 8); // ストライドは正確に8byte
ensure((uint8*) FNext - (uint8*)FHead == 8); // ストライドは正確に8byte
```

というわけで、`TObjectPtr[]`は `FObjectPtr[]`と同じと見てヨシ！ということです。(ほんまか？)

ただし、これは正確に8byteであることが前提であり、メモリレイアウトに強く依存するため実装依存です。なので、謎のアーキテクチャ上では動かないでしょう。
1byteが8bitかも分かりませんが、現実的にはUE5がサポートするゲーム機は64bitメモリ環境ばかりですから、問題はないということです。

ともかくこれで どのようにして`UPROPERTY`な`TArray`の内部に納めた参照が`EliminateReference`されるのかよく理解できました。

全然型安全じゃなかったけど、アドレスを直接弄り回してパフォーマンスを優先する、ということなのでしょう。

## `UPROEPRTY() TSet<UObject*> Objects;`

`UObject`派生型を指す `TSet` です。 こちらも`UPROPERTY`対応されており、`EliminateReference`の対象となります。

こまかい挙動は`TArray`の項と同様です。 UnrealHeaderToolで `TSet`が名指しで対応されており、`FSetProperty`が実装されております。

template関数 `AddReferencedObjects`は
* `TSet<UObjectType*>`
* `TSet<TObjectPtr<T>>`

がサポートされています。

## `UPROEPRTY() TMap<TObjectPtr<T>, TObjectPtr<T>> Objects;`

`UObject`派生型をKey, Valueとする `TMap`です。こちらも`UPROPERTY`対応されており、`EliminateReference`の対象となります。

template関数 `AddReferencedObjects` は

* `TMap<UObjectType*, TValue>`
* `TMap<TKey, UObjectType*>`
* `TMap<UObjectType*, UObjectType*>`
* `TMap<TKey, TObjectPtr<T>>`
* `TMap<TObjectPtr<T>, TValue>`
* `TMap<TObjectPtr<T>, TObjectPtr<T>>`

TKeyとTValueの組み合わせ全通りがサポートされています。

## UPROPERTY コンテナの中身もハードオブジェクトポインタは非推奨

`UPROEPRTY() TArray<UObject*>` のように生ポをコンテナの中に入れるのはインクリメンタルGCに対応できないため非推奨です。もし使用している場合、ビルド時にメッセージが出ているはずです。

とにかく`TObjectPtr`を使いましょう。
コンテナの中に入れるときも、 `TObjectPtr`を入れましょう。

* `UPROEPRTY() TArray<TObjectPtr<T>>`
* `UPROEPRTY() TSet<TObjectPtr<T>>`
* `UPROEPRTY() TMap<TObjectPtr<T>, TValue>`
* `UPROEPRTY() TMap<TKey, TObjectPtr<T>>`
* `UPROEPRTY() TMap<TObjectPtr<T>, TObjectPtr<T>>`

メモリ効率、型安全性あらゆる面において、`TObjectPtr >= ハードオブジェクトポインタ` なので `TObjectPtr`がいいです。
実行時効率はパッケージなら多分測定できないぐらいしか変わらん。

## 対応されないコンテナ型

ここまでの話で一つ気づいたことがあります。
`TWeakObjectPtr`, `TStrongObjectPtr`に対する、`AddReferencedObject`関数が存在しないことに。

つまり、
* `UPROPERTY() TArray<TWeakObjectPtr<T>>` 
* `UPROPERTY() TSet<TWeakObjectPtr<T>>` 
* `UPROPERTY() TMap<TWeakObjectPtr<T>, TWeakObjectPtr<T>>` 

は `EliminateReference`の対象外っぽいという予測がつきます。

確かに`TWeakObjectPtr`は弱参照なのだから、`UPROPERTY()`だったとしても到達可能性を考慮してほしくないです。
参照先がGC回収された後は、`TWeakObjectPtr`は`nullptr`として振る舞います。`TWeakObjectPtr`内部にはIndexとシリアル値が入っているため 0ではないのですが、ただの無効値であることからアクセス違反はしません。無効値の場合は`nullptr`として振る舞うため使用上も問題なさそうです。

(※シリアルNoが一周するような長時間プレーの場合はどうなるんだろう......)

## 最後に例題
ここまでのポインタに関する理解を整理するために改めて考えましょう。次の例題を解いてみてください。

```cpp
UCLASS()
class AMySpawner : public AActor
{
    void SpawnSomeActor(FName NickName)
    {
        FActorSpawnInfo SpawnInfo;
        SpawnInfo.Name = NickName;
        SpawnInfo.Owner =this;
        TObjectPtr<AActor> Actor = GetWorld()->SpawnActor(SpawnInfo);
        Map0.Emplace(Actor, NickName);
        Map1.Emplace(NickName, Actor);
        Map2.Emplace(Actor, Actor);
    }

    void DestroySomeActor(FName NickName)
    {
        if( TObjectPtr<AActor>* Ptr = Map1.Find(NickName))
        {
            TObjectPtr<AAcor> Actor = *Ptr;
            Actor->Destroy();

            // ※ わざとMapからRemoveしません
            // Map0.Remove(Actor);
            // Map1.Remove(NickName);
            // Map2.Remove(Actor);
        }
    }

    void ForceGarbageCollection()
    {
        GEngine->ForceGarbageCollection(true);
    }

    void LogMap()
    {
        UE_LOG(LogTemp, Warning, TEXT("Map0 %d"), Map0.Num());
        UE_LOG(LogTemp, Warning, TEXT("Map1 %d"), Map1.Num());
        UE_LOG(LogTemp, Warning, TEXT("Map2 %d"), Map2.Num());

        // Map[0]等にアクセスするとクラッシュしうるよ!
    }

    UPROPERTY() TMap<TObjectPtr<AActor>, FName> Map0;
    UPROPERTY() TMap<FName, TObjectPtr<AActor>> Map1;
    UPROPERTY() TMap<TObjectPtr<AActor>, TObjectPtr<AActor>> Map2;
}
```

さて、以下のような順番で走査したとします。
1. `AMySpawner`をレベル上に配置。
1. `MySpawner->SpawnSomeActor(TEXT("Hogehoge"));` を実行
1. `MySpawner->DestroySomeActor(TEXT("Hogehoge"));` を実行
1. `MySpawner->LogMap();` を実行
1. `MySpawner->ForceGarbageCollection();` を実行
1. `MySpawner->LogMap();` を実行

このとき、手順4と手順6の時点でMap内のハード参照されている、AActorはどうなるでしょうか？考えてみましょう。

これまで調べた要点をまとめます。

1. `AMySpawner`およびスポンした`AActor`はレベルのアクターリストに載っており、明示的に`Destroy()`しない限り、GC回収されない。
1. `AActor`は`AMySpawner::Map0` により `UPROPERTY()`としてハード参照されており参照グラフ上は到達可能。
1. `AMySpawner::DestroySomeActor` で `AActor`が 明示的に`Destroy`された。
1. `AMySpawner::DestroySomeActor`ではわざと`Map`から`Remove`しておらす参照が残っている。
1. `AActor::Destroy`は破棄予約であり実際のメモリ解放はGC回収まで遅延される。

仮説を立ててみましょう。

* 仮説A: `AActor`は `PindingKill`状態であるが、`AMySpawner`から到達可能であるためGC回収対象外である。 `AActor`はメモリに残り続けるはずだ。
* 仮説B: `AActor`は `Desroy()`されたのだから、GC回収される。`AMySpawner::Map0-3` には `EliminateReference`により nullptrがセットされる。
このとき `Map0`および`Map2`は `Key == nullptr`なエントリーが残り続けるため、 Mapの一意性が破壊される。

どちらもありえそうですね。実際にDebugBuildのPIEでカジュアルに実験してみました。デバッグブレイクを張って、メモリ上の値を覗いてみます。

はい、答えは仮説Bが真でした。

手順6の時点で

* `Map0-2`の`Num()`は1
* `Map0` には `(nullptr, "Hogehoge")`なタプルが
* `Map1` には `("Hogehoge", nullptr)`なタプルが
* `Map2` には `(nullptr, nullptr)`なタプルが

それぞれ格納されていました。`ForceGarbageCollection(true)`の威力は絶大ですね。
ちなみに、手順4の時点では`Map0-3`内の全てのActorが活きてはいましたが、`RF_MirroringGarbage`フラグはちゃんと`true`になっていました。`Destroy`したのでちゃんと死にかけです。

さて、今回はつまらない例だっため `DestroySomeActor`で`Map`から`Remove`しないのが悪い、と思うかもしれません。しかしながら、`WorldSubsystem`や `オレオレManager`などで `Actor`や`UObject`を `RegisterActor`/`UnRegisterActor`によりMap管理する場合はどうでしょうか？
検索性をあげるためによくやりますよね？レベルを跨いだときやActorがデスポンした瞬間に`TMap`内の`Actor`が`nullptr`ライトバックされることで`TMap`が破壊されることが予測されますね。

上記は有名な問題ですので、知っている人は`TMap<TObjectPtr<T>, TValue>`なんか使いません。
もしやるならば、 `TMap<FObjectKey, TValue>` を使用します。



# 最後に

めちゃくちゃ長くなりました。

UE5のポインタ周りはコンテナ型も巻き込んで把握しなければならず大変でした。本稿が皆様の一助となりますように。