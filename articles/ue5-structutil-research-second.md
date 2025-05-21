---
title: "UE5:Unreal EngineのStructUtilについてまとめた 後編"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: false
---

# `FSharedStruct` と `FConstSharedStruct`

`FInstancedStruct` の メモリ領域を共有するバージョンです。
シンプルに`TSharedPtr<FInstancedStruct>`みたいな機能を持つ構造体です。ただし、単純にラップしてしまうとダブルポインタ操作になってしまうので、これを避けるために、`FStructSharedMemory`を利用して、共有メモリを直接保持しています。`FInstancedStruct`とは結構異なる実装になっています。

重要なのは FSharedStruct は 同一のメモリ領域を参照カウント方式で共有しているという点です。
FInstancedStruct をコピーするとメモリがDeepCopyされていましたが、こちらは同一の領域を共有します。
BP型のように事前に型を決定できない状況で共有できます。

`FSharedStruct`と `FConstSharedStruct`はメモリ領域を書き換えられるかどうかだけの違いです。
以降は `FSharedStruct` について解説します。

## `FSharedStruct`の作成

`Make`関数を使います。
```cpp
FSharedStruct SharedStruct = FSharedStruct::Make<FFoo>(42);
```

`InitializeAs`関数を使ってもいいです。
```cpp
FSharedStruct SharedStruct;
SharedStruct.InitializeAs<FFoo>();
```

## `FSharedStruct`の破棄

`Reset`するか、デストラクタで参照カウントが減ります。
```cpp
void Main()
{
    // 参照カウント1
    FSharedStruct SharedStruct = FSharedStruct::Make<FFoo>(42);

    // 参照カウントが2に増える
    FSharedStruct Shared2 = SharedStruct;

    {
        // 参照カウントが3に増える
        FSharedStruct Shared3 = SharedStruct;
    } // 参照カウントが2に減る

    Shared2.Reset(); // 参照カウントが1に減る

}// 参照カウントが0になりFreeされる
```

## `FSharedStruct`の読み書き

`Get<T>` か `GetPtr<T>` で得ます。

`TSharedPtr<T>`と異なりポインタのように振る舞う型ではないので、`operator->`はありません。

```cpp
void Main()
{
    FSharedStruct SharedStruct = FSharedStruct::Make<FFoo>(42);
    FFoo& Foo = SharedStruct.Get<FFoo>();
    Foo.Value = 43;

    FFoo* Foo = SharedStruct.GetPtr<FFoo>();
    Foo->Value = 44;
}

```

## `FSharedStruct`のハンドリング

共有参照が必要なときは`const&`渡しがお勧めです。値渡しの場合は一時オブジェクトにより参照カウントが増えますのでオーバーヘッドが無駄です。
```cpp
FSharedStruct SharedData;

void RecieveData_Good(const FSharedStruct& InSharedData)
{
    // 共有したり
    this->SharedData = InSharedData;
}

// 引数に積まれるときに参照カウントが1増える
void RecieveData_Bad(FSharedStruct InSharedData){}
```

値を使いたいだけならば、 `FSharedStruct`/ `FConstSharedStruct`に変換して渡せます。
メモリの所有権が不要ならば、こちらでよいでしょう。Delegate型の種類を減らしたり、同じシグネチャの関数に渡せたり結構便利です。

```cpp
FSharedStruct SharedData;

void SimpleUseData(FStructView View)
{
    T& Data = View.Get<T>();
}

void Main()
{
    SimpleUseData(FStructView::Make(this->SharedData));
}
```



## `FSharedStruct`は UPROPERTY対応

`TSharedPtr<T>`はダメなのに、 それをラップする`FSharedStruct`は`UPROEPRTY`対応です。
何気にこれは凄いことです。

```cpp
UCLASS()
class UHoge : public UObject
{
    GENERATED_BODY()

private:
    UPROPERTY()
    FSharedStruct SharedData;

    UPROPERTY()
    FConstSharedStruct ConstSharedData;
}
```

なお`EditAnywhere`を付けたところで Detailsビューでは編集できませんでした。シリアライズされて保持される、ということでしょう。

## `FSharedStruct`は BPだめ

ダメでした。
`Error  : Type 'FSharedStruct' is not supported by blueprint.`

# `FSharedStruct` 詳解

疑似コードです。省略すると以下の通りになります。
```cpp
struct FSharedStruct
{
    TSharedPtr<FStructSharedMemory> Memory;
}
```
ただのラッパーですね。本体は`FStructSharedMemory`です。

