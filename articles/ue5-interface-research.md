---
title: "UE5:Unreal EngineのInterfaceについてまとめた"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: true
---

# はじめに
Unreal Engine 5 の `interface`機能とくに `UInterface`についてまとめます。

個人的に `UInterface`について理解が足らないと感じており、いまいちどう書けばいいのか、使いどころや選定基準が曖昧であったため、自己学習および将来の自分のためのメモとしてまとめることとします。さんざん擦られてきた`UInterface`ですがイマイチ深く踏み込んだすべての疑問を解決してくれる記事がなかったので調べました。
主に C++ 側での実装についてまとめます。

BP側の`interface`機能については 世にある記事をご参照ください。

- [【C++】インターフェース](https://zenn.dev/posita33/books/ue5_starter_cpp_and_bp_001/viewer/chap_03_cpp-interface)

# 検証環境
UE 5.5.4 ソースビルド

# interfaceについて
C++にはいわゆる`interface` と呼称される機能はありません。

本稿では純粋仮想関数を持つ`class/struct` のことを `interface` と呼称します。
以下、理解しやすさのために`interface`型には接頭辞`I`を付けて宣言することとします。

```cpp
// いわゆるインターフェース
class ICancelable
{
public:
    virtual void Cancel() = 0;
};
```

`UInterface`との差別化のため、本稿では`pure C++`で実装されたインターフェースのことを`native interface`と呼称します。

# UInterface の使い方
`UInterface` とは `Unreal C++` における インターフェース機能のことです。

初めに公式ドキュメント [UnrealEngineの インターフェース](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/interfaces-in-unreal-engine) をご一読ください。
`UInterface`は 接頭辞`U`で始まる型と接頭辞`I`で始まる型を必要とします。
以降、`UInterface`の使い方について述べます。

# 1: C++ Only UInterface
「C++で 完結する `UInterface`」 のことを `C++ Only UInterface`と呼称しましょう。
`C++ Only UInterface`は BPで一切使わない C++ で定義・実装・使用・保持する`UInterface`です。UE5でも `native interface`は利用できるのですが、`UObject`に実装する場合は`UInterface`を使った方がいいです。BP側には一切情報を露出させないようにしてみます。

`C++ Only UInterface`は `native interface` と使いどころがほぼ同じです。

## C++ Only UInterfaceの定義
サンプル事例として Idを提供する`interface`を挙げましょう。

```cpp
#include "CoreMinimal.h"

/**
* 例：オレオレId型で識別されしものが実装すべきinterface
* FFooObjectId の実際の型は主旨と無関係だから割愛.
*/
struct FFooObjectId{};

UINTERFACE(meta = (CannotImplementInterfaceInBlueprint))
class MYMODULE_API UFooObjectIdProvider : public UInterface
{
    GENERATED_BODY()
};

class MYMODULE_API IFooObjectIdProvider
{
    GENERATED_BODY()
public:
    virtual FFooObjectId GetFooObjectId() const = 0;
};
```
★ポイント
* C++でしか使わないため純粋仮想関数で定義してよい
* `PURE_VIRTUAL`マクロは不要
* `UINTERFACE()` 部分には `BlueprintType`と`Blueprintable`は付与しない
* C++でしか使わないため`CannotImplementInterfaceInBlueprint` を付与してBP実装させない

## C++ Only UInterfaceの使用
`Unreal C++`に適合した方法で使用します。

例として、Hitした相手アクターの具象型を知らずにinterfaceを実装しているかで操作する事例です。相手が弾丸なのか壁なのか味方なのかは`interface`経由で得てみましょう。

```cpp
UCLASS()
class AUserActor : public AActor
{
    // HitしたOtherアクターがIdProviderかをチェックする
    // 相手のIdから相手が何者かを知れる
    void OnHit(AActor* Other)
    {
        if(!IsValid(Other)){ return; }

        // 使い方1: Castする
        if(const IFooObjectIdProvider* PureProvider = Cast<IFooObjectIdProvider>(Other))
        {
            const FFooObjectId Id = PureProvider->GetFooObjectId();
            if(Id == 対象なら)
            {
                // IFooObjectIdProvider* は nativeポインタなので 直接TScriptInterfaceに格納できない
                // TScriptInterface::operator=は UObject系を引数に取る
                HitObject = Other;
                // ただしTWeakInterfacePtrは nativeポインタを直接入れられる
                 // TWeakInterfacePtr::operator=は interface型を引数に取る
                HitObjectWeak = PureProvider;
            }
        }

        // 使い方2: TScriptInterfaceコンストラクタ呼び出し
        // TScriptInterface::operator bool による比較があるのでifスコープが使える
        if( TScriptInterface<IFooObjectIdProvider> Provider(Other))
        {
            const FFooObjectId Id = Provider->GetFooObjectId();
            if(Id == 対象なら)
            {
                HitObject = Provider;
                HitObjectWeak = Provider; //TScriptInterfaceからTWeakInterfacePtrへの変換
            }
        }

        // 使い方3: Implements()
        if( Other->Implements<UFooObjectIdProvider>())
        {
            /// 省略
        }

        // 使い方4: ImplementsInterface()
        if( Other->GetClass()->ImplementsInterface(UFooObjectIdProvider::StaticClass()))
        {
            /// 省略
        }
    }

protected:
    UPROPERTY()
    TScriptInterface<IFooObjectIdProvider> HitObject;

    UPROPERTY()
    TWeakInterfacePtr<IFooObjectIdProvider> HitObjectWeak;
};
```

使い方1は `Cast<T>`および`null`チェック方式です。`Cast<T>`は`IsValid()==false`であるときやキャストできないときは`nullptr`が返りますので`null`チェックで十分なのです。この方式は対象が`T`型を実装しておりかつそれを使いたいときに便利です。

使い方2は`TScriptInterface` に格納してみる方式です。コンストラクタで`UObject`を代入すると、その`UObject`が`T`型を実装しているかチェックします。`T`型を実装している場合、有効な`TScriptInterface`を構築します。実装していない場合、`TScriptInterface`は`nullptr`として振る舞います。この方式は対象がT型を実装しているときに`UPROPERTY()`としてkeepしたいときに便利です。

使い方3は使い方4のシンタックスシュガーです。`template`関数であるため`constexpr`なコンパイル時チェックが走るという点で後述の使い方4よりも優れています。

使い方4は型情報で判定する方式です。この方式は対象`UObject`が`T`型を実装していることに意味があるときに使います。型タグで判断するような事例で便利です。

:::message
`Implements<T>()`関数には`U`で始まる型をtemplateパラメータとして渡します。ややこしいです。`I`の方を渡すとコンパイルワーニングが出るのでビルドログをちゃんとみましょう。
:::

## C++ Only UInterfaceの保持

`interface*`を保持するには`TScriptInterface`に格納します。
弱参照で保持するには `TWeakInterfacePtr`に格納します。

`UFooObjectIdProvider`は `UObject`派生型なのですが、`IFooObjectIdProvider` は`native class`ですからこのままでは`UPROPERTY()`として扱えません。そこで `TScriptInterface`に格納する必要があるのです。

```cpp
UCLASS()
class UFooObjectManager : public UObject
{
    GENERATED_BODY()

public:
    void RegisterObject(UObject* Object)
    {
        TScriptInterface<IFooObjectIdProvider> Instance(Object);
        check(Instance); // interfaceを実装していないものを登録してはいけない！

        Objects.Add(Instance->GetFooObjectId(), Instance);

        // TScriptInterface -> TWeakInterfacePtr変換はassign operatorで簡単に行える
        ManagedObjectWeak = Instance;
    }

    void UnregisterObject(const FFooObjectId& Id)
    {
        // 略
    }

private:
    // Interface 型をkeepするときは 
    // UPROPERTY() TScriptInterfaceにすること
    // GCのReachable判定で到達可能にしてくれるのでGC回収を妨げる
    UPROPERTY()
    TScriptInterface<IFooObjectIdProvider> ManagedObject;

    // GC回収を妨げたくないときは TWeakInterfacePtrで弱参照に
    // UPROPERTY() は付与できない
    TWeakInterfacePtr<IFooObjectIdProvider> ManagedObjectWeak;

    // TScriptInterfaceはコンテナ型に格納してもよい
    UPROPERTY()
    TMap<FFooObjectId, TScriptInterface<IFooObjectIdProvider>> Objects;
};
```

ラムダキャプチャするときは 基本的に`TWeakInterfacePtr`を使って弱参照でキャプチャしましょう。`TWeakInterfacePtr` はコンストラクタか`operator=`を使います。引数には`Interface*`を与えます。

```cpp
void UFooObjectManager::FooAsync()
{
    // 1. コンストラクタ を使う oneliner
    DoAsync([Weak = TWeakInterfacePtr(ManagedObject.GetInterface())]()
    {
        if(IFooObjectIdProvider* Provider = Weak.Get()){...}
    });

    // 2. operator= を使う
    TWeakInterfacePtr<IFooObjectIdProvider> Weak2 = ManagedObject.GetInterface();
    DoAsync([Weak2]()
    {
        if(IFooObjectIdProvider* P = Weak2.Get()){...}
    });
}
```

* `TObjectPtr<U>` ←→ `TScriptInterface<T>`
* `TWeakObjectPtr<U>` ←→ `TWeakInterfacePtr<T>`

`TScriptInterface` は内部に`UObject*`をもっていますので、`UPROPERTY()`であるかぎりGCを妨げることができる優れものです。

## C++ Only UInterfaceの実装
`interface`の実装は2番目以降に継承します。
`UInterface`を実装せしものは `UObject`派生型でなければなりません。

`UObject`派生型であれば`AActor`でも`ActorComponent`でもなんでも構いません。

```cpp
UCLASS()
class AConcreteImplActor : public AActor // 継承の1番目は必ずUObject派生型
    , public IFooObjectIdProvider   // interface継承は2番手以降でないとダメ
{
public:
    // IFooObjectIdProvider interface begin
    virtual FFooObjectId GetFooObjectId() const override { return Id; }
    // IFooObjectIdProvider interface end

private:
    // 実装は好きにしていい
    // 例としてエディタで設定するものとする
    UPROPERTY(EditAnywhere)
    FFooObjectId Id;
};
```
2番目以降に`interface`を継承せねばならないのはおそらくシリアライズ処理の関係です。
`this`ポインタを`UObject*`にc-styleキャストしたりするのでメモリレイアウト上`UObject`が先に来ないと困るからと思われます。

# 2: BP Callable UInterface

便宜上、「C++で実装してBPから使うだけのUInterface」のことを`BP Callable UInterface`と呼称しましょう。BPから関数を呼び出すだけでBP実装はしない`interface`です。

BP側は具象型に依存する必要がなくなるため、C++側での実装変更が容易になります。全BPの親クラス差し替えやリコンパイルは大変なので、BP側は具象型ではなく`interface`依存にしておくと将来楽になれるはずです。
BPコンパイル時間も速くなるようです。

## BP Callable UInterfaceの定義

サンプル事例としてUI用のinterfaceを定義します。
キャラクターのステータス表示UIやBP上でステータスごとの分岐に使う想定です。

```cpp
#include "CoreMinimal.h"

UINTERFACE(BlueprintType, meta=(CannotImplementInterfaceInBlueprint))
class UBuffStatusProvider : public UInterface
{
    GENERATED_BODY()
};

class IBuffStatusProvider
{
    GENERATED_BODY()
public:

    /** 毒状態ならばtrueを返す
    * UIに毒アイコンを出すために使用してよい
    */
    UFUNCTION(BlueprintCallable)
    virtual bool IsPoison() const = 0;
};
```
★ポイント

* `UINTERFACE(BlueprintType, meta=(CannotImplementInterfaceInBlueprint))`なclass
* `NotBlueprintable`は使わない
* `UFUNCTION(BlueprintCallable)` な純粋仮想関数

`UINTERFACE(BlueprintType, meta=(CannotImplementInterfaceInBlueprint))`でBPでの使用・変数・`Cast`できるけど、`interface`を実装したり`override`できないことを明言しています。

`BlueprintType`はこのインターフェースがBP上で変数として保持できるようにするため付与しています。BP変数は`TScriptInterface<T>`として保持されるようです。

`CannotImplementInterfaceInBlueprint` でこのインターフェースの実装をBPでできなくします。BPの `ClassSettings` > `Implement interface`欄のリストに表示されません。今回は C++で実装してBPからは関数呼び出しだけをできるように制限したい事例ですので適切です。

`NotBlueprintable`でもBP実装は防げるのですが、`Cast`やBP変数への保持ができなくなります。それは使い勝手が悪いので`CannotImplementInterfaceInBlueprint`の方がよいです。

BPに公開したい関数に`UFUNCTION(BlueprintCallable)` を付与します。
純粋仮想関数をBP上からメッセージ経由で適切に呼び出せるようになります。

:::message
interfaceのAPIは、実際のゲーム開発では`FGameplayTag`を使ったり、Delegateでイベント駆動にするなどもっと汎用的な`interface`にした方がいいです。例なので色々省略してます。
:::


## BP Callable UInterfaceのC++での使用

C++上は前回と全く同じです。Castして使ってください。
```cpp
if( IBuffStatusProvider* Provider = Cast<IBuffStatusProvider>(Object))
{
    bool bIsPosion = Provider->IsPoison();
}
```

## BP Callable UInterfaceのBPでの使用
この`UInterface`でやりたいことです。

BP上は`Target`に対して `IsPoison(Message)` ノード か `IsPoison`ノードを使います。

![](/images/ue5-interface-research/bp_interface_message_node.png)


* お手紙アイコンがついていない方が`Interface`への`Call Function`ノード
* お手紙アイコンがついているノードが`Interface Message`ノード

です。

`Interface Message`は `Target`がこの`interface`を実装していれば関数を呼び出し正しい値を返します。実装していなければ内部実行せずに戻り値型のdefaultバリューを返し次の実行ピンへ進みます。詳しくは詳解をご参照ください。

インターフェースを実装しているかは `Does Object Implement Interface`ノードでチェックできます。`Cast`ノードを使ってもいいです。

![](/images/ue5-interface-research/bp_does_object_implement_interface_node.png)




## BP Callable UInterfaceのC++での保持

`C++ Only UInterface`と同じです。

## BP Callable UInterfaceのBPでの保持

普通にBP変数でInterface型を指定するだけです。

## BP Callable UInterfaceのC++での保持

`C++ Only UInterface`と同じです。

## BP Callable UInterfaceのBPでの保持
`CannotImplementInterfaceInBlueprint`を付与して明示的にできないようにしているのでできません。想定通りです。一応確認しました。

比較のために実装可能な`IBuffStatusProvider2`を定義しました。
```cpp
UINTERFACE(BlueprintType)
class UBuffStatusProvider2 : public UInterface{GENERATED_BODY()};
class IBuffStatusProvider2{ GENERATED_BODY()
public:
    UFUNCTION() virtual bool IsPoison2() const = 0;
};
```

![](/images/ue5-interface-research/bp_interface_implement.png)

`IBuffStatusProvider` はリストビューに表示されず、`IBuffStatusProvider2`が表示されていますね。`CannotImplementInterfaceInBlueprint`が付与されているので`IBuffStatusProvider`実装できないのは正しい望んだ挙動です。一方、`IBuffStatusProvider2`は`CannotImplementInterfaceInBlueprint`も`NotBlueprintable`も付与されていないので実装できてしまいますね。

:::message
`interface`を実装することと関数を`override`することは別概念です。純粋仮想関数があっても`BP`で`interface`を実装するだけならばBPコンパイルエラーになりません。純粋仮想関数をBP上で`override`したときはじめてBPコンパイルエラーになります。
:::

:::message
`NotBlueprintable`により`interface`を実装だけならできるという仕様は`Does Object Implement Interface`ノードの挙動に関わります。
:::

---

# 3: BP Implementable UInterface
C++側で呼び出しを担いBPがinterfaceを実装するパターンです。
便宜上、C++で使用してBPで実装するUInterfaceのことを`BP Implementable UInterface`と呼称します。

サンプル事例としてダメージリアクションをBP上で実装します。ダメージリアクション呼び出し自体はC++上で実装される複雑なダメージ制御内で行われるものとします。BP側ではやられモーションやIK調整などC++から与えられた文脈に応じて制御するものとします。

## BP Implementable UInterfaceの定義

```cpp
#include "CoreMinimal.h"

UINTERFACE(BlueprintType, Blueprintable)
class UDamageReaction : public UInterface
{
    GENERATED_BODY()
};

class IDamageReaction
{
    GENERATED_BODY()
public:

    /** BPで実装する関数 */
    UFUNCTION(BlueprintImplementableEvent)
    void PlayReaction(FContext& Context);

    //virtual修飾は付けない
    //UFUNCTION(BlueprintImplementableEvent)
    //virtual void PlayReaction(FContext& Context);
}
```

★ポイント：
* `UINTERFACE(BlueprintType, Blueprintable)` な class
* `UFUNCTION(BlueprintImplementableEvent)` な非仮想関数

BPで実装するので`Blueprintable`が必須です。

`BlueprintType`はお好みでどうぞ。BPから呼び出せる関数が一切ないため`BlueprintType`はなくてもいい気がしますが`==`ノードや`IsValid`で判定したいときは`BlueprintType`が必要になるでしょう。

BPで実装する関数は、`UFUNCTION(BlueprintImplementableEvent)` を付与します。

`BlueprintCallable` を付与するかしないかは設計次第です。今回の事例では C++に呼び出し責務をすべて任せてBP側に実装責務を負わせる設計にしますので`BlueprintCallable`を付与しません。


### この場合 virtualにしない
大変ややこしいですが、`BlueprintImplementableEvent`ならば `virtual`関数にはしません。
これは`virtual`にするとC++で`override`できちゃうからと推察されます。誤って`BlueprintImplementableEvent`な関数に`virtual`をつけるとUHTが怒ってくれるので間違えることはありません。

## BP Implementable UInterfaceのC++での使用
C++ で呼び出すときはUHTによって自動生成される`static`関数を使用します。
この関数のことを`Thunk`関数と呼びます。BP側で実装された関数を呼び出すためにはUHTにより自動生成された特別な関数を呼び出す必要があるのです。

`Thunk`関数は `Execute_***`で始まる関数です。
今回は`IDamageReaction::Execute_PlayReaction()`となります。


```cpp: 使用例
void DamageLogic::ResolveDamage(AActor* Offence, AActor* Defence)
{
    //複雑なダメージロジック...
    FContext Context;
    Context.Offence = Offence;
    Context.Damage = 123;

    ...中略...

    // 対象が被ダメージリアクションを実装せしものなら呼び出す
    if(Defence->Implements<UDamageReaction>())
    {
        IDamageReaction::Execute_PlayReaction(Defence, Context);
    }
}
```

`Thunk`関数経由での呼び出しになるため、`Cast<T>`は使いません。型情報から判断するために`Implements<T>()`でチェックします。

このように`interface`にすることで、C++側は対象がキャラクターのようなリアクションを取るものなのか、壁や床のようにリアクションを取らないものなのかを気にせずに実装できました。
(`interface`ではなく`Delegate`を使ったイベントストリームにする作戦もありっちゃありだと思います。設計次第)

## BP Implementable UInterfaceのBPでの使用
意図的に `BlueprintCallable`を外してできなくしているのでできません。


## BP Implementable UInterfaceのC++の保持
C++ Only UInterfaceと同様。

## BP Implementable UInterfaceのBPの保持
普通に BP 変数にするだけ。

## BP Implementable UInterfaceのC++実装
C++ Only UInterfaceと同様。

ただし、`virtual`関数を持たないのでなんも`override`できません。

## BP Implementable UInterfaceのBP実装

BP側で実装して overrideします。

![](/images/ue5-interface-research/bp_interface_implement.png)

1. `ClassSettings` を押す
1. Implemented Interfaceの欄の Addボタンを押す
1. リストビューからインターフェース型を選択する

これで `Interface`を実装できました。次は関数を`override`します。
詳しい操作は以下のリンク先をご参照ください。

https://papersloth.hatenablog.com/entry/2018/09/19/195013#BlueprintImplementableEvent


`override`できました。これにて、C++から適切なタイミングで呼び出されるはずです。


# 4: BP Native UInterface

便宜上、C++で実装・使用してBPで実装・使用・overrideする全部入りUInterfaceのことを`BP Native UInterface`と呼称します。（いい名前が思いつかない）

全部入りです。ここまでくるとベースクラスでいいじゃん感がでてきます。とはいえ、ダイアモンド継承の危険を減らしたり、任意の`UObject`に`interface`を実装できるという点では優れています。

## BP Native UInterfaceのC++定義
C++で定義することで、C++でもBPでも 使用・overrideできるようにします。

サンプル事例としてC++/BP連携の塊であるインタラクション機能を挙げます。
典型例としてレベル上の宝箱を「開ける」というインタラクトを想定します。
他にもスイッチを「押す」、NPCに「話す」といった用途にも使えます。

```cpp
#include "CoreMinimal.h"

UINTERFACE(BlueprintType, Blueprintable)
class UInteractable : public UInterface
{
    GENERATED_BODY()
};

/**
 * インタラクト用インターフェース
 * - CanInteract: インタラクト可否
 * - GetInteractionText: UI 表示テキスト
 * - OnInteract: 実際のインタラクト処理
 */
class IInteractable
{
    GENERATED_BODY()
public:
    /** インタラクト可否を返す */
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interactable")
    bool CanInteract(AActor* Interactor) const;

    /** UI に出すテキスト */
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interactable")
    FText GetInteractionText() const;

    /** 実際のインタラクト処理（開く、拾う、話しかけるなど） */
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interactable")
    void OnInteract(AActor* Interactor);

    // UFUNCTIONに virtualは付けない
    // virtual bool CanInteract(AActor* Interactor) const;
    // virtual FText GetInteractionText() const;
    // virtual void OnInteract(AActor* Interactor);

    // デフォルト実装する場合は_Implementationを付ける
    // virtual bool CanInteract_Implementation(AActor* Interactor) const;
    // virtual FText GetInteractionText_Implementation() const;
    // virtual void OnInteract_Implementation(AActor* Interactor);
}
```
★ポイント：

* `UINTERFACE(BlueprintType, Blueprintable)` なclass
* `UFUNCTION(BlueprintNativeEvent, BlueprintCallable)` な関数

C++で使用/実装してBPでも使用/overrideするということから属性がたくさん付与されます。

`BlueprintType`にして変数として保持できるようにします。

`Blueprintable`にしてインターフェースを実装できるようにします。

`BlueprintNativeEvent`はデフォルト実装を持たせつつもBPでも実装できるようにするために必要です。インタラクション処理はC++実装とBP実装の両方が必要になりがちなのでこれを採用します。

`BlueprintCallable`はBP側の任意のタイミングで関数を呼び出したいことがあるためつけておきます。

ややこしいですが、`BlueprintNativeEvent`はvirtualを付けません。

## BP Native UInterfaceのC++での使用
`BlueprintImplementableEvent`と同様に、`Execute_`付の`Thunk`関数を使用する必要があります。

```cpp: 使用例
void APlayerCharacter::TryInteract(AActor* Other)
{
    if(Other->Implements<UInteractable>())
    {
        if(IInteractable::Execute_CanInteract(Other, this))
        {
            // ここにInteractor側のインタラクト処理を実装
            Interactor_DoInteract();

            // Interacable側のインタラクトされた時の処理呼び出し
            IInteractable::Execute_OnInteract(Other, this);
        }
    }
}
```
## BP Native UInterfaceのBPでの使用
普通の関数ノードつなぐだけ

## BP Native UInterfaceのC++での保持
上記と同様

## BP Native UInterfaceのBPでの保持
BP変数にもつだけ

## BP Native UInterfaceのC++実装
C++での実装はデフォルト実装となります。自分でデフォルト実装を実装することが可能です。
実装するには`_Implementation`付きの同名関数を実装します。`interface`型にデフォルト実装するには、`virtual`修飾つきで実装するとよいでしょう。C++派生型でさらに`override`できた方が都合がよいからです。

```cpp: .h
class IInteractable
{
    ... 略 ...

    // interfaceメソッドの定義
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interactable")
    bool CanInteract(AActor* Interactor) const;

    // デフォルト実装の定義
    virtual bool CanInteract_Implementation(AActor* Interactor) const;
}
```
```cpp: .cpp
bool IInteractable::CanInteract_Implementation(AActor* Interactor) const
{
    //ここにC++デフォルト実装を書く
    return IsValid(Interactor);
}
```

:::message
明示的にデフォルト実装を行わない場合はUHTによりデフォルト実装が自動生成されます。
戻り値は型に応じて`false`, `0`, `nullptr`あたりが採用されます。ユーザー定義型の場合 デフォルトコンストラクタが採用されるようです。
`FVector`のような型は`FVector(ForceInit)`が `return` されていました。助かる。
:::

----

さて、次は具象型にC++実装してみましょう。宝箱を例として挙げます。全部実装するとわかりにくいので一部のみ実装して説明します。

```cpp: TreasureBox.h

/**C++実装による宝箱アクターの基底クラス
 * BP側でいろんな宝箱を実装する際はこのクラスから派生すること*/
UCLASS()
class ATreasureBoxBase : public AActor, public IInteractable
{
    virtual bool CanInteract_Implementation(AActor* Interactor) const override;
}

/** 鍵付き宝箱アクターの基底クラス
 * BP側で鍵付き宝箱を実装する際はこのクラスから派生すること
 * デコレータパターン */
UCLASS()
class ALockedTreasureBox : public ATreasureBoxBase
{
    virtual bool CanInteract_Implementation(AActor* Interactor) const override;
};

```

```cpp: TreasureBox.cpp
bool ATreasureBoxBase::CanInteract_Implementation(AActor* Interactor) const
{
    // デフォルト実装 (相手が有効である)
    if(!Super::CanInteract_Implementation(Interactor)){return false;}

    if(!IsValid(this)) { return false; } // 宝箱自身が有効である
    if(宝箱がすでに空いている) {return false; }

    // 事例1: 鍵なし宝箱
    // 相手がIInteractorならばOK!
    // ACharacterやBP_Playerが実装しているはず
    if(Interactor->Implements<UInteractor>())
    {
        return true;
    }

    return false;
}

bool ALockedTreasureBox::CanInteract_Implementation(AActor* Interactor) const
{
    if(!Super::CanInteract_Implementation(Interactor)){ return false; }

    // 事例２： 鍵付き宝箱
    // この宝箱の指定のカギを所有していたらOK！
    if(IInventoryOwner* Inventory = Cast<IInventoryOwner>(Interactor))
    {
        return Inventory->HasItem(TEXT("Item.TreasureBoxKey.00"));
    }
}
```
こんな感じです。
デフォルト実装では共通実装して、派生宝箱クラスではちゃんとロジックを実装しています。
いわゆるデコレーターパターンがささった事例です。宝箱の条件なんかは試行錯誤の余地がない決まり切った仕様でしょうからC++で書いても問題ないでしょう。BP側でノードで書くと結構ながーくなっちゃいますし。

## BP Native UInterfaceのBP実装
上記のようにC++側で基盤実装されたうえで、さらにBP側でなんやかやoverride実装したいとします。


`BP_TreasureBox_Animatable` は豪華アニメ付き宝箱とします。この宝箱に対して `CanInteract`を実装してみましょう。登場演出アニメーション中は開けられないこととします。
宝箱ごときのアニメ状態は超単純であり、いちいちC++でもつとだるいのでBP変数で持つことにましょう。

他にも開封アニメやＳＥを再生したいため、`OnInteract`も実装するのが良さげです。

`override`のやり方は`BP Implementable UInterface`と同じです。実際の実装は主旨に外れるので割愛します。本事例を通して、C++で実装しつつBPでも実装したいという具体的な例が分かったかと思います。


# 詳解

ようやく本題です。

# 仮想デストラクタがなくても大丈夫
仮想関数を持ちし`class/struct`は仮想デストラクタが必須です。
しかしながら、本稿のサンプルコードでは一切仮想デストラクタを記述していません。

大丈夫なんでしょうか？大丈夫です。`UClass/UInterface`においては 仮想デストラクタは記述不要です。 `GENERATED_BODY()`によって`user-provided`な仮想デストラクタがちゃんと自動実装されるからです。別に自前実装しても問題ありません。自前実装したときは自動実装されません。しっかりしてますね。

`user-provided`と`user-declared`の違いで困ることはないと思いますが、いちおう言及しておきます。

https://qiita.com/yumetodo/items/424cc4d15de4edad436a


# `BlueprintImplementableEvent` 詳解
`Thunk`関数が分からなすぎるので生成されるコードを覗いてみましょう。
```cpp: 再掲
#include "CoreMinimal.h"

UINTERFACE(BlueprintType, Blueprintable)
class UDamageReaction : public UInterface
{
    GENERATED_BODY()
};

class IDamageReaction
{
    GENERATED_BODY()
public:

    /** BPで実装する関数 */
    UFUNCTION(BlueprintImplementableEvent)
    void PlayReaction(FContext& Context);
};
```

UHTにより自動生成されたコードが`GENERATED_BODY`部分に差し込まれた結果概ね以下のコードとなります。 主旨に不要な部分は省略しています。

```cpp: generated.h
class IDamageReaction
{
protected:
    // 自動生成された仮想デストラクタ
    virtual ~IDamageReaction() {}
public:
    // ブループリントVMへ呼び出す Thunk関数 
    static void Execute_PlayReaction(UObject* O, FContext& Context);

public:

    /** BPで実装する関数 */
    UFUNCTION(BlueprintImplementableEvent)
    void PlayReaction(FContext& Context);
};
```
なるほど。ちゃんと仮想デストラクタが生成されており、継承しても安全そうです。次はgen.cpp側です。

```cpp: gen.cpp
// デフォルト実装を使うとcheckで止まるようになっている
void IDamageReaction::PlayReaction(FContext& Context)
{
    check(0 && "Do not directly call Event functions in Interfaces. Call Execute_PlayReaction instead.");
}

// Thunk関数の実装
// BP側に実装があればそれを呼び出し、なければ呼び出さない
void IDamageReaction::Execute_PlayReaction(UObject* O, FContext& Context)
{
    check(O != NULL);
    check(O->GetClass()->ImplementsInterface(UDamageReaction::StaticClass()));
    UFunction* const Func = O->FindFunction(TEXT("PlayReaction"));
    if (Func)
    {
        DamageReaction_Parms Parms{.Context=Context}; // 引数を包むためのラッパ構造体
        O->ProcessEvent(Func, &Parms); //☆ ここで BP実装を実行
        Context = Params.Context; //BPで書き換えられた情報を書き戻す
    }
    else
    {
        // BP実装がなければ何もしない
        // C++側では実装をもたないやつだから
    }
}
```

謎であった`Execute_PlayReaction`関数が生成されているのが確認できます。
Blueprint上で実装された関数を実行するためのラッパー関数が生成されています。`FindFunction`でBP上で実装されているはずの関数を関数名から探し出して、あれば実行しています。

`UFUNCTION(BlueprintImplementableEvent)`はどうして`virtual`修飾しないのか、どうして`Execute_`付きの関数を呼び出さなくてはならないのかが判明しました。

# `BlueprintNativeEvent` 詳解
同様に `BlueprintNativeEvent`も見ていきましょう。
説明用ミニマム版。
```cpp: 最小API
UINTERFACE(BlueprintType, Blueprintable)
class UInteractable : public UInterface
{
    GENERATED_BODY()
};

class IInteractable
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interactable")
    bool CanInteract(AActor* Interactor) const;
};
```

UHTにより自動生成されたコードが`GENERATED_BDOY`部分に差し込まれた結果概ね以下のコードとなります。 主旨に不要な部分は省略しています。

```cpp: generated.h
class IInteractable
{
public:
    // デフォルト実装が自動生成される
    virtual bool CanInteract_Implementation(AActor* Interactor) const { return false; }; 
protected:
    // 仮想デストラクタが自動生成される
    virtual ~IInteractable() {}

public:
    // Thunk関数が自動生成される
    static bool Execute_CanInteract(const UObject* O, AActor* Interactor); 
public:
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interactable")
    bool CanInteract(AActor* Interactor) const;
};
```

Thunk関数はこんな感じです。
```cpp: gen.cpp
bool IInteractable::Execute_CanInteract(const UObject* O, AActor* Interactor)
{
    check(O != NULL);
    check(O->GetClass()->ImplementsInterface(UInteractable::StaticClass()));
    
    // BP実装があればそれを優先的に呼び出す
    UFunction* const Func = O->FindFunction(TEXT("CanInteract"));
    if (Func)
    {
        Interactable_Parms Parms{.Interactor=Interactor};
        const_cast<UObject*>(O)->ProcessEvent(Func, &Parms);
    }
    // BP実装がなければC++側のnative実装を呼び出す
    // virtual関数なのでC++側でoverrideされていればそれが呼び出される
    else if (auto I = (const IInteractable*)(O->GetNativeInterfaceAddress(UInteractable::StaticClass())))
    {
        Parms.ReturnValue = I->CanInteract_Implementation(Interactor);
    }
    return Parms.ReturnValue;
}
```
`BlueprintNativeEvent`の場合は`Thunk`関数内で、BP実装がなければC++実装をコールするようになっています。`CanInteract_Implementation`は `virtual`関数なので`ALockedTreasureBox`の場合でもちゃんと`ALockedTreasureBox::CanInteract_Implementation`が呼ばれます。

`BlueprintNativeEvent`と`BlueprintImplementableEvent`の違いはC++実装へフォールバックするかしないかであることがわかりました。ドキュメントの通りでした。

# Interface Message vs Call Function

`BlueprintCallable`なくせにBlueprintから全然`TScriptInterface`経由で`call`できなかったので、カッとなって調査しました。結論から言えば、`Interface Message` なる存在を知らなかったが故の勘違いでした。

まずは画像をご覧ください。
![](/images/ue5-interface-research/bp_call_function.png)

前提となるコードはこちらです。
```cpp: 再掲

class IBuffStatusProvider
{
    UFUNCTION(BlueprintCallable)
    virtual bool IsPoison() const = 0;
}

class IBuffStatusProvider2{
public:
    UFUNCTION()
    virtual bool IsPoison2() const = 0;
};

UCLASS(Blueprintable)
class AStatusActor : public AActor,
    public IBuffStatusProvider,
    public IBuffStatusProvider2
{
	GENERATED_BODY()
public:
	UFUNCTION(BlueprintCallable)
	virtual bool IsPoison() const override { return bPoison; } 

	UFUNCTION(BlueprintCallable)
	virtual bool IsPoison2() const override { return bPoison; }
};
```

selfは `AStatusActor`です。
画像を見ると同じ関数なのにTargetが異なるやつがありますね。一体これはなんでしょうか？

* 上段ノードは `Class > IBuffStatusProvider > IsPoison(Message)`
* 中段ノードは `Class > IBuffStatusProvider > IsPoison`
* 下段ノードは `Call Function > IsPosion`

どれを使えばいいんでしょうか？何が異なるのでしょうか？調べました。

## Call Function
`Call Function` とはTarget型`T`の`UFUNCTION`を呼び出すBPノードで、その実体は`UK2Node_CallFunction`ノードです。BPコンパイル時に ノードが指す`UFucntion`への関数呼び出しが内部的に登録されます。いわゆるメンバー関数呼び出しであり、最終的に`T::Func`が呼ばれます。

よって中段と下段の違いは`UFunction`に何が入っているかの違いです。調べたところ、
* 中段ノード: `IBuffStatusProvider::IsPoison`の`UFunction`
* 下段ノード: `AStatusActor::IsPoison`の`UFunction`

が格納されていました。なるほど、つまり`override`したメンバー関数を普通に呼び出すのか`Super`型へキャストして親の関数を直接呼び出しちゃうのか、という違いですね。
この場合`AStatusActor` は`IsPoison`を`override`しているため、下段の`AStatusActor::IsPoison`を呼び出すのが正解っぽそうです。

よって、原則として自身が具象型を知っているなら具象型の関数を呼んだらええ、ということがわかりました。

- `BlueprintCallable` な具象型のAPIは BPから`Call Function`(関数呼び出し)で呼び出す
- `BlueprintCallable` な`interface API` はBPから `Interface Message`経由で呼び出す

しかし待ってください、`IBuffStatusProvider::IsPoison()=0`は純粋関数です。少し怪しいですね、どこか勘違いしているようです。


## UFUNCTIONのおさらい
ここでまず `UFunction`とは何なのかをさらーと触れておきます。
`UFunction`は 関数ポインタを持つ`class`です。他にもいっぱい機能がありますがようは関数オブジェクト、ファンクタみたいなやつとでも理解しておきます。

```cpp: 疑似コード
using FNativeFuncPtr = void(*)(UObject* Context, FFrame& TheStack, void* const RESULT_PARAM);

class UFunction : UStruct
{
    // C++実装されたUFUNCTION()への関数ポインタ
    FNativeFuncPtr Func;
}
```
`UFUNCTION(BlueprintaCallable)`で修飾されたメンバ関数に対してはUHTにより以下のグルーコード（Thunk関数？）が生成されます。
```cpp: 疑似コード
void IBuffStatusProvider::execIsPoison(UObject* Context, FFrame& TheStack, void* const RESULT_PARAM)
{
    // thisポインタ経由でoverrideされた仮想関数を呼び出すイメージ
    RESULT_PARAM.Result = this->IsPosion();
}
```

そして、この `execIsPoison`関数アドレスが名前とともに静的テーブルに保持されますので、`UClass`へ登録します。
```cpp: 疑似コード
static const TTuple<FName, FNativeFuncPtr> Funcs[]
{
    { "IsPoison", &IBuffStatusProvider::execIsPoison},
};

// ここでU付きのクラスが登場したぞ！
FNativeFunctionRegistrar::RegisterFunction(UBuffStateProvider::StaticClass(), Funcs, 1);
```
関数テーブルを保持する`UClass`を作るために`U`付のクラスを実装してたんですねー。

ここまで来たらあとは `UFunction::Func`に関数アドレスをセットするだけですね。
```cpp: 疑似コード
UFunction* Func = NewObject<UFunction>(...);
Func->Func = Funcs[0].Pointer; //&IBuffStatusProvider::execIsPoison
```

`IBuffStatusProvider::execIsPoison()`を指す`UFunction`がどこかに保持されました。
同様に、`AStatusActor::execIsPoison()`を指す`UFunction`がどこかに保持されました。

`UFunction`とは様々なシグネチャの関数をグルーコードでラップすることにより統一的な関数ポインタにまとめて保持する関数オブジェクト、と言えることが分かりました。

## Call Function は Call UFunction
CallFunctionノードとは 関数名からUFunctionを探しそれを実行するノードでした。

> よって中段と下段の違いは`UFunction`に何が入っているかの違いです。調べたところ、
> * 中段ノード: `IBuffStatusProvider::IsPoison`の`UFunction`
> * 下段ノード: `AStatusActor::IsPoison`の`UFunction`

先ほどの説明をより正確に書き下しましょう。

* 中段ノード: `IBuffStatusProvider::execIsPoison`を指す`UFunction`
* 下段ノード: `AStatusActor::execIsPoson`を指す`UFunction`

実際にbreakポイントを張って`UFunction::Func`のアドレスを見ると, `Hogehoge.dll!IBuffStatusProvider::execIsPoison(UObject..略..)`という関数アドレスが格納されていました。

`this`ポインタ経由で `IsPoison()`が呼び出されるならば `vtable`上の正しい実装へのアドレスが入っているはずなので純粋関数でも大丈夫そうです。納得。

## Interface Message
先述の通り、`Interface Message` とはBPノードの右上にお手紙マークがついているノードです。（お手紙マークはmessageという意味だったのか......)
このノードは`Call Function`とほぼ同じですが、インターフェースを実装しているかチェックします。

