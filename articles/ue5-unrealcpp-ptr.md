---
title: "UE5:Unreal Engineのポインタについてまとめた 前編"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: true
---

# はじめに
本稿では Unreal Engine 5のポインタについて一覧しまとめます。
どのようなポインタがあり、それぞれどう使うのか、そしてpure C++のポインタとどう違うのかについて言及します。正しい扱い方について述べますので、アクセスバイオレーションを起こさないように気を付けましょう。

あまりに内容が膨大なので、三編に分けました。
本稿は前編です。

* [中編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr-second)
* 後編 - 工事中

# 用語説明

## 用語定義
説明のために以下の構造体とUCLASSを定義します。
C++においてはstructとclassにほとんど違いはないため基本的にstructで説明します。
```cpp
namespace MyPlugin
{
    // pure C++な構造体
    struct FCppStruct
    {
        int Value = 0;
    }
}

// Unreal C++なUSTRUCT
USTRUCT()
struct FStruct
{
    GENERATED_BODY();

    int Value = 0;
}

// Unreal C++なUCLASS
UCLASS()
class UMyClass : public UObject
{
    GENERATED_BODY();

public:
    UPROPERTY()
    int Value = 0;
}

```

## ポインタ一覧
Unreal Engineには以下のポインタがあります。

### アンマネージドポインタ
UnrealEngine によって管理されないメモリ領域を指すポインタです。
ガベージコレクションの対象ではありません。
使い方も機能もpure C++に該当するそれらとほぼ同じです。


| アンマネージド | 名前 | 補足 |
| ---- | ---- | ---- |
| `FCppStruct* Object;` | 生ポインタ | 使うべきでない |
| `TUniquePtr<FCppStruct> Pointer;`| ユニークポインタ | `std::unique_ptr`に該当 |
| `TSharedPtr<FCppStruct> Pointer;` | シェアードポインタ | `std::shared_ptr`に該当 |
| `TWeakPtr<FCppStruct> Pointer;` | ウィークポインタ | `std::weak_ptr`に該当 |

:::message
上記のアンマネージドポインタは UObject を参照してはいけません。`TSharedPtr<UObject>` はダメです。`FHoge` のように `F`で始まるクラス・構造体のみ型パラメータに入れると覚えてください。
:::

### マネージドポインタ
UnrealEngine によって管理されるメモリ領域を指すポインタです。
ガベージコレクションの対象です。

| マネージド | 名前 | 補足 |
| ---- | ---- | ---- |
| `UPROPERTY() Ubject* Pointer{};` | ハードオブジェクトポインタ | 古い書き方で今どき使わない。`TObjectPtr`使え |
| `UPROPERTY() TObjectPtr<UObject> Pointer;` | オブジェクトポインタ | ハードオブジェクトポインタの進化版。基本はこれ。 |
| `UPROPERTY() TSoftObjectPtr<UObject> Pointer;` | ソフトオブジェクトポインタ | UE5独自のやわらかい参照(後述) |
| `UPROPERTY() TWeakObjectPtr<UObject> Pointer;` | ウィークブジェクトポインタ | `TObjectPtr`の所有権を持たない版 |
| `UPROPERTY() TStrongObjectPtr<UObject> Pointer;` | ストロングブジェクトポインタ | `TObjectPtr`の所有権を持つ版 |
| `Ubject* Pointer{};` | ワイルドポインタ | UPROPERTY()としてリフレクションシステムに辿られない野生のポインタ。ダングリングポインタになる。バグなので直せ。`TObjectPtr`にしろ |


:::message
UEのマネージドポインタには強弱と硬軟の二軸あることを気に留めておいてください。
:::

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

## Raw Pointer (生ポ)
```cpp
FStruct* Pointer = nullptr;
UObject* Object = nullptr;
```
生ポ尽く死すべし！

pure C++と全く同じです。言及することはありません。
今時のC++では使いませんし、UnrealEngineでも使いません。生ポはとにかくダングリングポインタになりやすいので、スマートポインタを使いましょう。

関数の戻り値として生ポを返すことはあります。BP関数のように、ゲームスレッドで使われることが前提とされている関数では生ポでも良いのです。