`FStructSharedMemory`の疑似コードです。
```cpp
struct FStructSharedMemory
{
    TObjectPtr<const UScriptStruct> ScriptStruct;
    uint8 Memory[0];
}
```
型情報とメモリ領域をもっており、`FInstancedStruct`と同じ感じです。
ですが、長さ0の配列を持っていますね。キモイですね。

これは`Flexible array member`パターンです。C言語ではC99でサポートされていますが、C++ではサポートされてたかよくわかりませんでした。

フレキシブル配列ですが、次のような雰囲気のメモリ確保します。
理解を助けるために簡単化した疑似コードです。

```cpp
//こんな感じ
const int32 ToalMemSize = sizeof(FStructSharedMemory) + sizeof(T);
uint8* AllocatedMemory = Malloc(ToalMemSize);
uint8* PointerAreaMemory = AllocatedMemory;
new (PointerAreaMemory) FStructSharedMemory();

FStructSharedMemory* SharedMem = reinterpret_cast<FStructSharedMemory>(PointerAreaMemory);
uint8* StructAreaMemory = SharedMem->Memory;
new (StructAreaMemory) T();
TSharedPtr<FStructSharedMemory> Ptr = MakeSharable<FStructSharedMemory>(AllocatedMemory, CustomDeleter);
```

順番に解説します。
重要な点は どかんと連続した領域にメモリ確保していることです。
```
const int32 ToalMemSize = sizeof(FStructSharedMemory) + sizeof(T);
uint8* AllocatedMemory = Malloc(ToalMemSize);
```

このように、8byte + 型Tの分確保します。図では32byte確保してます。
(パケット図は本当はbit表記だけどbyteで読んでください)

```mermaid
packet-beta
title AllocatedMemory
0-31: "未初期化領域"
```

連続したメモリ領域に確保することでダブルポインタによる2連続fetchを回避します。キャッシュ効率があがるはずです。

次に`placement new` でメモリ領域にオブジェクトを構築していきます。まずは`FStructSharedMemory`部分を構築します。
先頭アドレスから `sizeof(FStructSharedMemory)` 分構築します。
`sizeof(FStructSharedMemory)` = 8+0 =8byteです。`TObjectPtr`1つ8byte、長さ0の配列は0byteです。


```cpp
uint8* PointerAreaMemory = AllocatedMemory;
new (PointerAreaMemory) FStructSharedMemory();
```

```mermaid
packet-beta
title AllocatedMemory
0-7: "ScriptStruct*"
8-31: "未初期化領域"
```

次にデータ領域を構築していきます。フレキシブル配列メンバーの場合、`FStructSharedMemory::Memory[0]`はoffsetされた位置を差しています。
今回の場合はまさに 8byte分進んだ位置を差しています。そこに T型で構築します。

```cpp
FStructSharedMemory* SharedMem = reinterpret_cast<FStructSharedMemory>(PointerAreaMemory);
uint8* StructAreaMemory = SharedMem->Memory;
new (StructAreaMemory) T();
```

```mermaid
packet-beta
title AllocatedMemory
0-7: "ScriptStruct*"
8-31: "struct T"
```

上記手順により `AllcatedMemory`は無事構築されました。

あとは`TSharedPtr`に渡すだけです。今回は特殊な初期化をしてしまったので、単純なdelete では解放できません。正しく実装したカスタムデリータを渡します。

```cpp
TSharedPtr<FStructSharedMemory> Ptr = MakeSharable<FStructSharedMemory>(AllocatedMemory, CustomDeleter);
```

```mermaid
packet-beta
title FSharedStruct
0-7: "Ptr"
8-15: "ReferenceCount"
16-24: "WeakCount"
```

↑上図の`FSharedStruct::TSharedPtr<T>::Ptr`の部分が ↓下図のオブジェクト先頭を指しています。

```mermaid
packet-beta
title AllocatedMemory
0-7: "ScriptStruct"
8-31: "struct T"
```

# 共有ポインタ対応表

以下の通り対応表が出揃いました。

|ベース型 | 共有コンテナ型 | 
| --- | --- |
| `UObject*`型 | `TStrongObjectPtr<UObject>` |
| `USTRUCT*`型 | `FSharedStruct` |
| `Native*`型  | `TSharedPtr<FNative>` |