`interface Message` の直接の実体は `UK2Node_Message`です。BPコンパイル時に内部的に`CastToInterface`ノードおよび`FunctionCall`ノードに変換されます。ノードのTargetが`interface`を実装していなければデフォルト値を返します。
`Thunk`関数と似たような処理を`BlueprintVM`上で行っているという訳です。

ということはわざわざ自前で`Does Object Implement Interface`ノードを使わずとも、呼び出すだけなら `Message`ノード使っても良さそうですね。

# まとめ

* C++のみ → 純粋仮想関数でOK. `CannotImplementInterfaceInBlueprint` 
* BPからメッセージ呼び出し -> `BlueprintCallable`, `BlueprintType`, `CannotImplementInterfaceInBlueprint`
* BPから関数を呼び出す -> `BlueprintCallable`, `BlueprintType`, `Blueprintable`
* BPでoverride -> `BlueprintImplementableEvent`と `Execute_`呼び出し
* BPで実装 -> `BlueprintNativeEvent`と `Execute_`呼び出し
* 全部入り -> `BlueprintType`と`Blueprintable`, `BlueprintCallable`と`BlueprintNativeEvent`と `Execute_`呼び出し

共通ルール
* `BlueprintImplemetableEvent` は `virtual`つけない
* `BlueprintNativeEvent` は `_Implementation`を`virtual`で実装する
* `BlueprintImplemetableEvent` と `BlueprintNativeEvent`は `Execute_`を使って呼びだす