:::message alert
生ポインタに`IsValid` を使ってはいけません。 `IsValid(UObject*)`は UPROPERTY()なハードオブジェクトポインタに対して使われるよう設計されております。
生ポに対して`IsValid` を使った場合、破棄済み(`IsPendingKill`)なオブジェクトを触る危険や、GCによる回収→NewObjectで再利用された生きている別オブジェクトを触る危険があります。
:::

## TUniquePtr
```cpp
 TUniquePtr<FCppStruct> Pointer;
```
単一の所有権を持つポインタです。`std::unique_ptr`と同じです。
`MakeUnique`で作り`MoveTemp`によって転送することができます。
ラムダキャプチャする場合は、Moveキャプチャが必要です。

```cpp
{
    TUniquePtr<FCppStruct> Ptr0 = MakeUnique<FCppStruct>(); // New FCppStructする
    TUniquePtr<FVector> Vector = MakeUnique<FVector>(1, 2, 3); //New FVector(1,2,3)する

    TUniquePtr<FVector> Ptr1 = MoveTemp(Ptr0); //Ptr0をPtr1に移動する
    
    check(!Ptr0.IsValid()); // Ptr0の所有権は手放し済み
    check(Ptr1.IsValid()); // Ptr1は所有権を譲り受けた

    if(Vector) // null checkは operator bool でok.
    {
        int Sum = Vector->X + Vector->Y + Vector->Z;// 生ポと同じ感じで触れる
    } 
    Ptr1.Reset(); // 明示的にdelete される

    //ラムダキャプチャはmoveキャプチャすること
    auto Lamda = [Vec = MoveTemp(Vector)]()
    {
        check(*Vec == FVector(1,2,3));
    };

} // ここでデストラクタが呼ばれてオブジェクトはdeleteされる

```
## TSharedPtr
参照を共有する共有ポインタです。`std::shared_ptr`と同じです。
参照カウントによって管理されています。そのため、循環参照には気を付けないといけません。
`MakeShared`で作り、コピーで参照を増やせます。

```cpp
TSharedPtr<FCppStruct> SharedPtr0 = MakeShared<FCppStruct>(); // New FCppStructする
TSharedPtr<FCppStruct> SharedPtr1 = SharedPtr0; // 参照カウント増やす

check(SharedPtr0.GetSharedReferenceCount() == 2);
```

参照カウントを増やしたくない場合は `ToWeakPtr`で WeakPtrに変換できます。

```cpp
TWeakPtr<int> WeakPtr = SharedPtr0.ToWeakPtr();
```

### TSharedRefという共有参照
UEには`TSharedRef`という型が存在します。これは非nullな共有参照型です。pure c++にはありません。
非null保障が出来る状態であるならば、`TSharedRef`を引き回すとnullチェックを省略できます。
```cpp
TSharedRef<int> SharedRef0 = MakeShared<int>(100); // MakeSharedはTSharedRefを返す
TSharedPtr<int> SharedPtr = SharedRef0.ToSharedPtr(); //相互変換可能
if(SharedPtr.IsValid())
{
    TSharedRef<int> SharedRef1 = SharedPtr.ToSharedRef();
}

```

### TSharedPtrのスレッドセーフ性は選択可能
`std::shared_ptr`の参照カウントはスレッドセーフです。逆にスレッドセーフじゃなくすることはできません。単一のスレッドでしか使わないような状況の場合はこのオーバーヘッドを削りたくなります。一方、`TSharedPtr`ではスレッドセーフ性を型パラメータによって選択することができます。
デフォルトでは ~~非スレッドセーフです。~~ よく見たらスレッドセーフでした。
```cpp
TSharedPtr<int, ESPMode::NotThreadSafe> SharedPtr = MakeShared<int, ESPMode::NotThreadSafe>(123);
TSharedPtr<int, ESPMode::ThreadSafe> SharedPtr = MakeShared<int, ESPMode::ThreadSafe>(123);
TSharedPtr<int> SharedPtr = MakeShared<int>(123); // 指定しないときはスレッドセーフ

```

