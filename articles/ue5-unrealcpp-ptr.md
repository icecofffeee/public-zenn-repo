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

* [UE5:Unreal Engineのポインタについてまとめた 前編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr)
* [UE5:Unreal Engineのポインタについてまとめた 中編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr-second)
* [UE5:Unreal Engineのポインタについてまとめた 後編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr-third)

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
参照を共有する共有ポインタです。`std::shared_ptr`と大体同じです。
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

`TSharedRef`と`TSharedPtr`は相互に暗黙的に変換できます。無駄なコストなので引数や返り値型に取るときは統一した方がよさげです。
ただし、`nullptr`になりうる文脈では `TSharedPtr`を、必ず非`nullptr`を期待したい文脈では`TSharedRef`を、要求するという設計もありです。
例えばファクトリー関数です。

```cpp
class IMyObject{};
class FMyFactory
{
    TSharedRef<IMyObject> MakeDefaultInstance(); //必ず成功するので 非nullptr

    TSharedPtr<IMyObject> TryMakeInstance(); //失敗したらnullptr
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

[Unreal スマート ポインタ ライブラリ](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/smart-pointers-in-unreal-engine) の説明では、
デフォルトではスレッドセーフじゃない、という記載がありますが、少なくともUE5.5時点では事実上デフォルトは `ESPMode::ThreadSafe`です。

### TSharedPtrの 前方宣言でESPMode::ThreadSafeになってる

単に`TSharedPtr`と述べていますが、本名は `TSharedPtr<T, ESPMode>`という型です。そのため、必ず型パラメータに`ESPMode`を設定しないとコンパイルエラーになります。
暗黙的に推論されたとしても`ESPMode::NoThreadSafe=0`が採用されるはずなのですが、普通に`TSharedPtr<T>`が使えています。なぜかと思って調べました。

原因は`Templates/SharedPointerFwd.h`によってテンプレート部分特殊化がなされているからです。

```cpp: Templates/SharedPointerFwd.h
template< class ObjectType, ESPMode Mode = ESPMode::ThreadSafe > class TSharedRef;
template< class ObjectType, ESPMode Mode = ESPMode::ThreadSafe > class TSharedPtr;
template< class ObjectType, ESPMode Mode = ESPMode::ThreadSafe > class TWeakPtr;
template< class ObjectType, ESPMode Mode = ESPMode::ThreadSafe > class TSharedFromThis;
```

スレッドセーフ機能がONになるようになっています。
よって次のクラスは `ESPMode::ThreadSafe`としてtemplate解決されます。
```cpp
TSharedRef<int> IntRef = ...; // TSharedRef<int, ESPMode::ThreadSafe>と同じ
TSharedPtr<int> IntPtr = ...; // TSharedPtr<int, ESPMode::ThreadSafe>と同じ
```
ヘッダによるテンプレート特殊化ということはそのヘッダファイルをインクルードしなかったら解釈されなくなっちゃうのですが、安心してください。
`Tempaltes/SharedPointer.h` > `Templates/SharedPointerInternal.h` > `Templates/SharedPointerFwd.h` という流れでインクルードされているので、`TSharedPtr`を知っているクラスなら必ず上記テンプレート特殊化は有効です。

という訳で指定を省いた場合は`ESPMode::ThreadSafe`です。

### `MakeShared`は デフォルトで`ESPMode::ThreadSafe`
`MakeShared`を用いた場合も`ESPMode::ThreadSafe`です。これはテンプレート関数なのですが、型パラメータにデフォルトパラメータが指定されています。
```cpp
template <typename InObjectType, ESPMode InMode = ESPMode::ThreadSafe, typename... InArgTypes>
[[nodiscard]] FORCEINLINE TSharedRef<InObjectType, InMode> MakeShared(InArgTypes&&... Args){...}