見えないThunk関数に気を付ける。間違った使い方をすると`check`で止まる。`DO_CHECK`マクロを外していると止まらず気が付けない。

`UInterface` とは`interface`のように振る舞う型であり`interface`そのものではない。C++の仮想関数を適切に呼び出せるように拡張しつつ、`UHT`と`UObject`の仕組みを利用してリフレクション・GC・BP連携などをサポートしたインターフェース機構である。

# 付録： UInterface 設計方針
個人的に`interface`はデフォルト実装やデータを持つべきではないと考えております。データを持つならBaseクラスでいいじゃん、と思っているからです。ただし、そうせざるを得ないときもあります。`interface`とはこうあるべき論は本稿の主旨ではないので言及しません。

:::message
`interface`はデータフィールドを持たないほうがいいでしょう。なぜならば実装側のメモリレイアウトが変更されてしまうからです。特にUEではとあるフィールドを派生型で`UPROPERTY`にするしないを変更することができません。また`UPROPERTY`の属性やメタ情報も変更することができません。HPという値を`UPROPERTY(Replicated)`にするべきか、`UPROPERTY(ReplicatedUsing=OnRep_Hp)`なのかは実装者側にしかわからないでしょう。
ひとたびデータを持つと、エディタでのオーサリングや`SaveGame`などinterfaceが負うべき責任がどんどん増えてしまいます。
:::

