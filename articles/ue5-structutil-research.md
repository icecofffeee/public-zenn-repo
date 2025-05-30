---
title: "UE5:Unreal EngineのStructUtilについてまとめた 前編"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: true
---

# はじめに
`StructUtil`プラグインが UE5.5においてエンジンプラグインへと格上げされました。
本稿ではその使い方と活用事例についてまとめます。
主にC++実装サイドについて触れます。

`StructUtil`が晴れてエンジン公式プラグインとして標準化されたことで、サードパーティープラグインやオレオレライブラリから依存ライブラリとして簡単に使用できるようになりました。
`StructUtil`はめちゃくちゃ便利です。

本稿はその前編です。

* [UE5:Unreal EngineのStructUtilについてまとめた 前編](https://zenn.dev/allways/articles/ue5-structutil-research)
* [UE5:Unreal EngineのStructUtilについてまとめた 中編](https://zenn.dev/allways/articles/ue5-structutil-research-second)

# `StructUtil`とは

`StructUtil`とは `USTRUCT*`を`UPROPERTY`として利用できるようにする拡張機能です。

`StructUtil`を使用すれば、`UPROPERTY() FFoo* FooPtr;` を事実上実装可能になります。
`USTRUCT`ではなく`USTRUCT*`です。

# 背景

`UPROPERTY() FFoo* Ptr{};`はコンパイルエラーとなります。
構造体を`UPROPERTY()` にするには、データ実体を定義する必要があります。

```cpp
// Foo.h
USTRUCT()
struct FFoo
{
    GENERATED_BODY();
    UPROPERTY() float FloatVal = 123.0f;
    int IntValue = 42;
}

// UMyObject.h
#include "Foo.h"

UCLASS()
class UMyObject : public UObject
{
    GENERATED_BODY();

    // 実体が必須
    UPROPERTY() FFoo Foo{};

    // コンパイルエラー
    UPROPERTY() FFoo* FooPtr{};
}
```

## 問題点

これの問題は、ポインタ型`FFoo*`でないために派生型を格納することができないことです。`FFoo`を継承して`FFoo2`にカスタム拡張した構造体を差すことが出来ず拡張性に難があります。
次に依存性の問題もあります。`UMyObject`は必ず `FFoo`の[完全型](https://learn.microsoft.com/ja-jp/cpp/c-language/incomplete-types?view=msvc-170)を必要とします。前方宣言`struct FFoo;`ではなく`#include "Foo.h";`が必要となります。`UMyObject`を利用するものもまた`#include "Foo.h";`をすることになり、コンパイル時間が伸びていくこととなります。

## 従来手法

上記問題に対処するため、わざわざ`UObject*`派生型を定義し,`UPROPERTY() TObjectPtr<UFoo> Foo;`にしていました。`UObject`は大きな`class`であるためメモリの無駄が大きい点が玉に瑕です。

他の回避手法として、

* `void*` と 型情報
* `union`
* `TVariant`

が考えられますが、`UPROEPRTY()`で扱えません。残念。

そこで、`StructUtil`を使用すれば、`UPROPERTY() FFoo* FooPtr;` を事実上実装可能になります。

:::message
**`TObjectPtr<TUObject>` の `USTRUCT`バージョンが `TInstancedStruct<T>`です**

* `UPROPERTY() TObjectPtr<UObject> Obj{};`
* `UPROPERTY() TInstancedStruct<FFoo> Foo{};`
:::


# StructUtilの主要型

* `FInstancedStruct`
* `TInstancedStruct`
* `FStructView`
* `FConstStructView`
* `FSharedStruct`
* `FConstSharedStruct`
* `FStructArrayView`
* `FConstStructArrayView`

ひとつずつ触れていきましょう。

---

# `FInstancedStruct`
`StructUtil`の中核の担う構造体です。

`FInstancedStruct` は型を保持したバイト配列です。
`FInstancedStruct`は 具象型`T`を隠蔽しつつも、型付けすることによって、型安全にデータの読み書きができます。任意の`USTRUCT`である`T`型を内部で保持することができます。

これだけならば、ヘルパー構造体を自前実装すれば済むのですが、`FInstancedStruct`の真骨頂はここからです。

* `UPROPERTY`サポート
* シリアライズサポート
* `Unreal Header Tool`:UHTサポート
* Detailsビューのようなエディタ統合のサポート
* Blueprintサポート

Unreal Editor上で問題なく触れます。エディタ統合が公式になされているという点は強いです。

## `FInstancedStruct` の使い方

`Make<T>`もしくは`InitializeAs<T>` で初期化して、 `Get<T>`で読みます。

```cpp
USTRUCT()
struct FMySettings
{
    GENERATED_BODY()
    UPROPERTY() FName Name{};
    UPROPERTY() int Difficulty = 100;
}

void Main()
{
    FMySettings Settings
    {
        .Name = TEXT("Hogehoge"),
        .Difficulty = 100,
    };

    // 初期化
    const FInstancedStruct InstancedStruct = FInstancedStruct::Make<FMySettings>(Settings);

    // readonly アクセス
    FMySettings const& Settings = Data.Get<FMySettings>();

    // read-write アクセス
    FMySettings& MutableSettings = Data.GetMutable<FMySettings>();
}
```

## `FInstancedStruct` の初期化

初期化方法はいくつかあります。どれを使ってもいいです。

```cpp
USTRUCT()
struct FMySettings
{
    GENERATED_BODY()
    UPROPERTY() FName Name{};
    UPROPERTY() int Difficulty = 100;
}

void Main()
{
    // デフォルトコンストラクタ
    const FInstancedStruct Data = FInstancedStruct::Make<FMySettings>();
    ensure(Data.Difficulty = 100);

    // パラメータパックの完全転送によるコンストラクタ呼び出し
    const FInstancedStruct Data = FInstancedStruct::Make<FMySettings>(/*Name*/TEXT("Hogehoge"), /*Difficulty*/42);
    ensure(Data.Difficulty = 42);

    // const&　渡しによる値コピー
    FMySettings Settings0
    {
        .Name = TEXT("Hogehoge"),
        .Difficulty = 42,
    };
    FInstancedStruct Data = FInstancedStruct::Make<FMySettings>(Settings0);

    // 完全転送による初期化
    FInstancedStruct Data = FInstancedStruct::Make<FMySettings>(MoveTemp(Settings0));
    ensure(Data.Difficulty = 42);

    // デフォルトコンストラクタ
    FInstancedStruct Data;
    Data.InitializeAs<FMySettings>();
    ensure(Data.Difficulty = 100);

    // パラメータパックによるコンストラクタ呼び出し割り当て
    FInstancedStruct Data;
    Data.InitializeAs<FMySettings>(TEXT("Hogehoge"), 100));

    // const& 渡しによる値コピー
    FMySettings Settings1;
    FInstancedStruct Data;
    Data1.InitializeAs<FMySettings>(Settings1);

    // 右辺値参照による内部割り当て
    FInstancedStruct Data;
    Data.InitializeAs<FMySettings>(MoveTemp(Settings1));
}
```
`Make<T>`で作る方式が`const`修飾できて一番良さそうな気がします。`Make<T>`は内部で`InitializeAs<T>`を呼び出しており、最適化されたら全部同じ処理になりそうなのでどれでもいいと思います。

:::message
いずれの初期化を使用してもメモリ実体はMallocされてヒープに存在します。メモリにコピーされるのでオリジナルの引数は捨てて問題ありません。
:::

### `USTRUCT`型制約
型`T`に指定できるのは`USTRUCT`限定です。これは型情報を得るために `UScriptStruct`が必要だからです。primitive型やnative型は使用できません。template型もだめです。

```cpp
void Main()
{
    FInstancedStruct Data;
    // コンパイルエラー. primitive型はUSTRUCT型ではない
    Data.InitializeAs<int>(42);
    // コンパイルエラー. FVectorはUSTRUCT型ではない
    Data.InitializeAs<FVector>(FVector::OneVector);
    // コンパイルエラー. TArray<int>はUSTRUCT型ではない
    Data.InitializeAs<TArray<int>>({1,2,3});
}
```

こういうときは `USTRUCT`にラップします。継承だと`USTRUCT`になれないので所有します。

```cpp
// サードパーティ型
#include "opencv/matrix.h"
USTRUCT()
struct FOpenCVMat44
{
    GENERATED_BODY()

    // native型なので UPROPERTY()にはできないよ
    cv::Mat Mat{};
}

// 所有バージョン
USTRUCT()
struct FMyArray1
{
    GENERATED_BODY()
    UPROPERTY() TArray<int> Numbers{};
}

// 継承バージョン. これはダメ
// コンパイルエラー
USTRUCT()
struct FMyArray2 : public TArray<int>
{
    GENERATED_BODY()
}

void Main()
{
    FInstancedStruct Data0 = FInstancedStruct::Make<FOpenCVMat>();
    FInstancedStruct Data1 = FInstancedStruct::Make<FMyArray1>();
    Data1.GetMutable<FMyArray1>().Numbers.Add(1);
    Data1.GetMutable<FMyArray1>().Numbers.Add(2);
    Data1.GetMutable<FMyArray1>().Numbers.Add(3);
}
```
## `FInstancedStruct` の保持

通常通り、`UPROPERTY() FInsancedStruct InstancedStruct{};`として持たせれば良いです。`USTRUCT`も`UCLASS`も変わりありません。

```cpp
// structバージョン
USTRUCT()
struct FHolder
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, meta = (BaseStruct = "/Script/MyProject.FFoo"))
    FInsancedStruct InstancedStruct{};
}

// classバージョン
UCLASS()
class UHolder : public UObject
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, meta = (BaseStruct = "/Script/MyProject.FFoo"))
    FInsancedStruct InstancedStruct{};
}
```

`BaseStruct` メタデータを付与することでベース型を指定できます。

このあたりのエディタ連携・BP連携はヒストリア様の記事を見た方が早いです。
https://historia.co.jp/archives/38415/

## `FInstancedStruct` は`UPROPERTY` 参照チェインを張る
非常に重要な神機能です。
親オブジェクトから`UPROPERTY() FInstancedStruct`として保持されるならば、`T`型のもつ`UPROPERTY() TObjectPtr<UObject> Object{};` への参照はちゃんとリンクします。
以下のコードをご覧ください。

```cpp
USTRUCT()
struct FMyData
{
    GENERATED_BODY()

    UPROPERTY() TObjectPtr<AActor> Owner{};
    UPROPERTY() TObjectPtr<UObject> Object{};
}

UCLASS()
class AMyActor : AActor
{
    GENERATED_BODY()

protected:
    virtual void BeginPlay() override
    {
        //エディタもしくはコンストラクタで FMyData派生型で初期化されているはず
        AnyData.Get<FMyData>().Owner = this;
        AnyData.Get<FMyData>().Object = NewObject<UObject>(this);
    }

    virtual void EndPlay() override
    {
        FMyData* MyData = AnyData.GetMutable<FMyData>();

        // 参照が追跡されているのでGCされていない
        ensure(IsValid(MyData.Object));
        ensure(IsValid(MyData.Owner));

        MyData.Object = nullptr;
        MyData.Owner = nullptr;
    }

private:
    // UPROPERTY()として追跡できるよ！
    UPROPERTY(EditAnywhere, meta = (BaseStruct = "/Script/MyProject.FMyData"))
    FInstancedStruct AnyData{};
}
```
`Level`→`AMyActor`→`AMyActor::AnyData`→`FMyData`→`FMyData::Object` という流れで参照チェインが到達可能です。そのため、自前で`NewObject`した`FMyData::Object`に格納された参照はGC回収を妨げます。これこそが `FInstancedStruct`の醍醐味でしょう。

これまで`TObjectPtr`を持つデータ型は参照チェインを張るために 
* `FGCObject`を継承して`AddReferenceObject`を実装する
* `UObject`を継承する

等の必要がありましたが、直接`Plain Old Data:POD`な構造体として定義し、`FInstancedStruct`に格納できます。`UObject`を継承しないことでメモリサイズを大きく削減できることになります。

### コンテナに格納しても参照チェインは張られるよ
`TArray`, `TMap`, `TSet`など`UPROPERTY()`対応済みコンテナに入れても大丈夫です。
```cpp
// UPROPERTY()として追跡できるよ！
UPROPERTY()
TArray<FInstanceStruct> AnyDataArray;

// UPROPERTY()として追跡できるよ！
UPROPERTY()
TMap<FName, FInstanceStruct> AnyDataMap;
```
## `FInstancedStruct` はBP関数対応

`BlueprintCallable`な`UFUNCTION`に渡せます。
`Delegate`の引数にも取れます。

`TInstancedStruct<T>`も同様です。

```cpp

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMyDelegate, const FInstancedStruct&, Data);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMyDelegateT, const TInstancedStruct<FMyBase>&, Data);

UCLASS()
class UHoge : public UObject
{
public:
    GENERATED_BODY()
	
    UFUNCTION(BlueprintCallable)
    void SetData(FInstancedStruct AnyData)
    {
        Data = AnyData;
        OnDataChanged.Broadcast(AnyData);
    }

    UFUNCTION(BlueprintCallable)
    void SetData2(const FInstancedStruct& AnyData)
    {
        Data = AnyData;
        OnDataChanged.Broadcast(AnyData);
    }

    UFUNCTION(BlueprintCallable)
    void SetData3(TInstancedStruct<FMyBase> AnyData)
    {
        DataT = AnyData;
        OnDataChangedT.Broadcast(AnyData);
    }

    UFUNCTION(BlueprintCallable)
    void SetData4(TInstancedStruct<FMyBase> AnyData)
    {
        DataT = AnyData;
        OnDataChangedT.Broadcast(AnyData);
    }

    UFUNCTION(BlueprintCallable)
    FInstancedStruct GetData1() const {return Data;}

    UFUNCTION(BlueprintCallable)
    const FInstancedStruct& GetData2() const {return Data;}

    UFUNCTION(BlueprintCallable)
    TInstancedStruct<FMyBase> GetData3() const {return DataT;}

    UFUNCTION(BlueprintCallable)
    const TInstancedStruct<FMyBase>& GetData4() const {return DataT;}

    UPROPERTY()
    FInstancedStruct Data;
    UPROPERTY()
    TInstancedStruct<FMyBase> DataT;

    UPROPERTY(BlueprintAssignable)
    FMyDelegate OnDataChanged;

    UPROPERTY(BlueprintAssignable)
    FMyDelegateT OnDataChangedT;
};
```

## `FInstancedStruct` の読み書き

`Get<T>`、`GetMutable<T>`を使います。実行時チェックが入っており型安全です。内部型と異なる型を与えると`check`で止まります。

`GetPtr<T>`, `GetMutablePtr<T>` は型が異なるとき(キャストに失敗したとき)は`nullptr`を返します。型がよくわからないときはコレでチェックできます。

```cpp
USTRUCT() struct FFoo : {GENERATED_BODY() };
USTRUCT() struct FFoo2 : public FFoo {GENERATED_BODY() };
USTRUCT() struct FBar : { GENERATED_BODY() };

void Main()
{
    FInstancedStruct Data;
    Data.InitializeAs<FFoo>();

    // 例外! FFoo型はFBar型と継承関係にないので checkに引っかかる
    const FBar& Bar = Data.Get<FBar>();

    // 例外! FFoo型はFBar型と継承関係にないので checkに引っかかる
    FBar& Vec = Data.GetMutable<FBar>();

    // チェックして使う
    if( FFoo* Ptr = Data.GetPtr<FFoo>())
    {
        Ptr->...;
    }
    else if(FFoo2 Ptr = Data.GetPtr<FFoo2>())
    {
        Ptr->...;
    }
}
```

## `FInstancedStruct` のCopyはDeepCopy
`operator=`や`コピーコンストラクタ`によるコピーはDeepCopyです。
```cpp
// 1KiBの構造体
USTRUCT()
struct FBigData1024{ GENERATED_BODY() uint8 Data[1024]{}; }

// 1KiBのMalloc
FInstancedStruct InstanceA = FInstancedStruct::Make<FBigData1024>();
InstanceA.GetMutable().Data[0] = 0xff;

// 1KiBのMallocを行いそこに書き込む
// FBidData1024はBlittableな構造体なのでそのままコピーされる
FInstancedStruct InstanceB = InstanceA;

ensure(InstanceA.GetPtr() != InstanceB.GetPtr());
ensure(InstanceB.GetMutable().Data[0] == 0xff);
```
迂闊に無駄なコピーをしないよう注意が必要です。ラムダキャプチャ時など明示的にコピーしたいときはコピーキャプチャを、転送したいときは`MoveTemp`で送ってください。
この辺は`TArray`の扱いと同様です。

:::message
`const FInstancedStruct&`で引き回すのも悪くはないですが、後述の`FStructView`, `FConstStructView`を使うとコピーレスにviewを引き回せます。
:::

---

# `FInstancedStruct` の詳解
ここからが本題です。

`FInstancedStruct` の実体は 型情報とバイト列のペアです。ポインタを2つだけ持つ構造体ですので`sizeof(FInstancedStruct)` は 64bit環境で16バイトです。

以下疑似コードです。
```cpp
USTRUCT()
struct FInstancedStruct
{
    // リフレクションシステムによる型情報
    const UScriptStruct* StructClass{};
    // データ実体
    uint8* Memory{};
}
```

仕組みは非常に単純で、初期化時に`sizeof(T)`の分だけメモリを割り当てて、データ領域を構築します。読み書きするときは `T`型に`reinterpret_cast`することでアクセスします。

以下疑似コードです。
```cpp
struct FInstancedStruct
{
    // 初期化
    template<class T, typename... TArgs>
    void InitializeAs<T>(TArgs&&... InArgs)
    {
        // alignof(T)のアラインメントでsizeof(T)のメモリを確保
        this->Memory = FMemory::Malloc(sizeof(T), alignof(T)));
        // 完全転送 と placement new で構築する
        new (Memory) T(Forward(InArgs)...);
    }

    // デストラクタ
    ~FInstancedStruct()
    {
        //先頭アドレスをT*にキャストしてデストラクタ呼び出し
        reinterpret_cast<T*>(Memory)->~T(); 
        // メモリ解放
        FMemory::Free(Memory);
    }

private:
    const UScriptStruct* StructClass{};
    uint8* Memory{};
}
```

データ実体はヒープ上にメモリ割り当てが行われて`T`型に構築されるという点を抑えておけばOKです。そして実体が`uint8*`であるにもかかわらず適切に`T*`としてシリアライズされたりリフレクションから辿れるすごい奴ということです。

:::message
説明のための疑似コードです。実際の実装は`UScriptStruct`を使用して`GetStructureSize()`でサイズ計算したりアラインメントあわしたりしてます。BPクラスをサポートするためリフレクション経由で算出されます。概念だけ理解してください。
:::

---

# `TInstancedStruct`

一言でいえば、`TSubclassOf<T>`の `USTRUCT`バージョンです。

`FInstancedStruct` は任意の`USTRUCT`が格納できました。
任意の`USTRUCT`を格納できるということは、逆に言えば何でもかんでも格納できてしまう、ということです。`FInstancedStruct`に格納する型`T` は実行時に何度でも変更できます。`InitializeAs<T1>`で初期化した後、`InitializeAs<T2>`で同じインスタンスを別の型に初期化しなおすことが可能です。実行時型付けということは静的解析できませんし、特定の型に制約出来た方が取り回しは良くなることがあります。


`TInstancedStruct` はこの点を解決しています。
`TInstancedStruct` は `FInstancedStruct` を型付けする`thin`ラッパーな`template`クラスです。
`TInstancedStruct<T1>` は 型`T1`の派生型のみを受け付け、`TInstancedStruct<T2>`は型`T2`の派生型のみを受け付けます。

## `TInstancedStruct<T>`の初期化
FInstancedStructと同様です。
ただし、`template`クラスなのでコンパイル時に静的チェックできます。

少しややこしいのですが、型パラメータ`TBase`の派生型`TDerrieved`で `InitializeAs`できます。
```cpp
USTRUCT() struct FMyBase{ GENERATED_BODY() };
USTRUCT() struct FMyDerivedA : public FMyBase { GENERATED_BODY() };
USTRUCT() struct FMyDerivedAA : public FMyDerivedA { GENERATED_BODY() };
USTRUCT() struct FMyDerivedB : public FMyBase { GENERATED_BODY() };

void Main()
{
    // 型パラメータがFMyDerivedAであることに注意
    TInstancedStruct<FMyDerivedA> ASubClass;

    // OK. 型パラメータと同じ型
    ASubClass.InitializeAs<FMyDerivedA>();

    // OK. 型パラメータの派生型
    ASubClass.InitializeAs<FMyDerivedAA>();

    // コンパイルエラー!
    // FMyBaseは FMyDerivedA のベースクラスであってサブクラスではない
    ASubClass.InitializeAs<FMyBase>();

    // コンパイルエラー!
    // FMyDerivedBは 共通の祖先をもつが、FMyDerivedAのサブクラスではない
    ASubClass.InitializeAs<FMyDerivedB>();
}
```

## `TInstancedStruct<T>`の読み書き
`Get`するときは`InitializeAs`で与えた型と一致させましょう。
格納されている型が派生型か確信が持てないときは `GetPtr<T>`でチェックしてください。

```cpp
void Main()
{
    // TInstancedStruct<FMyDerivedA> を FMyDerivedAA で初期化してよい.
    TInstancedStruct<FMyDerivedA> ASubClass;
    ASubClass.InitializeAs<FMyDerivedAA>();

    // 型パラメータを省略してGetできるのは FMyDerivedA型
    const FMyDerivedA& DerivedA = ASubClass.Get();

    // 派生型は明示的に指定すればOK
    const FMyDerivedAA& DerivedAA = ASubClass.Get<FMyDerivedAA>();

    // コンパイルエラー. アップキャスト方向はできない
    // std::is_base_of_vを満たせないから
    const FMyBase& Base = ASubClass.Get<FMyBase>();
}
```

## `TInstancedStruct<T>`は`UPROPERTY`サポート
`template`クラスではありますが、`UPROEPRTY()`に対応しています。
Unreal Header Tool で直接名指しでサポートされています。

```cpp
UCLASS()
class UHoge : public UObject
{
    GENERATED_BODY()

    // FMyDerivedA と FMyDerivedAAが格納できる
    UPROPERTY(EditAnywhere, Category = Foo)
    TInstancedStruct<FMyDerivedA> A_SubClass;

    // FMyDerivedB を格納できる
    UPROPERTY(EditAnywhere, Category = Foo)
    TInstancedStruct<FMyDerivedB> B_SubClass;
}
```

仕組みとしては `FInstanceStruct`の `UPROPERTY`対応にそのままのっかっているようです。

## `TInstancedStruct<T>`は自動でメタタグ付与する
`UPROPERTY() TInstancedStruct`にはUHT経由で自動的に型`T`に対する`BaseStruct`メタタグが付与されます。楽ちんです。

逆に、手動で`BaseStruct`メタタグを付与しようとするとコンパイルエラーとなります。


```cpp
tempalte<typename T>
struct TInstancedStruct
{
    UPROPERTY( meta = (BaseStruct = "ここに型Tの名前が自動で記載される仕組みみたい")))
    FInstancedStruct Struct;
}
```

## `TInstancedStruct<T>`の型エイリアスはダメ
UHTサポートは`"TInstancedStruct"`という文字列に対する名指しなので型エイリアスは使えません。

```cpp
using FMyInstancedBase = TInstancedStruct<FSuperUltraLongLongName>;

UCLASS()
class FFoo
{
    // コンパイルエラー
    // FMyInstancedBase は `USTURCT`でないからわからない
    UPROPERTY(EditAnywhere)
    FMyInstancedBase Base;

    // 本名で記載するべし
    UPROPERTY(EditAnywhere)
    TInstancedStruct<FMyBase> Base;
}
```

## `TInstancedStruct<T>`を拡張するなら所有しかない
もし拡張したい場合は所有します。継承するとUSTRUCTにはできません。

```cpp
USTRUCT()
struct FMyInstancedBaseEx
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    TInstancedStruct<FMyBase> Base;

    UPROPERTY(EditAnywhere)
    int ExtraValue = 42;
};

// コンパイルエラー. 継承元がUSTRUCTじゃないから USTRUCTになれない
USTRUCT()
struct FMyInstancedBaseEx : public TInstancedStruct<FMyBase>
{
    GENERATED_BODY()
};
```

## `TInstancedStruct<T>`は不完全型 OK
前方宣言でOKです。依存性の連鎖を下げられます。

```cpp
UCLASS()
class UHoge : public UObject
{
    GENERATED_BODY()

    // 不完全型で型指定してよい
    UPROPERTY(EditAnywhere)
    TInstancedStruct<struct FIncompleteType> Incomplete;
};
```

--- 

# `FInstancedStruct` vs `TInstancedStruct<T>`
ガイドライン

:::message
* 型が分かるなら`TInstancedStruct<T>`が 良い
* 型が分からないなら`FInstancedStruct`が 良い
:::


基本性能や使い方はどちらも同じです。一番の違いは型情報をコンパイル時に与えるか否かです。可能であるならば、`TInstancedStruct<T>`の方が型安全です。格納する内部型`T`が分かっているならば`TInstancedStruct<T>`で型付けした方がよいです。

プラグイン開発などで、本当にあらゆる型を受けいれたい場合は`FInstancedStruct`を使います。Blueprintクラスを受け入れたいときも `FInstancedStruct`になるでしょう。C++ビルド時にBPクラスはまだ存在しませんから。しかしながら、BPクラスの基底クラスが`AMyActorBase`のように定められる状況であれば、`TInstancedStruct<AMyActorBase>`の方が強いでしょう。

実例として`StateTree`プラグインは 任意のユーザー型を引き回すために`FInstancedStruct`を多用しています。状態遷移を組むときにどのような情報が必要になるかを決定できないからですね。

## `TInstancedStruct<T>`も`FInstancedStruct`もStructOps対応

`FInstancedStruct` には `struct TStructOpsTypeTraits<FInstancedStruct>` が定義されています。
`Serialize`関数や`NetDeltaSerialize`等全て備えているので、隙がありません。
つまり`Replication`も簡単ってこと。

```cpp
template<>
struct TStructOpsTypeTraits<FInstancedStruct> : public TStructOpsTypeTraitsBase2<FInstancedStruct>
{
	enum
	{
		WithSerializer = true,
		WithIdentical = true,
		WithExportTextItem = true,
		WithImportTextItem = true,
		WithAddStructReferencedObjects = true,
		WithStructuredSerializeFromMismatchedTag = true,
		WithGetPreloadDependencies = true,
		WithNetSerializer = true,
		WithFindInnerPropertyInstance = true,
		WithClearOnFinishDestroy = true,
		WithVisitor = true,
	};
};
```

# `TInstancedStruct<T>` 詳解
本題その２です。

といってもあまり書くことはありません。ほとんど `FInstancedStruct` の機能にのっているからです。

## `TInstancedStruct<T>` は `FInstancedStruct`として扱われる
リフレクション層では `TInstanedStruct` は `FInstancedStruct`として扱われます(!)

```cpp: 疑似コード
template<class T>
struct TInstancedStruct
{
    FInstancedStruct InstancedStruct;
}
```
このように`FInstancedStruct`を所有しているだけなので、`FInstancedStruct`に`reinter_pret`キャストできます。リフレクション機能などは `FInstancedStruct`にキャストしてしまって、`FInstancedStruct`用のAPIを利用しているようです(!)

もし`TInstancedStruct`をエンジン改造する際は メモリレイアウトが1byteも変わらないようにしないといけませんね。

## `TInstancedStruct<T>` の UHTサポート
`TInstancedStruct`はUHT側にて名指しでサポートされています。

UHTが自動で `BaseStruct`metaデータを付与します。
C++ファイル解析により、型パラメータとして与えられた`USTRUCT`が判明しているので、プロパティを付与しています。
そられのプロパティ情報を使用して、最終的に `gen.cpp` に書き込まれるのでしょう。(あんまり追いかけてない)

`UhtStructProperty.cs` に実装が存在しますので、詳しくそちらをご覧ください。

```cs: UhtStructProperty.cs
[UhtPropertyType(Keyword = "TInstancedStruct")]
private static UhtProperty? InstancedStructProperty(...)
{
    // ...
}
```

## `TInstancedStruct<T>` の型情報の静的チェック
`is_base_of`などC++の`type_traits`をバリバリ使っています。

`Make<T>`, `InitializeAs<T>`, `Get<T>`といった`template`関数は型`T`が`USTRUCT`であるか、初期化時に指定した型の制約を満たしているかを静的チェックしています。誤った型を使って初期化できないようになっています。静的解析ですので、コンパイルエラーとなります。初期化で与えるコンストラクタ引数も適合するコンストラクタがなければコンパイルエラーとなります。

`TInstancedStruct<T>`はコンパイル時に型情報が確定するので、静的チェックに強いです。
可能であるならば、`FInsatncedStruct`を使うよりも `TInstancedStruct<T>`の方が便利です。

## `TInstancedStruct<T>` の型情報の実行時チェック
BP構造体やユーザー構造体を受ける場合など、コンパイル時に決定できないことがあります。
その場合でも`FInstancedStruct`は型情報を保持しているため、 `Get<T>`で与えた型が初期化時の型`T`とコンパチブルか`check`しています。
未定義動作で生動きする危険が大幅に減りました。

---

# `FStructView` と `FConstStructView` 

`FStructView` と `FConstStructView`はほぼ同じです。`FStructView` は `FConstStructView`と異なり `T`を書き換えられるだけです。
そのため`FConstStructView`について説明します。

`FConstStructView`は 構造体へ型付けされた`FInsancedStruct` に対応する、`readonly`なビューです。
`FInsancedStruct`と異なり、こちらはメモリを所有しません。メモリ領域の先頭アドレスを保持して`T`型でアクセスするだけです。

ちいさな値オブジェクトです。そのデータ実体がどこにあるかを隠蔽しつつデータを扱いたい場合に利用します。生ポインタ`T*`と何が違うのかというと、型情報を持っている分、生ポよりも型安全です。

データ実体がどこにあるかを隠蔽するので、
* スタック領域
* ヒープ領域
* 静的メモリ領域
* メンバーフィールド
* TArrayの部分区間

だろうがなんだろうが、とにかく生ポ代わりに構造体を渡す操作をしたいときに使えます。

## View のreadonly 参照
書き換えられません。
`Get`では `const T`しか得られません。

```cpp
void Use(FInstancedStruct& InstancedStruct)
{
    FConstStructView View(InstancedStruct);

    // 型パラメータにはconst修飾すること
    const FMyBase& ReadonlyRef = View.Get<const FMyBase>();
    const FMyBase* ReadonlyPtr = View.GetPtr<const FMyBase>();

    // 型パラメータのconst修飾を省くと警告でる.
    const FMyBase& ReadonlyRef = View.Get<FMyBase>();
}
```

余談ですが、`FConstStructView`自体は`mutable`です。
```cpp
void Use(FInstancedStruct& InstancedStruct)
{
    FConstStructView View(InstancedStruct);
    ensure(View.IsValid() == true);

    // 内部ポインタをnullptrにする
    View.Reset();
    ensure(View.IsValid() == false);
}
```

## Viewをcopyレスに渡す

`FInstancedStruct` のコピーはメモリをDeepCopyしていました。
`FConstStructView`は コピーせずに同じメモリ領域への`readonly`参照を提供します。

`Make`関数で作成できます。
```cpp
FInstancedStruct Instance;
FConstStructView View = FConstStructView::Make(Instance);
```

小さなオブジェクトなので オブジェクトの値渡しでＯＫです。下手に参照渡しにするよりも値渡しでＯＫです。ビューですから。

```cpp
static void PassByValue(FConstStructView View){} //値渡しでOK
static void PassByRef(FConstStructView& RefView){} //参照渡しにする意義がない
static void PassByRValue(FConstStructView&& RefView){} //転送する意義がない

static void Main()
{
    FInstancedStruct Instance;
    FConstStructView View = FConstStructView::Make(Instance);
    PassByValue(View);

    // ワンライナーで渡すことが多いかも
    PassByValue(FConstStructView::Make(Instance));
}
```

意味論的にビューは「メモリの所有権を持たない」、と契約していることが重要です。
生ポインタだと所有権を持っているかどうかは判断できませんが、ビューならば持たないことは確実です。

ユーザーはビューが指し示すメモリ領域の寿命を知りません。そのため、`FConstStructView`が渡されたときその寿命は関数スコープだと考えるべきです。
ラムダキャプチャしたりMoveTempで送るときは気を付けましょう。

## Viewから意図的にDeepCopyできる

ワーカースレッドに渡したりキューに溜めるときにビューのままだと困ります。そういうときは `FInstancedStruct`に戻してコピーします。
具象型を知らずにデータをデータのまま扱えます。

```cpp
TQueue<FInstancedStruct> Queue;

void Enqueue(FConstStructView PayloadView)
{
    // コンストラクタでビューからコピー
    FInstancedStruct Copy(PalyloadView);
    // Make関数でビューからコピー
    FInstancedStruct Copy = FInstancedStruct::Make(PalyloadView);
    Queue.Enqueue(Copy);

    // FConstStructViewからFInstancedStructへの暗黙的な型変換はできない
    Queue.Enqueue(PayloadView);
}

```

## Viewをstackメモリ領域から作る

ビューはメモリ領域を隠蔽しますので、メモリがどこに存在していても寿命にさえ気を付ければ問題ありません。データが十分に小さかったり、寿命が短かったり、一時的なデータであるなど、条件を満たせばmallocレスで使用できます。static領域であるなら寿命は気にしなくすみます。

```cpp

USTRUCT()
struct FFoo
{
    GENERATED_BODY()
    UPROPERTY() int32 Value;
}

TFunction<void(FConstStructView View)> OnDataChanged;

// 送信側はmallocなしで送ってもいい.
void Publish()
{
    // 一時変数としてスタックメモリ上にインスタンスを構築
    FFoo Foo{.Value = 42};

    // スタックメモリ上のアドレスを指してViewに変換
    FConstStructView PayloadView(FFoo::StaticStruct(), reinterpret_cast<const uint8*>(&Foo));

    // 送信
    OnDataChanged(PayloadView);

}// ここでFooの寿命が尽きる

// 受信側はメモリ領域を意識しなくていい
void Receiver()
{
    OnDataChanged = [](FConstStructView View)
    {
        int32 Value = View.Get<FFoo>().Value;
        ensure(Value == 42);
    }
}
```

無駄な一時オブジェクトのmallocやコピーを省略しており、効率的です。
Delegateを利用する側はメモリ領域を意識する必要がありません。
`FStructView`のみに依存すればいいです。具象型`T`も自身の興味ある型だけでよいです。


:::message
あまり無茶苦茶な使い方をするとダングリングViewになるので気を付けてください
:::

---

# つづく