:::message
参照カウントがスレッドセーフなだけで共有している`T`型のオブジェクトはスレッドセーフではありません。`SharedPtr->Value++;`のようにマルチスレッドでRead/Writeが発生するときはちゃんと排他制御しましょう。
:::

### TSharedFromThis
`std::enable_shared_from_this` に該当する型です。
thisポインタから`TSharedPtr`を作成したい場合は、`TSharedFromThis`を継承します。
使いどころとしては木構造や親子構造を作るときに自身のweakポインタを渡したいときや、ラムダキャプチャしたいときです。

`AsShared`で`this`への`SharedPtr`を作れます。
以下はラムダ式に`this`を渡すサンプルです。
```cpp
struct FStruct : public TSharedFromThis<FStruct>
{
    void DoAsync()
    {
        // AsShared() によりTSharedRef<FStruct> ThisPtr としてコピーすることで
        // thisへの参照カウントを増やす
        AsyncTask(ENamedThreads::Type::AnyThread, [ThisPtr = AsShared()]()
        {
            // TSharedRefは非nullであるのでnullチェックは不要
            ThisPtr->DoSomthing();
        });
    }
    void DoSomthing(){}
}

void Main()
{
    TSharedPtr<FStruct> Instance = MakeShared<FStruct>();
    Instance->DoAsync();
    Instance.Reset(); // ここで参照カウントを減らしてもラムダ式はちゃんと実行される
}

```

`AsWeak()`で`this`へのWeakPtrを作れます。
親となるファクトリーやマネージャーが子オブジェクトへ自身への参照を渡すときに便利です。循環参照を避けられます。
```cpp
struct FMyNode : public TSharedFromThis<FMyNode>
{
    explicit FMyNode(TWeakPtr<FMyNode> InParent)
		:Parent(InParent)
	{
	}

	TSharedPtr<FMyNode> CreateChild()
	{
		TWeakPtr<FMyNode> ThisPTr = this->AsWeak();
		TSharedPtr<FMyNode> Child = MakeShared<FMyNode>(ThisPTr);
		return Child;
	}

private:
    TWeakPtr<FMyNode> Parent;
    TSharedPtr<FMyNode> Left;
    TSharedPtr<FMyNode> Right;
}

void Main()
{
    TSharedPtr<FMyNode> Root = MakeShared<FMyNode>(nullptr); //ルートの親はnullptr
    TSharedPtr<FMyNode> 子 = Root->CreateChild();
    TSharedPtr<FMyNode> 孫 = 子->CreateChild();
}
```

:::message alert
すでにご存じかとはおもいますが、MakeShared(this)はthisに対して二重にdeleteが呼ばれてしまいバグとなります。
ちゃんと`TSharedFromThis::AsShared`を使いましょう。
https://cpprefjp.github.io/reference/memory/enable_shared_from_this.html
:::

### AsSharedで派生型を返したい場合
`TSharedFromThis`を継承したクラスから更に派生したとき、`AsShared`で返るのはベースクラスの型です。少し困るのでそういうときは `SharedThis(this)`を使います。

```cpp
struct FMyBase : public TSharedFromThis<FMyBase>
{
    virtual ~FMyBase() = default;
}

struct FMyDerived : public FMyBase
{
    virtual ~FMyDerived() override = default

    void DoFunc()
    {
        TSharedPtr<FMyBase> SharedThisPtr = this->AsShared(); // FMyBase型で返ってきて困る
        TSharedPtr<FMyDerived> SharedDerivedThisPtr= this->SharedThis(this); // FMyDerived型で貰える
    }
}
```

`SharedThis(this)` は後述の `StaticCastSharedPtr`を使って、`TSharedPtr<FMyBase>`を`TSharedPtr<FMyDerived>`にキャストしているだけです。

### TSharedPtrのキャスト
:::message 
shared_ptrで管理されるポインタ型をキャストするには専用のcastが必要です。
:::

`std::static_poiter_cast`に該当する機能は`StaticCastSharedPtr()`です。
専らダウンキャストに使いますが型が判明しているならアップキャストにも使えます。使いどころとしては`interface`型にキャストするときでしょう。