`UObject*` に対する参照カウント方式共有参照は `TStrongObjectPtr<T>`、
`USTRUCT*` に対する参照カウント方式共有参照は `FSharedStruct`、
`Native`型に対する参照カウント方式共有参照は `TSharedPtr<T>`です。


---

# 以下したがき

ライブラリ側で使います。
任意のユーザー型`T`を`FInstancedStruct`で曖昧に受けたとき、`FStructView`や `FConstStructView`を利用して曖昧型のままハンドリングできます。

`FStructView`を使わずに`FInstancedStruct&`で引き回すしたらいいと思われるかもしれません。
しかし、メモリの所有権を持たない、ということが重要に思います。`FInstancedStruct&`は `InitializeAs<T>`でメモリを再確保できてしまいます。
失敗できるということは誰かがバグらせる余地があるということです。マーフィーの法則にしたがえば、必ずバグります。
そもそも間違えられないように作るのが理想ですから、FStructViewを使って、メモリのR/Wのみ許可してAlloc/Freeはできないようにすると良いです。

## 頑健なアクセス制御
TConstInstancedStruct のように便利ラッパーが存在します。
readonly アクセスのみ公開したいようなケースで有効です。

TConstSharedStruct や TSharedStructのように具象型を隠蔽しつつ、共有制御もできます。

## `StructUtil`が解決した課題

これまで具象型に依存しない汎用型のデータホルダー実装そちて、`union`や `TVarint`がありました。
しかしながら、弱点が沢山あり滅多に使えませんでした。

  * 内部データ型を知るために`enum`や `FName`を使って`switch-case`する必要がある
  * コーディングをミスるとアクセス違反してしまう
  * 誤った型を入力することができて、コンパイル時に検知できない
  * メンテンスが大変
  * 型T1,T2...の中から`sizeof(T)`が最大の分だけ確保しなくてはならずメモリが無駄
  * **BP非対応**
  * `UPROPERTY`非対応

これらの問題を解決したのが `FInstancedStruct`です。

  * 静的型付け
  * 実行時型チェック
  * BP対応
  * エディタ拡張対応
    * Detialsビューでstructを選択できる
  * UPROPERTY対応
    * シリアライズ
    * UPROPERTY修飾子

至せり尽くせりです。実際に使用して評価してみましたが、弱点が見当たりません。


# `StructUtil` の 使いどころ

小さな構造体の取り回しをよくする機能であるので、差さるところには非常に差さります。

## メッセージのPub/Subに最適
具象型が隠蔽できるため、任意の型をペイロードとして載せることが可能です。

```cpp: publisher.cpp
void Publisher()
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
        UFoo* Obj = NewObject<UFoo>();
        FStructView View(Foo::StaticStruct(), &Obj->Foo);
        Delegate.BroadCast(View);
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


```cpp
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

疎結合になり、柔軟性が増しました。型情報を使ってif分岐できますから、`enum`や`FName`を使う必要はありません。
`UScriptStruct*` のアドレスが一意であることを利用して`TMap<const UScriptStruct*, TValue>`に突っ込むことも可能です。
(個人的にアドレスの値を利用するのは邪悪だと思っていますが)

## プラグインやライブラリに最適

具象型を隠蔽できるということは、ユーザー側に型注入させることができるということです。

これはライブラリやプラグイン開発者にとっては可用性やユーザーカスタムの柔軟性を大きく広げます。
実際に`StateTree`等のエンジンプラグインでは `FInstancedStruct`による具象型の隠蔽が行われています。

プラグイン開発時点ではユーザー側のゲームコードに依存することはできません。なぜならば、まだその実装がないからです。
そこで、いくつかの対策が取られてきました。

* `IDataProvider` のように プラグイン側で`interface`を定義してユーザーに実装させる作戦
* `FCustomDataBase`のように プラグイン側でベースクラスを用意してユーザー側に継承してもらう作戦
* `int64`や`void*`といった固定長のユーザーデータ型を用意する作戦
* `JSON`など構造化された文字列にしてパースする作戦

継承だとvirtual関数の設計の仕方など対応しきれない部分がでたり、そもそもダイアモンド継承問題の危険があります。
固定長ユーザーデータ型は、特定の用途ではサイズが不足していたり、使わない場合は無駄だったり、といった問題があります。
文字列はデータ量増えるし、パースが遅いのもダメです。

その点、`FInstancedStruct`なら、シリアライズによる永続化もできて、Replicationにも対応していて、エディタで簡単に触れますから便利ですね。