```

よって、`MakeShared<FVector>`等は`MakeShared<FVector, ESPMode::ThreadSafe>`に推論されます。
```cpp
auto Ptr = MakeShared<int>(123); // TSharedRef<int, ESPMode::ThreadSafe>に型推論される
TSharedPtr<int> Ptr = MakeShared<int>(123); // TSharedRef<int, ESPMode::ThreadSafe>から暗黙的にTShaedPtr<int, ESPMode::ThreadSafe>に変換される
```

### MakeShared vs MakeSharable vs コンストラクタ

何が違うの、どれをいつ使えばいいのか悩ましいので調べました。
結論としては、ほとんどのユースケースのおいて `MakeShared<T>`を使ってOKです。


```cpp
TSharedRef<T> Ref1 = MakeShared<T>();
TSharedRef<T> Ref2 = MakeShareable(new T()); //通常はonelinerで new した方がいい
TSharedRef<T> Ref3(); //デフォルトコンストラクタ. 内部で new T()が呼ばれて自動でdeleteされないので使っちゃいけない
```
使用感はどれも似た感じなのですが、微妙に挙動が違います。

1. `MakeShared<T>` - 任意のコンストラクタを呼べる. ヒープが分かれないのでキャッシュ効率がいい
1. `MakeShareable<T>` - 非侵襲型なのでweak参照と仲良し
1. コンストラクタ - `Please do not use!` とコメントがあります


`MakeSharable`は特殊な限られた状況で使用します。

1. 型`T`のコンストラクタが呼べず、ファクトリー関数しか公開されていない
1. やんごとなき事情により、外部から貰ったインスタンスを共有管理したい
1. 非侵襲型の共有参照をどうしても使いたい
1. 型`T`のサイズがめちゃくちゃ大きい、かつ`WeakPtr`の寿命がめっちゃ長い

こういうときは `MakeShareable`もありかなって思います。

### `MakeShared` で作るSharedPtrは 侵襲型です

先に侵襲型と非侵襲型の説明をします。
侵襲型は 型T とコントロールブロックを同じ型に格納するという意味です。
非侵襲型は 型T とコントロールブロックを別々のインスタンスでnewしてポインタで持つという意味です。
もともとの型Tを破壊しなくてもいいという意味で非侵襲です。


```cpp: 疑似コード
// 侵襲型
template<class T>
class FIntrusiveReferenceCntroller
{
    int32 SharedCounter; //コントロールブロック
    int32 WeakCounter;　　//コントロールブロック
    uint8 Data[sizeof(T)]; // sizeof(T)のメモリをthisに持っている

    T* Get() { return reinterpret_cast<T*>(Data); }
}

//非侵襲
template<class T>
class FNonIntrusiveReferenceController
{
    int32 SharedCounter;　//コントロールブロック
    int32 WeakCounter;　//コントロールブロック
    T* Object;
}
```

侵襲型の場合は同じオブジェクトにTの実体とコントロールブロックがいます。メモリ配置は連続しています。そのため、デリファレンスするときにキャッシュ効率がいいのです。Objectがすぐそばにいるので。

非侵襲型の場合はポインタなので Objectの先は`new`で確保したヒープのどこか別の場所を指しています。概念的に`this->Object->`とデリファレンスを2回行う必要があります。
実行時効率上は非侵襲型の方が悪くなります。メモリの解放という点では有利です。(後述)

侵襲型は`WeakPtr`が生きている限り、メモリを解放できません。

弱参照なのにどういうことなんです？厳密に解説すると、`SharedCounter == 0`になった時に **デストラクタを呼び出し**、`WeakCounter == 0`になったときに `delete this`を実行します。
これは`WeakPtr`が `FIntrusiveReferenceCntroller::WeakCounter`にアクセスしにくるからです。
`WeakPtr`が活きている限り、`FIntrusiveReferenceController`のインスタンスを`delete`するわけにはいかないのです。

一方、非侵襲型はオブジェクトが分かれていることから、コントロールブロックを残しつつ、`T`型を指す部分のメモリ解放ができます。つまり、`SharedCounter == 0`になった時に `delete Object;`します。メモリは`sizeof(T)`の分だけ解放されます。そして`WeakCounter == 0`になったときに `delete this`を実行します。メモリは`sizeof(int) + sizeof(int) + sizeof(T*)`で 16byte解放されます。2回deleteが走るんですね。

という訳で、`MakeShared<T>`で作ったポインタは侵襲型であるからして、長寿命のWeakPtrと相性が悪いです。

この点を考慮した上でやはり、ほとんどのケースにおいて`MakeShared<T>`で問題ないでしょう。
メモリ消費量が問題になるとしてもそんなに大きな型`T`も滅多にないでしょう。たかだか数百byteならば解放が遅れてもいいじゃん、って思います。
ラムダキャプチャしたラムダ式がずっと残り続けるような、そんな特殊な状況なら `MakeShareable`を使うと改善した気持ちになれるかもしれません。

現実的には`MakeShareable`へと対処しなければならないことはあるでしょうが、早すぎる最適化の類であり後回しでいいと思います。スマートポインタよりもアルゴリズムや仕様を見直した方が桁違いに効果を発揮するのですから。

### 参考

* [Unreal スマート ポインタ ライブラリ](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/smart-pointers-in-unreal-engine)
* [共有のポインタ](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/shared-pointers-in-unreal-engine)

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