```cpp
class ISoundService
{
    virtual int PlaySE(int SeNumber) = 0;
}

class FSoundServiceImpl : public ISoundService
{
    virtual int PlaySE(int SeNumber) override
    {
        // 実装省略
         return 0;
    }
}
class FMyServiceResolver
{
    // 実装を隠蔽できてinterfaceのみ公開出来ていい感じ
    static TSharedPtr<ISoundService> CreateDefaultSoundService()
    {
        TSharedPtr<FSoundServiceImpl> Impl = MakeShared<FSoundServiceImpl>();
        TSharedPtr<ISoundService> Service = StaticCastSharedPtr<ISoundService>(Impl)
        return Service;
    }
}
void Main()
{
    TSharedPtr<ISoundService> SoundService = FMyServiceResolver::CreateDefaultSoundService();
    SoundService->PlaySE(123);
}
```
`TSharedPtr`に格納する必要はあるのかと問われると疑問だし、`UWorldSubSystem`使えばいいじゃんと言われたそうなのですが、サンプルなので参考にとどめてください。
`Slate`周りなど、Unreal Editorに関わる箇所は `TSharedPtr`が使われるので有効かと思います。

:::message alert
`StaticCastSharedPtr`で間違えたインスタンスを指定した場合、`nullptr`にはならず成功してしまいます。アクセスすると未定義動作となるので間違えないようにしましょう。
:::

## TWeakPtr
所有権を持たない弱いポインタです。`std::weak_ptr`と同じです。`TSharedPtr`と異なり参照カウントを増やさないため、オブジェクトの破棄を妨げることがありません。

```cpp
TWeakPtr<FStruct> Poitner;
```

`TWeakPtr`を使う際は必ず`Pin`でピン止めして使用中に破棄されないようにしなければなりません。
```cpp
if(TSharedPtr<FStruct> Ptr = WeakPointer.Pin())
{
    // このPtrが生きているスコープ内において、(*Ptr)は有効
    // なぜなら参照カウントを1増やしているから
    int Val = Ptr->Value;
}
```

これは WeakPtrの中身を触っている間に、参照カウントが0になりdeleteされる危険があるからです。マルチスレッドでアクセスする場合は操作中にdeleteされないよう、明示的に参照カウントを増やしておきます。自身が使い終わったら`Pin`で止めた参照を速やかに解放するべく、`if`スコープにするのが王道です。

### `TWeakPtr::Get` は使わない
`TWeakPtr`の`Get`は`Pin`で留められたスコープ内において、素早くデリファレンスするためのメソッドです。とはいえ、Pinの返り値を使えばいい話なのでほとんど使いどころはありません。
```cpp
if(TSharedPtr<FStruct> _ = WeakPointer.Pin())
{
    // Pin止めしたスコープ内ではオブジェクトは破棄されないので
    // Get()で直接デリファレンスしてもいい
    // してもいいけどこんな書き方する意味ある？TSharedPtr使いなよ
    int Val = WeakPointer.Get()->Value;
}
```


`null`かどうかが大事な場面であり、中身に興味がない場合は`IsValid`か`operator bool`を使います。
```cpp
if(WeakPointer)
{
    UE_LOG(LogTemp, Dispaly, TEXT("ヌルじゃないことに意味があり中身にアクセスしないのでPinは不要"));
}
else
{
    UE_LOG(LogTemp, Dispaly, TEXT("オブジェクトが破棄されているかnullptrです"));
}
```

### TWeakPtr使用上の注意
`TMap`, `TSet`のキーとして`TWeakPtr`を使ってはいけません。所有権を持たないが故に、いついかなるときでも破棄される恐れがあるためです。
```cpp
// TWeakPtrはValue型なら使ってもいい
TMap<int, TWeakPtr<FStruct>> WeakLookUpTable;
```

# つづく
予想より書くことが多かったので[中編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr-second)に続きます。

# Reference

* [Overview of UObject and "Smart" pointer types in Unreal](https://unrealcommunity.wiki/pointer-types-m33pysxg)
* [Unreal スマート ポインタ ライブラリ](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/smart-pointers-in-unreal-engine)