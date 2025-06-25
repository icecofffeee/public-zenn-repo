---
title: "UE5:Unreal EngineのStructUtilについてまとめた 後編"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: false
---

# はじめに
StructUtilシリーズ第三弾です。
本稿は後編です。

* [UE5:Unreal EngineのStructUtilについてまとめた 前編](https://zenn.dev/allways/articles/ue5-structutil-research)
* [UE5:Unreal EngineのStructUtilについてまとめた 中編](https://zenn.dev/allways/articles/ue5-structutil-research-second)

# StructUtilの主要型
再掲。

* `FInstancedStruct`
* `TInstancedStruct`
* `FStructView`
* `FConstStructView`
* `FSharedStruct`
* `FConstSharedStruct`
* `TSharedStruct<T>`
* `TConstSharedStruct<T>`
* `FStructArrayView` <- 本稿
* `FConstStructArrayView`
* `FInstancedStructContainer`

ひとつずつ触れていきましょう。

# `FStructArrayView`

`FStructArrayView` は `TArray<T>` や`T[]`から `T`を隠蔽して、共通のインターフェースを提供するビューです。型消去された連続データ領域への透過的なビューです。`FStructArrayView` は`FStructView`の配列バージョンであるとみなして構いません。実際に、`FStructView[]` のように振る舞います。

## `FStructArrayView`の作り方

コンストラクタで何らかの配列を渡してあげます。

```cpp
USTRUCT() struct FFoo
{
    GENERATED_BODY()

    UPROPERTY() FVector Position;
    UPROPERTY() float Yaw;
}

void Main()
{
    TArray<FFoo> FooArray;
    FooArray.SetNum(3);

    //case1: TArray<T>&を受けるコンストラクタ
    {
        FStructArrayView StructArrayView(FooArray);
        ensure(StructArrayView.GetData() == FooArray.GetData()); //同じ領域を指します
    }


    //case2: TArrayView<T>を受けるコンストラクタ    
    {
        TArrayView<FFoo> ArrView(FooArray);
        FStructArrayView StructArrayView(ArrView);
        ensure(StructArrayView.GetData() == ArrView.GetData()); //全て同じ領域を指します
        ensure(StructArrayView.GetData() == FooArray.GetData()); //同じ領域を指します
    }

    //case3: 生配列. (TStaticArray<T>使え)
    {
        FFoo RawArray[3]{};
        FStructArrayView StructArrayView(FFoo::StaticStruct(), &RawArray[0], 3);
    }

    //case4: 固定長配列
    {
        using FFooArray3 = TStaticArray<FFoo, 3>;
        FFooArray3 StaticArray{};
        UScriptStruct* ScriptStruct = FFooArray3::ElementType::StaticStruct();
        FStructArrayView StructArrayView(ScriptStruct, &StaticArray[0], StaticArray.Num());
    }
}
```

:::message
viewなのでデータを所有しません。ポイントしているデータ領域の寿命が有効なことをcaller側できちんと保障してください。
:::


## `FStructArrayView`の読み書き

配列なので`for`文を使って走査します。
`operator[]` では `FStructView`が返ってきます。
```cpp: for と添え字演算子
void ReadData(FStructArrayView Array)
{
    for(int i=0; i < Array.Num(); ++i)
    {
        FStructView ElementView = Array[i];
        FFoo& Foo = ElementView.Get<FFoo>();
    }
}
```

`iterator`が実装してあるので、`range-based for` も使えます。
```cpp: range-based for
void ReadData(FStructArrayView Array)
{
    // Viewなので　FStructView& ではなく FStructViewで受け取っても別にいい
    for(FStructView ElementView : Array)
    {
        FFoo& Foo = ElementView.Get<FFoo>();
        FVector Pos = Foo.Position;
    }
}
```

直接具象型`T`を触りたいときは`GetAt<T>`で `T&`を得られます。型指定を間違ったときは `check`で止まります。この文脈では必ずこの型である、と分かっているなら`GetAt<T>`を使う方が`Shipping`時に型判定を省略できてお得です。

```cpp: read/write
void WriteData(FStructArrayView Array)
{
    for(int i=0; i < Array.Num(); ++i)
    {
        FFoo& Foo = Array.GetAt<FFoo>(i);
        Foo.Position = FVector(1,2,3);
    }
}
```

型が不明であり実行時型判定をしたいときや無効値が入る可能性があるときは、`GetScriptStruct()`を使うか、`GetPtrAt<T>`で`nullptr`判定します。
```cpp: read/write
void WriteData(FStructArrayView Array)
{
    for(int i=0; i < OutArray.Num(); ++i)
    {
        // nullptrは型指定を間違えたとき返ってくる
        // 元が配列なのでデータ領域は必ず存在しているから純粋なnullptrはありえない
        if(FFoo* Foo = OutArray.GetPtrAt<FFoo>(i))
        {
            Foo->Position = FVector(1,2,3);
        }
    }
}
```

### `GetAt<T>` vs `GetPtr<T>`

`FStructArrayView`はコンストラクタで実配列を渡す仕組みであるため、無効値を渡すことは非常に難しいです。`FStructArrayView`そのものが無効であるときは`Num()==0`になるので`for`文の中を通りません。そのため、`for`文の中で`GetPtrAt<T>`を使うよりも、`for`文の外で型判定する方が良さそうです。配列という特性上すべての要素は同じ型です。

```cpp
    // 1回の型判定
    if(Array.GetScriptStruct()->IsChildOf<FFoo>())
    {
        for(int i=0; i < Array.Num(); ++i)
        {
            FFoo& Foo = Array.GetAt<FFoo>(i);
        }
    }

    // N回の型判定
    for(int i=0; i < OutArray.Num(); ++i)
    {
        if(FFoo* Foo = OutArray.GetPtrAt<FFoo>(i)){}
    }
```

### `FStructArrayView`の厳密な型判定
上記例では `IsChildOf<T>`で `T`と`T派生型`を許容しました。必ず特定の型じゃないといけないときは`T::StaticStruct()`と比較してください。
```cpp: read/write
    // exactly equal
    if(Array.GetScriptStruct() == FFoo::StaticStruct()){}
```

## `FStructArrayView`の`Slice`

`FStructArrayView` は `ArrayView`なので 部分区間を`Slice`できます。
`std::span`や `TArrayView`とほぼ同じです。

```cpp
// 100個の配列
TArray<int> Array;
Array.SetNum(100);
for(int i=0; i < 100; ++i)
{
    Array[i] = i;
}

// 全区間 [0-99]
FStructArrayView View(Array);

// 0番目から50個の部分区間 [0-49]
FStructArrayView Slice0_49 = View.Slice(0, 50);

// 50番目から50個の部分区間 [50-99]
FStructArrayView Slice50_99 = View.Slice(50, 50);

// 左から数えて4個目までの区間 つまり4個
//  left 
// [0 1 2 3] [4...]
FStructArrayView LeftSlice = View.Left(4);

// 右から数えて5個目までの区間 つまり5個
//               right
// [0 ... 94] [95 96 97 98 99]
FStructArrayView RightSlice = View.Right(5);

// 左側から (100-6)番目までの区間 つまり94個
// leftChop
// [0 ... 93] [94 95 96 97 98 99]
FStructArrayView LeftChop = View.LeftChop(6);

// 右側から (100-7)番目までの区間 つまり93個
//                  rightChop
// [0 1 2 3 4 5 6] [7 ... 99]
FStructArrayView RightChop = View.RightChop(7);
```

ややこしいので、`Slice()`だけ使えばいいと思いました。Sliceを2分割する`Mid`もあります。
2分探索やクイックソートのような部分区間を使うアルゴリズムでは威力を発揮しそうです。

# `FStructArrayView`の詳解
本題です。

`FStructArrayView`は `TArrayView<T>`から型情報を取っ払ってリフレクション情報を使うようにした配列用の`view`として実装されています。全てを`void*`で持つことで、ありとあらゆる型を受けられるようになっています。反面、`void*`にすることで静的な型安全性が失われるのですが、`type_trait`と`UScriptStruct`を駆使することで型安全性を補っております。

`FStructArrayView`の疑似コードです。

```cpp: 疑似コード
struct FStructArrayView
{
    void* DataPtr = nullptr; // データ先頭
    const UScriptStruct* ScriptStruct = nullptr; // 型情報
    uint32 ElementSize = 0; // sizeof<T>
    int32 ArrayNum = 0; // 要素数
}
```

`void* DataPtr;` でメモリを保持しています。`UScriptStruct*`に型情報を持っています。
`ElementSize`に型`T`の実際の要素サイズを保持しています。型情報をコンパイル時に得られないため`sizeof<T>`が決まらないからです。


:::message
実際の`ElementSize`は `sizeof<T>`ではなく `UScriptStrct::GetStructSize()`で得たサイズです。`T`を継承した`Blueprint`型は`sizeof<T>`より少し大きいのです。
:::

`FStructArrayView`では`i`番目の要素を得るために、`ElementSize`に要素サイズつまりストライド量を覚えておき、そのストライドでポインタを操作します。
色々省略するとこうなります。

```cpp: 疑似コード
struct FStructArrayView
{
    template<class T>
    T* GetPtrAt(int index)
    {
        uint8* DataHead = static_cast<uint8*>(DataPtr); // 1
        uint8* ElementHead = Head + (ElementSize * Index); //2
        void* Ptr = ElementHead; //3
        return static_cast<T*>(Ptr); //4
    }
}
```
興味深いので1行ずつ解説します。

```cpp
        uint8* DataHead = static_cast<uint8*>(DataPtr); // 1
```
`void* DataPtr;` をいったん`uint8*`にキャストします。これはコンパイラ依存の操作を避けるための重要な処理です。

前提情報としてポインタに対する算術演算はポインタの指す型のサイズだけオフセットするという操作です。つまり`T*`への算術演算は`sizeof<T>`バイトのストライドでオフセットすることになります。ところが`void*`は指し示す型がありません。じゃあ`sizeof<void>`だけオフセットすればよいのではと考えるわけですが、`sizeof<void>`はコンパイルエラーとなります。`void`は"無”を意味するもので`data-type`ではありません。よって`sizeof<void>`は定義できません。

では、`void*`に対する算術演算はどうなるのでしょうか？
答えはコンパイラ依存です。[msvc](https://learn.microsoft.com/ja-jp/cpp/cpp/raw-pointers?view=msvc-170)は 1byteのようです。

> void* は、char (1 バイト) のサイズでインクリメントされます。 型指定されたポインターは、指し示す型のサイズによってインクリメントされます。

コンパイラ依存だと困るので、いったん`void*`から`uint8*`にキャストします。
`sizeof<uint8>` は 1byteなので`uint8*`に対する演算はbyte単位での操作となります。

ポインタのバイト単位操作ができるようになったので、次は要素のアドレスを得ます。

```cpp
        uint8* ElementHead = Head + (ElementSize * Index); //2
```
`(ElementSize * Index)` バイトだけずらせば`index`番目のアドレスが取れます。

```cpp
        void* Ptr = ElementHead; //3
```
void* へ戻して受けます。

```cpp
        return static_cast<T*>(Ptr); //4
```
index番目の要素を指す`void*`が得られたので、 `T*`にキャストしちゃいます。
これにて安全に`GetPtrAt<T>`が実装できました。

(`void*`からのキャストは`static_cast`なのすっかり忘れてました)

## `FStructArrayView`は`Base`型の配列としてアクセスできる

上記のようなややこしい手順を踏まずとも、最初から`DataPtr`を `T*`にキャストして `((T*)DataPtr)+Index`ではダメなのでしょうか？

答えはダメです。`GetPtrAt<T>`で`Base`型へアップキャストしたいからです。`GetPtrAt<T>`のテンプレートパラメータ`T`がインスタンスの型ではなくその`Base`型だったとき、`sizeof<Base>`と `ElementSize`が一致する保障はありません。`ElementSize`から算出する必要があります。

```cpp: base-type array
USTRUCT() alignas(4) struct FBase{ GENERATED_BODY() int Value0=1; }
USTRUCT() alignas(4) struct FFoo : public FBase { GENERATED_BODY() int Value1=2; }
USTRUCT() alignas(4) struct FBar : public FFoo { GENERATED_BODY() int Value2=3; }

static_assert(sizeof<FBase>() == 4);
static_assert(sizeof<FFoo>() == 8);
static_assert(sizeof<FBar>() == 12);

FBar Array[10] = {}; // 12byte * 10 =120byte
FStructArrayView View(FArrayView(Array));

// View は FBar[]である
FBar& Bar = View.Get<FBar>(); // ok

// Base[]としてストライドアクセス可能
// sizeof<Base>は4byteだがちゃんと12byteストライドでアクセスされるので安全
FBase* Base5 = View.GetPtrAt<FBase>(5); 
check(Base5 != nullptr); // 有効な値を返す
check(Base5->Value0 == 1); // 有効な値を返す

// FFoo[]としてストライドアクセス可能
// sizeof<FFoo>は8byteだがちゃんと12byteストライドでアクセスされるので安全
FFoo* Foo6 = View.GetPtrAt<FFoo>(6);
check(Foo6 != nullptr); // 有効な値を返す
check(Foo6->Value1 == 2); // 有効な値を返す
```
ストライドを保持しているため、`FFoo[]`を `FBase[]`としてアクセスできることが分かりました。`FStrideArrayView`と同じように使えます。


# `FStructArrayView` vs `TArrayView<T>`
`TArrayView<T>` もまた配列へのビューなのですが、こちらは型付けされており`T`が分かっています。`template`クラスであるが故に型が違えば別のクラスです。そのため、`TArrayView<T1>` と `TArrayView<T2>`は別の型でありそれぞれ互換性はありません。共通のインターフェースを提供したいときにこれでは不便なことがあります。


```cpp: StructUtilがないとき 
#include "FooModule/FFoo.h" // 不完全型は使いたくないのでFFooに依存する

struct IDataAccess
{
    // データを渡したいだけなのにオーバーロードを沢山用意しないといけない...
    // 値の意味を型付けしたいときはどんどん増える......
    virtual void SetVector3s(TArrayView<FVector> Datas) = 0;
    virtual void SetNames(TArrayView<FName> Datas) = 0;
    virtual void SetFoos(TArrayView<FFoo> Datas) = 0;
}
```

ベースクラスを指定して`TArrayView<FFoo*>`にすればポインタにしたら多少は幅広く受けられます。`include`しなくて済みます。
```cpp: ポインタじゃねんだわ
struct FFoo; // 前方宣言で許されるものの
struct IDataAccess
{
    // ポインタ配列を渡したい訳じゃないんだよな
    virtual void SetFoos(TArrayView<FFoo*> PointerArray) = 0;
}
```
よく見るとデータ配列ではなくポインタ配列になっています。これではキャッシュ効率上の不利があります。配列は連続したメモリ領域に確保される、というせっかくの仕様が台無しです。
データ指向を保ちつつも型情報を除去したい。そういうときにこそ `FStructArrayView`を使います。


```cpp: StructUtilがあるとき
#include "StructUtil/StructArrayView.h"

struct IGenericDataAccess
{
    // ありとあらゆる USTRUCT派生型の配列を渡せる
    virtual void SetDatas(FStructArrayView CommonDatas) = 0;
}
```
`FFoo`への`include`文は消え失せて、エンジン標準の `StructArrayView`だけに依存するようになりました。`FStructArrayView`は `UScriptStruct*` という型情報を持っていますから、中身がどのようなデータ配列なのかを実装側で期待することができます。

```cpp: impl.cpp
#include "IGenericDataAccess.h"

#include "FooModule/FFoo.h" // FFooに依存する

struct FMyDataAccess : public IGenericDataAccess
{
    virtual void SetDatas(FStructArrayView CommonDatas) override
    {
        if( CommonDatas.GetScriptStruct() == FFoo::StaticStruct())
        {
            for(int i=0; i < CommonDatas.Num(); ++i)
            {
                FFoo& Foo = CommonDatas.GetAt<FFoo>();
                ThisDatas.Add(Foo);
            }
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("期待した型ではないので何もしない"));
        }
    }

    TArray<Foo> ThisDatas{};
}
```
`Foo`への依存はcpp側に封じ込めることができました。大変疎結合になりました。

# `FStructArrayView` vs `TArray<FInstancedStruct>`
`FStructArrayView`が型消去した配列へのビューであるというならば、汎用型の配列を使えばいいじゃんと思うかもしれません。両者の違いについて述べておきましょう。

* `FStructArrayView` は 配列への汎用的なビュー
* `TArray<FInstancedStruct>`は汎用型の配列

## データレイアウトが違う

`FStructArrayView`は `T[]`をラップするものであるため、必ず連続したメモリ領域を指しています。そして、`T[]`であるため`i`番目と `j`番目で必ず同じ型`T`のデータです。

一方、`TArray<FInstancedStruct>` は実質的にポインタの配列です。[前編](https://zenn.dev/allways/articles/ue5-structutil-research)で述べたように`FInstancedStruct`は内部に型情報とポインタを持つ構造体でした。よって配列内のそれぞれの要素が指すデータ領域は別々の`heap`に確保されています。メモリブロックの連続性は保証されていません。ポインタ配列であるため、 `i`番目と`j`番目でそれぞれ別々の型`Ti`, `Tj`を持つことができます。`Ti`と`Tj`は継承関係にある必要もありません。

このことはインスタンスを作成してみるとよくわかります。
```cpp: FStructArrayView
USTRUCT() struct FDamage{ GENERATED_BODY() };
USTRUCT() struct FHitInfo{ GENERATED_BODY() };
void Main()
{
    TArray<FDamage> Damages;
    TArray<FHitInfo> Hits;

    // コンパイルエラー. こんなコンストラクタはない
    FStructArrayView View(Damages, Hits);


    // checkで止まる. 
    // FDamage配列をViewに入れて型を隠蔽したとてFHitInfoは入れられない
    FStructArrayView DamageView(Damages);
    DamageView[0].Get<FHitInfo>() = FHitInfo();
}
```

`TArray<FInstancedStruct>`なら `FDamage`も`FHitInfo`も入れられます。

```cpp: TArray<FInstancedStruct>
void Main()
{
    TArray<FInstancedStruct> GenericArray;
    GenericArray.Add( FInstancedStruct::Make<FDamage>());
    GenericArray.Add( FInstancedStruct::Make<FHitInfo>());

    FDamage& Damage = GenericArray[0].Get<FDamage>();
    FHitInfo& HitInfo = GenericArray[1].Get<FHitInfo>();
}
```

## `FStructArrayView`はビュー、`TArray<FInstancedStruct>`は実体
そもそも `FStructArrayView`は ビューであり、`TArray<FInstancedStruct>`は実体です。全然違います。

```cpp
UCLASS()
class UHoge: public UObject
{
    // ちゃんとシリアライズして保持できる!
    UPROPERTY() TArray<FInstancedStruct> Datas; 

    // コンパイルエラー. ビューを所有するんじゃない！
    UPROPERTY() FStructArrayView View; 
}
```
`FStructArrayView`は名前に`view`とあるように、一時オブジェクトとして扱うのがよいです。
`interface`や`Delegate`の引数・戻り値型に使うのがよいでしょう。

---
# `FConstStructArrayView`

`FConstStructArrayView`は `StructArrayView`の中身を書き換えられない`readonly-view`バージョンです。`FConstStructArrayView` からは `ConstStructView`が得られます。

そのほかは`StructArrayView`と同じです。

---

# `FInstancedStructContainer`
真の汎用配列です。このコンテナは、`TArray<FInstancedStruct>`の代替として、より高いパフォーマンスが求められる場面で使用されます。異なる型の構造体を連続したメモリブロックに確保します。これによりメモリアクセスの局所性が高まりキャッシュ効率が向上します。

要素の追加・削除はメモリ全体の再配置が必要になるためコストが高いです。
そのため、単一要素の`Add`はなく、一括追加の`Append`しかありません。
`ReserveBytes`であらかじめ必要なメモリ量を確保して、`Append`で要素を書きこむ、というものです。

## `FInstancedStructContainer`の メモリレイアウト
疑似コードです。
```cpp: 疑似コード
USTRUCT()
struct FInstancedStructContainer
{
    uint8* Memory = nullptr;
    int32 AllocatedSize = 0;
    int32 NumItems = 0;
}
```

`FInstancedStructContainer`は特徴的なメモリレイアウトとなっています。確保されたメモリブロック (`Memory`) の先頭からは、各構造体の実データが順番に配置されます。メモリブロックの末尾からは、各要素の型情報とオフセットを持つメタデータ`FItem`が逆順に配置されます。

データレイアウトは次の通りとなります。

```
[ StructA Data | StructB Data | ... | (未使用領域) | ... | Item 1 | Item 0 ]
^                                                                        ^
Memory                                              Memory + AllocatedSize
```

先頭から構造体`T0`があり、そのすぐ後ろに構造体`T1`が続きます。末尾には型情報とオフセットを保持した`FItem`構造体が格納されています。i番目の要素を取得するときは `FItem`を取得して型情報とメモリ位置を取得し、データ領域へのビューを返します。


## `FInstancedStructContainer` の作成と保持

通常は `UPROPERTY()`としてどこかに保持します。

```cpp: holder.h
UCLASS()
class UHolder :UObject
{
    UPROPERTY()
    FInstancedStructContainer Container;
}
```
ただの構造体なので普通にコンストラクタで作成できますし、デストラクタで破棄されます。

```cpp: constructor
void Main()
{
    {
        FInstancedStructContainer Container;
        Container.ReserveBytes(128); // MemAlloc
        Container.Append(...);
    }// ここでデストラクタが呼ばれてMemFreeされる
}
```
## `FInstancedStructContainer`の要素の追加
効率のため`ReserveBytes` でメモリ確保して`Append`で一気に要素を書き込みます。

`Append`は受け付ける型が非常に厳しく、`TConstArrayView<FInstancedStruct>` か `TConstArrayView<FConstStructView>`のいずれかです。
1つだけ`Append`したいときは長さ1の`TConstArrayView`に包んでください。

```cpp
USTRUCT()
struct FFoo32
{
    GENERATED_BODY()
    UPROPERTY() int32 Value{};
}
static_assert(sizeof(FFoo32) == 4);

USTRUCT()
struct FFoo64
{
    GENERATED_BODY()
    UPROPERTY() int32 Value0{};
    UPROPERTY() int32 Value1{};
}
static_assert(sizeof(FFoo32) == 8);

void WriteData()
{
    constexpr int BufferSize = sizeof(FFoo32) * 2 + sizeof(FFoo64)* 2 + FInstancedStructContainer::OverheadPerItem * 4;
    Container.ReserveBytes(BufferSize); // (4*2) + (8*2) + (16*4) = 88byte以上確保される(Alignment補正が入る)

    TArray<FInstancedStruct> InstancedStructArray =
        {
            FInstancedStruct::Make<FFoo32>(FFoo32{1}),
            FInstancedStruct::Make<FFoo64>(FFoo64{2, 3}),
            FInstancedStruct::Make<FFoo32>(FFoo32{4}),
            FInstancedStruct::Make<FFoo64>(FFoo64{5, 6})
        };
    TConstArrayView<FInstancedStruct> InstancedStructView(InstancedStructArray);
    Container.Append(InstancedStructView);
}
```
AppendはDeepCopyしますので、入力に与えたデータは破棄して構いません。

このときのデータレイアウトは大体次の通りとなります。(アラインメント次第)

```
[ FFoo32 | FFoo64 | FFoo32 | FFoo64 | Item3 | Item2 | Item 1 | Item 0 ]
^                                                                     ^
Memory                                                         Memory + 88 byte
```

今回は例として `FFoo32`, `FFoo64`, `FFoo32`, `FFoo64`と交互に配置してみました。
`ReserveBytes`していない場合は自動でメモリ確保されますが、バッファが不足するたびにリアロックと`Memmove`が繰りかえされるので、予約しておいた方が効率的です。

## `FInstancedStructContainer`の要素の削除
`RemoveAt`で削除します。削除したら隙間を埋めるべく要素の移動が行われます。メモリのリアロックは行いません。可能ならば末尾から削除すると要素の移動がなくて効率的です。

```cpp
Conatiner.RemoveAt(1); // FFoo64を削除
Conatiner.RemoveAt(2); // FFoo64が2番目に移動して来ているので削除
```
このときのデータレイアウトは次の通りとなります。

```
[ FFoo32 | FFoo32 | destructed | destructed | destructed | destructed | Item 1 | Item 0 ]
^                                                                                       ^
Memory                                                                      Memory + 88 byte
```
削除された要素はちゃんとその型のデストラクタが呼ばれます。


## `FInstancedStructContainer`の読み込みと走査
`operator[]`でアクセスできます。`ranged-based for` 対応しています。
メモリブロックの一部を指した`View`が返ってきます。

```cpp: for
void ScanData()
{
    // for loop
    for(int i=0; i < Container.Num(); ++i)
    {
        FStructView View = Container[i];

        // i番目に何が入っているかわからないときは都度チェック
        if ( FFoo32* Foo32 = View.GetPtr<FFoo32>())
        {
            Foo32->Value += i;	
        }
        if ( FFoo64* Foo64 = View.GetPtr<FFoo64>())
        {
            Foo64->Value0 += i;	
            Foo64->Value1 += i;	
        }
    }

    // range-based for
    for(FStructView View: Container)
    {
        // 同上
    }

    // const関数内 や const 修飾されているときのoperator[]はFConstStructViewを返す
    const FInstancedStructContainer& ConstContainer = Container;
    FConstStructView ConstView = ConstContainer[0];
}
```
## `FInstancedStructContainer` の破棄
`Reset()`, `Empty()`のいずれかで破棄したらよいです。
デストラクタでも完全に解放されるため、`UPROPERTY()`化して追跡しているならOwnerと寿命を同じくするでしょう。プールオブジェクトがContainerを維持し続けて実質メモリリークになるような場合は、明示的に`Empty()`で内部メモリを解放した方がいいケースもあります。

* `Reset`は中身はデストラクトしますがメモリは解放しません。
* `Empty`は中身をデストラクトしメモリを解放します。
* デストラクタは `Empty`と同じです。

```cpp
void ConsumeAndFlush()
{
    for(auto View : Container)
    {
    }
    // メモリは残しておいて次のAppendに備える
    Container.Reset();
}

void OnUnRegister()
{
    // メモリを解放する
    Container.Empty();
}
```


## `FInstancedStructContainer`シリアライズ対応
シリアライズ可能です。`UPROPERTY() FInstancedStructContainer` として保持しておけば保存されます。コンテナに格納した`UObject`への参照もGCに通知されます。

```cpp
UCLASS()
class UHoge : public UObject
{
    GENERATED_BODY();

private:
    //シリアライズされる
    UPROPERTY()
    FInstancedStructContainer Container; 
}
```

```cpp
template<>
struct TStructOpsTypeTraits<FInstancedStructContainer> : public TStructOpsTypeTraitsBase2<FInstancedStructContainer>
{
    enum
    {
        WithSerializer = true,
        WithIdentical = true,
        WithAddStructReferencedObjects = true,
        WithGetPreloadDependencies = true,
        WithExportTextItem = true,
        WithImportTextItem = true,
    };
};
```

## `FInstancedStructContainer`は UI非対応
カスタムDetails対応されていないため、Unreal Editor上ではセットしたりできません。
`UPROPERTY(EditAnywhere) FInstancedStructContainer;` としてもUIでいじれません。

もし UIサポートを受けたいならば、素直に `UPROPERTY(EditAnywhere) TArray<FInstancedStruct>` を使いましょう。


## `FInstancedStructContainer` vs `TArray<TVariant<T0, T1>>`

格納する型のサイズが大きく異なるならば、`FInstancedStructContainer`の方が有利です。
格納する型のサイズが概ね同じならば `TArray<TVariant>` の方がオーバーヘッドが小さく済む可能性が高いです。

`FInstancedStructContainer` は FItemというメタ情報をメモリ領域に所有するためオーバーヘッドが大きくなります。実際に格納できるデータ量が要素数に比例して少なくなります。

`TVariant` は静的型付けされた`union`のようなもので、最も大きな要素型のサイズとなります。固定のサイズであるため無駄な領域は発生しますが、格納する要素型のサイズが十分近しいのであるならば、その無駄を無視できることがあります。

例えば`FHitResult` は264byteもあるクソデカ構造体で、`FDamageEvent`は16byteです。`TVariant<FHitResult, FDamageEvent>` のサイズは`Max(264, 16)=264`byteとなります。
よって`TArray<TVariant<FHitResult, FDamageEvent>>` は 264*要素数の大きなデータを確保します。仮にHitResultを2個、DamageEventを3個追加したとすると `264*5=1320`byte確保されます。
一方で`FInstancedStructContainer` は `264*2 + 16 * 3 + 16 * 3 = 624`byteで済みます。

```cpp
void Main()
{
    // 固定サイズ要素型の配列
    {
        TVariant<FHitResult, FDamageEvent> VHitResult;
        TVariant<FHitResult, FDamageEvent> VDamageEvent;
        VHitResult.Emplace<FHitResult>(FHitResult{});
        VDamageEvent.Emplace<FDamageEvent>(FDamageEvent{});

        TArray<TVariant<FHitResult, FDamageEvent>> VariantArray;
        VariantArray.Reseve(5); // 264 * 5 byte
        VariantArray.Add(VHitResult);
        VariantArray.Add(VHitResult);
        VariantArray.Add(VDamageEvent);
        VariantArray.Add(VDamageEvent);
        VariantArray.Add(VDamageEvent);
    }

    // 可変サイズ要素型の配列
    {
        FHitResult HitResult{};
        FDamageEvent DamageEvent{};
        FConstStructView HitResultView = FConstStructView::Make(HitResult);
        FConstStructView DamageEventView = FConstStructView::Make(DamageEvent);
        
        TArray<FConstStructView> ConstStructViewArray;
        ConstStructViewArray.Add(HitResultView);
        ConstStructViewArray.Add(HitResultView);
        ConstStructViewArray.Add(DamageEventView);
        ConstStructViewArray.Add(DamageEventView);
        ConstStructViewArray.Add(DamageEventView);

        FInstancedStructContainer Container;
        Container.ReserveBytes(624); // 624byte
        Container.Append(ConstStructViewArray);
    }
}
```
どちらも連続したメモリブロックに確保されるためキャッシュ効率は概ね同じです。

---
# `StructUtil` の ユースケース

構造体の取り回しをよくする機能であるので、差さるところには非常に差さります。

## 使いどころ1: メッセージのPub/Sub
具象型が隠蔽できるため、任意の型をペイロードとして載せることが可能です。
Publisher側は`view`を使ってコピーレスにBroadcastすることができます。

```cpp: publisher.cpp
struct FPublisher
{
    FFoo Foo{};

    void Publish()
    {
        //スタック領域のインスタンスからviewを作って引き回せる
        {
            FFoo Foo;
            FStructView View(Foo::StaticStruct(), &Foo);

            FFoo& Foo2 = View.Get();
            ensure(&Foo == &Foo2);

            Delegate.BroadCast(View);
        } //Fooの寿命を迎えたのでメモリは解放される

        {
            // ヒープ領域のviewを作る
            TUniquePtr<FFoo> Foo = MakeUnique<FFoo>();
            FStructView View(Foo::StaticStruct(), Foo.Get());

            Delegate.BroadCast(View);
        }

        {
            // メンバーフィールドからビューを作る
            FStructView View(Foo::StaticStruct(), &this->Foo);
            Delegate.BroadCast(View);
        }
    }
}

```
リスナー側は汎用型のまま受信できます。ハンドルしたりコピーしたりできます。

```cpp: subscriber.cpp

UPROPERTY()
TQueue<FInstancedStruct, Mpsc> Queue;

// 別スレッドから受信する
void OnReceived_Concurrent(FConstStructView PayloadView)
{
    // FConstStructViewから FInstancedStructへのコピー
    // PayloadViewが指すメモリ領域の寿命が分からないのでコピーする
    Queue.Enqueue(PayloadView);

    //購読者が勝手に書き換えることは出来ないので安全
    // コンパイルエラー
    //PayloadView.GetMutable().Value = ...;
}

// メインスレで消費する
void Consume_GameThread()
{
    while(FInstancedStruct& Payload = Queue.Pop())
    {
        ...
    };
}
```

なんでもメッセージングシステム。
```cpp: MyMessageSubsystem
// MyMessageSubsystem.h

// 具象型に依存しないコンテナ型としてペイロードを定義できる
DECLARE_DELEGATE_OneParam(FOnMessageRecieved, const FInstancedStruct&);

// 型を絞りたいならベース型を指定してもよい
DECLARE_DELEGATE_OneParam(FOnMessageBaseRecieved, const TInstancedStruct<FMyMessageBase>&);

// 適当にマネージャークラスを用意
UCLASS()
class UMyMessageSubsystem : UWorldSubsystem
{
    GENERATED_BODY()

    FOnMessageRecieved Recieved;
}

// Subscriber.cpp
// 購読側は自身の興味のある具象型のみを知ればよい
#include "FMyMessage0.h"
#include "FMyMessage1.h"

UCLASS()
class ASubscriber : public AActor
{
    virtual void BeginPlay()
    {
        UMyMessageSubssytem* MessageSystem;
        MessageSystem->Recieved = [](const FInstancedStruct& Payload)
        {
            if(const FMyMessage0* Message0 = Payload.GetPtr<FMyMessage0>())
            {
                // 処理する
            }
            else if( const FMyMessage1* Message1 = Payload.GetPtr<FMyMessage1>())
            {
                // 処理する
            }
            else
            {
                // 未知のメッセージ. もしくは自身に関係ないメッセージ
            }
        };
    }
}
```

疎結合になり、柔軟性が増しました。型情報を使ってif分岐できます。`UScriptStruct*` のアドレスが一意であることを利用して`TMap<const UScriptStruct*, TValue>`に突っ込むことも可能です。

## 使いどころ2: プラグインやライブラリに最適

具象型を隠蔽できるということは、ユーザー側に型注入させることができるということです。

これはライブラリやプラグイン開発者にとっては可用性や柔軟性を大きく広げます。
プラグイン開発時点ではユーザー側のゲームコードに依存することはできません。なぜならば、まだその実装がないからです。
そこで、いくつかの対策が取られてきました。

* `IDataProvider` のように プラグイン側で`interface`を定義してユーザーに実装させる作戦
* `FCustomDataBase`のように プラグイン側でベースクラスを用意してユーザー側に継承してもらう作戦
* `int64`といった固定長のユーザーデータ型を用意する作戦
* `JSON`など構造化された文字列にしてパースする作戦

継承だと`virtual`関数の設計の仕方など対応しきれない部分がでたり、そもそもダイアモンド継承問題の危険があります。
固定長ユーザーデータは、特定の用途でサイズが不足していたり、使わない場合は無駄、といった問題があります。
文字列はデータ量増えるし、パースが遅くてダメです。

その点、`FInstancedStruct`なら、シリアライズによる永続化もできて、Replicationにも対応していて、エディタで簡単に触れて便利ですね。

実際に`StateTree`等のエンジンプラグインでは `FInstancedStruct`による具象型の隠蔽が行われています。
`StateTree`で自由にパラメータを設定できるのは `StructUtil`の機能のおかげなのです。

# まとめ

前編、中編、後編と長きにわたり解説しました。

## `StructUtil`が解決した課題

これまで具象型に依存しない汎用のデータホルダー実装手法として、`union`や `TVarint`がありました。しかしながら、弱点が沢山ありUnreal C++では滅多に使えませんでした。

  * 内部データ型を知るために`enum`や `FName`を使って`switch-case`する必要がある
  * コーディングをミスるとアクセス違反してしまう
  * 誤った型を入力することができて、コンパイル時に検知できない
  * メモリ確保量が型T1,T2...の中から最大のものとなる
  * **BP非対応**
  * **`UPROPERTY`非対応**

これらの問題を解決したのが `StructUtil`および`FInstancedStruct`です。

  * 静的型付け
  * 実行時型チェック
  * **BP対応**
  * **UPROPERTY対応**
    * シリアライズ
    * UPROPERTY修飾子
  * エディタ拡張対応
    * Detialsビューでstructを選択できる