# ぽえむ

本稿を読んだことで`UInterface`を使いたくなるかもしれませんが、なんでもかんでも`UInterface`にすればいいというものではありません。細やかにコンポーネント指向になっているならば`interface`を使わずともいいケースもあるはずです。

`interface`がクラス間を疎結合にしているというのはある種正しいのですが、より正確な実態は`interface`に依存を集約しているだけです。`interface`による「依存性の逆転」ではなく`interface`への「依存性の抽出」という表現がより正確なところではないでしょうか？
広く依存される`interface`は依存性が抽出されまくった濃厚でドッピオなコーヒーのようなものなので、ひとたび名前やAPIを変更しようとすると大変な苦労に見舞われます。事実上不可能なため`IInteractable2`や `IMyObjectEx`みたいな次期`interface`が生えたりします。

`interface`に高度に抽象化されているが故に、この文脈でこのポインタにはどの具象クラスが入っているからわからん、デバッグが大変だーということもあります。コードを読む側も実装がどこにあるか探すのが大変なことがあります。

シグネチャを統一したいだけならば`concept`や`template`関数を使えば代替できたりします。`interface`はツール群の一つとして捉えてください。`UInterface`を使うときは具体的に解決したい課題を見据えて導入するのが良さそうだと思いました。


# 参考資料

* [Unreal Interfaces](https://dev.epicgames.com/documentation/en-us/unreal-engine/interfaces-in-unreal-engine)
* [UE4 の C++ プログラミング入門](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/introduction-to-cplusplus-programming-in-ue4?application_version=4.27)
* [[UE4/UE5] Interfaces (BP + C++)](https://kitatusandfriends.co.uk/ue4/ue5/interfaces/)
* [“Interface Call” (vs Interface Message/Call Function)](https://forums.unrealengine.com/t/interface-call-vs-interface-message-call-function/344723)
* [UE4 インターフェイスについてのメモ](https://qiita.com/unknown_ds/items/bbe94534048ac50fb4b9)
* [UAbilitySystemInterface marked with CannotImplementInterfaceInBlueprint](https://forums.unrealengine.com/t/uabilitysysteminterface-marked-with-cannotimplementinterfaceinblueprint/2522628/1)