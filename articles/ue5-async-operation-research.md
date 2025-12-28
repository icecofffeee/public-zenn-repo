---
title: "UE5:Unreal Engineの非同期APIについての研究"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5,cpp,unrealengine,unrealengine5]
published: false
---
# はじめに
Unreal Engine 5 の 非同期APIについての研究報告です。

主にC++での非同期APIについての研究です。
もっぱら`TaskSystem` についてです。`Latent`ノードについては触れません。

# モチベーション
ことのおこりは `DataRegistry` で非同期ロードしたあとに `TableRow`から得た `TSoftObjectPtr` を更に非同期ロードするという二段階非同期ロードを試みたことです。

* 非同期ロードして非同期処理したい
* `async-await`な感じで流れるように書きたい
* コールバックのネスト地獄を回避したい

特にスレッドをゲームからワーカーへワーカーからゲームへと柔軟に切り替える感じの非同期処理を実装したい、という気持ちからモダンなUnreal C++の書き方を調べたのですがインターネッツ上にはなかなか情報が見つかりません。AIに聞いてもハルシネーションを見抜ける知識がないのでよくわかりませんでした。模索しました。


# 検証環境
* UE 5.7.1 ソースビルド
* C++ 20
* MSVC 14.44

UE5.5.4でも確認済みです。

# 非同期機構一覧
Unreal Engine 5.5.4時点では以下の非同期に扱えそうなAPIが存在します。

* TaskGraph
* Task System
* TFuture-TPromise
* Async
* FQueuedThreadPool
* FRunnable

なにが違ってどれを使えばいいんでしょうか。

# 結論: Task Systemの勝ち
`UE::Tasks` 名前空間内の `Task System`を使うのが一番汎用性が高く現実的でした。

# 研究概要
非同期のユースケースとしては、
* セーブシステム:
  * セーブデータの収集・圧縮・ファイルI/O・展開・反映処理を非同期かつフレーム分散する
* 通信周り:
  * 認証・Lobby取得・Join/Leave・RPCや同報といった順序を持った非同期通信の連鎖
* 遅延ロードと遅延初期化:
  * ActorやAssetを遅延ロードして非同期で待ち受けてから初期化等の後続処理を実行する

が思いつきますね。これらのユースケースに幅広く対応できて、使いやすく、書き味がよく、デッドロックしにくい、非同期機構の勝利です。

本稿ではサンプル事例として`TSoftObjectPtr`の非同期ロードを行いその中身を利用する、というあるあるな事例を取りあげます。アセットを非同期ロードしてロード完了後にそのアセットを使う、というサンプルを複数の非同期機構を使って実装していきます。

以降のコードではサンプルコードを簡潔に保つため`GENERATED_BODY`などの分かり切った処理は割愛します。悪しからず。

---

# 1. 基本のコールバック式
まずは古くから存在し広く使われているコールバック方式をおさらいします。

`TSoftObjectPtr`からの非同期ロードは`FStreamableManager` 経由で実行します。ロード完了後の`continuation`はコールバックを渡します。

```cpp: callback
TSoftObjectPtr<UFooAsset> SoftPtr;
FSoftObjectPath Path = SoftPtr.ToSoftObjectPath();
FStreamableManager& StreamableManager = UAssetManager::GetStreamableManager();
StreamableManager.RequestAsyncLoad(Path, [SoftPtr]()
{
    if(SoftPtr.IsValid())
    {
        UFooAsset* LoadedAsset = SoftPtr.Get();
        // ここに ロードされたアセットを使ってさらに処理
    }
});
```
ロード後に追加の処理を実装したいときはこのコールバックからDelegateを呼ぶかさらに非同期APIにコールバックを渡すことでチェインを張ります。コールバックチェインです。

```cpp: コールバックチェイン
StreamableManager.RequestAsyncLoad(Path, [SoftPtr]()
{
    UFooAsset* LoadedAsset = SoftPtr.Get();
    DoAsync1(LoadedAsset, []()
    {
        DoAsync2([]()
        {
            DoAsync3([]()
            {
                //...以下無限にコールバックをネストしていく
            })
        });
    });
});

```

これは正直つらいので、チェインが簡単に張れるようにラップしてみましょう。

```cpp: コールバックスタイル
template<TAsset>
static void LoadAssetAsync<TAsset>(TSoftObjecPtr<TAsset> Ptr, TFunction<TAsset*>&& Completion)
{
    // 即時リターン:無効な参照
    if(SoftPtr.IsNull())
    {
        if(Completion){Completion(nullptr);}
    }

    // 即時リターン:ロード済みパターン
    if(SoftPtr.IsValid())
    {
        if(Completion){Completion(SoftPtr.Get());}
    }

    FSoftObjectPath Path = SoftPtr.ToSoftObjectPath();
    FStreamableManager& StreamableManager = UAssetManager::GetStreamableManager();
    // MoveTempで転送キャプチャする
    StreamableManager.RequestAsyncLoad(Path, [SoftPtr, Completion =MoveTemp(Completion)]()
    {
        if(Completion)
        {
            Completion(SoftPtr.Get());
        }
    });
}
```
多少便利になるようにラップできたぞ！よし使ってみよう！

```cpp: コールバック地獄
void BeginPlay()
{
    // Aをロードして成功したらBをロードして成功したらCをロードする例
    LoadAssetAsync<UMyDataAsset>(SoftPtr, [](UMyDataAsset* Asset)
    {
        LoadAssetAsync<UMyChidAsset>(Asset->SoftPtr, [](UMyChidAsset* Child)
        {
            LoadAssetAsync<UMyChildChidAsset>(Child->SoftPtr, [](UMyChildChidAsset* ChildChild)
            {
                //
            });
        });
    })
}
```

何か便利になったかというと、何も便利になっていません。即時パスやエラーハンドリングが付いたぐらいです。これはダメですね。

ラムダ式を MoveTempしたりと面倒くさいですし、結局コールバックを使った時点でコールバックを次々とネストさせるしかありません。なんとかしたいです。

# 2. `TFurure-TPromise` パターン

Unreal Engine には `TFuture-TPromise`が実装されています。こちらは`std::future/std::promise`に相当する機能です。ということは 典型的な `future-promise`パターンでcontinuation処理を実装できそうです。
`future`と異なり`TFuture`は continuation処理に対応しており`Then()`と`Next()`が実装されています。先ほどの非同期ロードメソッドが `TFuture`を返すようにしてみます。

:::message
`Then`は後続処理の本体でありますが、値変換するシンタックスシュガー的な関数である`Next`の方が便利です。`Next`を推奨します。
:::

```cpp: future版
/**
* Futureを返す LoadAsset関数
*/
template<typename TAsset>
static TFuture<TAsset*> LoadAssetAsyncFuture<TAsset>(TSoftObjecPtr<TAsset> Ptr)
{
    TSharedRef<TPromise<TAsset*>> Promise = MakeShared<TPromise<TAsset*>>();
    TFuture<TAsset*> Future = Promise->GetFuture();

    // 即時リターン:無効な参照
    if(SoftPtr.IsNull())
    {
        Promise->SetValue(nullptr)
        return Future;
    }

    // 即時リターン:ロード済みパターン
    if(SoftPtr.IsValid())
    {
        Promise->SetValue(SoftPtr.Get())
        return Future;
    }

    // Promiseの共有参照をコピーキャプチャする
    FSoftObjectPath Path = SoftPtr.ToSoftObjectPath();
    FStreamableManager& StreamableManager = UAssetManager::GetStreamableManager();
    StreamableManager.RequestAsyncLoad(Path, [SoftPtr, Promise]()
    {
        Promise->SetValue(SoftPtr.Get());
    });

    return Future;
}
```

`Promise`を共有参照で作成し`RequestAsyncLoad`のコールバックに渡しています。
これによってコールバックを`TFuture`で包むことができました。

## 解説
こちらの実装はUnreal C++に対する非常に多くの学びがありました。
`FStreamableManager::RequestAsyncLoad`は `FStreamableDelegate`を要求します。これはUEの標準のDelegateなのですが、コピー可能である必要があります。ムーブキャプチャを伴うラムダ式はコピー不可になるため`FStreamableDelegate`にできません。そのため、`TPromise`をムーブキャプチャじゃなくてコピーキャプチャしたいのですが、`TPromise`は`UnCopyable`を継承しておりコピー不可です。
つまり、`TPromise`は`RequestAsyncLoad`に対してコピー渡しもムーブ渡しもできなくて詰んでいることがわかりました。

そこで`TPromise`を`TSharedRef`に包むことでコピーキャプチャできるようにします。
```cpp
    TSharedRef<TPromise<TAsset*>> Promise = MakeShared<TPromise<TAsset*>>();
```
`FDelegate`などにpromiseを渡すときは`TSharedRef`に包むということですね。

:::message
約束はコピーできない
:::

:::message
`TUniqueFunction`のような型をコールバック型として受け取るAPIなら`MoveTemp`で転送可能です
:::

## 使い方
ラップした関数の使い方は次の通りです。

```cpp
UPROPERTY(EditDefaultOnly)
TSoftObject<UDataAsset> DataAssetRef;

void AActor::BeginPlay()
{
    TFuture<UDataAsset*> Future = LoadAssetAsyncFuture(DataAssetRef);
    Future.Next([](UDataAsset* DataAsset)
    {
        // StremableManagerのコールバックはゲームスレッドなのでここもゲームスレッド
        if(IsValid(DataAsset))
        {
            UE_LOG(LogTemp, Log, TEXT("ロードできたよ"));
        }
    });
}
```
`TFuture::Next()` を使うことで継続処理`Continuation`を実装することができます。

`Next`に与えるラムダ式は `TFuture<TResult>` の場合 `TFunction<void(TResult)>`となります。
今回はロード結果を`Next`に与えるラムダ式の引数から得られます。

```cpp
UCLASS()
class UMyDataAsset : public UDataAsset
{
public:
    UPROPERTY(EditDefaultOnly) int32 Value;
    UPROPERTY(EditDefaultOnly) int32 Value2;
}

// UMyDataAssetが差さっているとする
TSoftObject<UDataAsset*> DataAssetRef;

void AMyActor::BeginPlay()
{
    // メソッドチェーンで書き下せる
    LoadAssetAsyncFuture(DataAssetRef)
    .Next([](UDataAsset* Asset) -> UMyDataAsset*
    {
        return Cast<UMyDataAsset*>(Asset);
    })
    .Next([](UMyDataAsset* MyData) -> int32
    {
        return IsValid(MyData) ? MyData->Value : 0;
    })
    .Next([](int32 Value)
    {
        UE_LOG(LogTemp, Log, TEXT("%d"), Value);
    });
}
```
このようにネストなしにNextを連続して付与することができます。ラムダ式の返り値型は推論してくれるのですが、皆様の読みやすさのために明示的に指定しております。前のNextに与えるラムダ式の返り値型が次のNextに与えるラムダ式の引数になります。

`Future-Promise`パターンの強みであるflatなコールバックが実装できています。
なんだかjavascriptっぽくなってきましたね。これは期待できます。

## 結局使えない future-promiseパターン

が、しかし何も便利になっていません。
上記サンプルでは単純に同期処理しかしていません。だったら一つの関数にに書き下せばいい話ですし、そちらの方が保守性も高いでしょう。

`future`でチェインしたいのはさらなる非同期処理です。
`DataAsset` の中のソフト参照を更にロードしてみます。

```cpp: load chain
UCLASS()
class UMyDataAsset : public UDataAsset
{
public:
    UPROPERTY(EditDefaultOnly)
    TSoftObjectPtr<UChildDataAsset> ChildAsset;
}

UPROPERTY(EditDefaultOnly)
TSoftObject<UMyDataAsset*> MyDataAssetPtr;

void AActor::BeginPlay()
{
    LoadAssetAsyncFuture(MyDataAssetPtr)
    .Next([](UMyDataAsset* Asset)
    {
        if(!IsValid(Asset)){ return; }

        // 更に子をロード
        LoadAssetAsyncFuture(Asset->ChildAsset);
        .Next([](UFooDataAsset* Child)
        {
            if(IsValid(Child)){ Child->GetHogeValue();}
        });

    });
}
```
`Next`がネストしている！ぐわぁー！

Aをロードしてそのロード結果からBをロードするような非同期に非同期を連ねた処理がフラットに書けませんでした。

javascriptの`unwrap`のような便利関数はありません。
無理矢理`Next`をフラットに書くことも可能です。ただしその場合は `TFuture<TFuture<TResult>>` 型となるので `Next`に入るのは `TFuture<TResult>`です。`TFuture<TResult>`からは`GetResult()`で結果を待ち受けて取得できます。


```cpp: デッドロック
void AActor::BeginPlay()
{
    // 例としてBeginPlay(ゲームスレッド)を開始地点とする
    LoadAssetAsyncFuture(MyDataAssetPtr)
    .Next([](UMyDataAsset* Asset) -> UFooDataAsset*
    {
        return IsValid(Asset) ? Asset->ChildAsset : nullptr;
    })
    .Next([](TFuture<UFooDataAsset* Child> FutureResult)
    {
        // デッドロック！
        UFooDataAsset* Result = FutureResult.GetResult();
    });
}
```

残念ながらデッドロックします。
原因は `BeginPlay`ゲームスレッドで開始したロード処理をゲームスレッドで`GetResult`しているからです。`FStreamableManager::RequestAsyncLoad`はゲームスレッドでコールバックを返します。
その結果 `Promise::SetValue`もゲームスレッドで実行され、上記`Next`に与えたコールバックもゲームスレッドで呼ばれます。
一方で、`Future::GetResult`は呼び出しスレッドをブロックして`promise`を待ち受けるメソッドです。`GetResult`をゲームスレッドで呼ぶということはゲームスレッドをブロックするということです。結果、`FStreamableManager`の完了コールバックが動かないので、永遠に待ち続けます。

この問題が難しいのは `SoftObjectPtr`がロード済みであったパターンです。
`LoadAssetAsyncFuture`はロード済みの場合は即時`return` するのでロードしません。
よって`FutureResult`には有効な値が格納されているので`GetResult`もすぐに制御を返します。

たまにデッドロックするけどしないときもある、という非同期処理の典型的なバグを踏みました。

Workerスレッドに投げるなどの回避策を取ることは可能ですが、そのあとゲームスレッドに戻す必要があることがほとんどでしょう。
それをやるぐらいなら`Next`をネストさせた方がマシです。
```cpp: スレッドを行き来する
    LoadAssetAsyncFuture(MyDataAssetPtr)
    .Next([](UMyDataAsset* Asset) -> UFooDataAsset*
    {
        return IsValid(Asset) ? Asset->ChildAsset : nullptr;
    })
    .Next([](TFuture<UFooDataAsset* Child> FutureResult)
    {
        AsyncTask(WorkerThread, []()
        {
            // 待ち受けスレッドを止めて待つ！
            UFooDataAsset* Result = FutureResult.GetResult();

            // UObjectにワーカースレッドで触るのはよくない
            Result->GetValue();

            // ゲームスレッドに返す
            AsyncTask(GameThread, [Strong = TStrongObjectPtr(Result)]()
            {
                // ゲームスレッドに帰ってきた
                Strong.Get()->GetValue();
            });
        });
    });

```

何をしても結局ネストしまくりなので、あまり利点を感じません。
根本的な原因は`FStreamableManager`のコールバックがゲームスレッド指定であることにあります。それ自体は`UObject`を触るので当然の仕様なのですが、`Future-Promise` にはスレッド切り替えの機能がないが故に使い勝手が悪いのです。

そこで一部処理をワーカースレッドに投げてみましょう。

# `Future-Async`

こちらはpure C++の `std::async`に該当する機能です。`Async`は`Future-Promise`にスレッド選択機能が拡張されたもので、渡した関数を指定のスレッドで実行してくれる機能です。`caller`はその結果を `TFuture`経由で待ち受けることができます。

```cpp: async
void WaitInThread()
{
    using FResult = TSoftObjectPtr<UMyCharacterData>;
    TFuture<FResult> Future = LoadAsyncFuture<UMyCharacterData>(CharacterData);

    // 非同期ロードをワーカースレッドで待ち受けてみる
    Async(EAsyncExecution::Thread, [Future]()
    {
        while(true)
        {
            // このスレッドを最大100msブロックして待つ. 贅沢な使い方なのでよくない
            if (Future.WaitFor(FTimespan::FromMilliseconds(100)))
            {
                // ロード成功. ただしワーカースレッドなのでUObjectへの読み書きには注意
                FResult Result = Future.Get();

                // UObjectアクセスするために、ここからゲームスレに戻したい
                // TimerManagerとか使うの？Async使えないの？
            }
            else
            {
                // タイムアウト
                // タイムアウト時のリトライ機構やエラーハンドル機構がない！
            }
        }
    });
}
```

:::message
`while(true)`や `WaitFor`で待つのは極悪です。説明のための疑似コードなのでコピペしないように
:::

第1引数に`EAsyncExection::TaskGraph`を指定すると内部で`TaskGraph`を利用します。`Thread`を指定するとワーカースレッドで動かしてくれます。

`TPromise`を直接使うよりもはるかに簡単にスレッド処理を実装できました。しかしながら、結局`TFuture`が返ってくるので`Next`でチェインさせることは難しいです。
今回はアセットロードであったため、`Async`の威力が発揮できませんでした。
`Future-Promise`と違って `Future-Async`はラムダ式の実行スレッドを選択できるのですから、ゲームスレッド以外に任せたい処理を書くべきです。題材が悪かったので別の例で考えてみましょう。

疑似コードです。
```cpp: スクショとるasync
void TakeScreenShotAsync()
{
    // 1.スクショを取って
    // 2.PNG圧縮して
    // 3.slackへ投稿
    // 4.更にローカル保存しておく
    auto Future0 = Async(EAsyncExecution::Thread, []()
    {
        auto ScreenShot = TakeScreenShot();

        Async(EAsyncExecution::Thread, [ScreenShot]()
        {
            auto PNG = CompressPNG(スクショ画像テクスチャ);
            Async(EAsyncExecution::Thread, [PNG]()
            {
                // 必要ならさらにJsonシリアライズして...という非同期処理が入る
                auto FutureSlack = Network::SendPNGToSlackAsync("https://<path/to/slack/api...>", PNG);
                // FutureSlack は FireAndForgetで捨ててよいとする
                
                Async(EAsyncExecution::Thread, [PNG]()
                {
                    // ローカルドライブにPNG保存する
                    MyFileSystem::OpenWrite("./hoge.png", PNG);
                });

            });
        });
    });
}
```
やっぱりネストします。`Future`を返すので結局ネストします。まぁまぁやりたいことは書き下せましたし、スレッドが選べるようになったけど、ネスト地獄が始まります。

もうちょっとマシになりませんか？

# Task System
非同期処理の決定版です。`UE::Tasks`を使うべきでしょう。`Task System`はタスクのネスト、事前条件を設定できるのタスクチェインを簡単に設定できます。ネスト地獄からおさらばです。

上記例を Task Systemバージョンに書き直します。

```cpp: Tasys スタイル
#include "Tasks/Task.h"

template<typename TAsset>
static UE::Tasks::TTask<TAsset*> LoadAssetAsyncTask<TAsset>(TSoftObjecPtr<TAsset> Ptr)
{
    // 即時リターン:無効な参照
    if(SoftPtr.IsNull())
    {
        return UE::Tasks::MakeCompletedTask<TAsset*>(nullptr);
    }

    // 即時リターン:ロード済みパターン
    if(SoftPtr.IsValid())
    {
        return UE::Tasks::MakeCompletedTask<TAsset*>(SoftPtr.Get());
    }

    // FTaskEventの寿命を延ばすために共有状態を作る
    struct FState
    {
        explicit FState(const TCHAR* DebugName):Event(DebugName){}

        void SetResult(TAsset* InResult)
        {
            Result = InResult;
            Event.Trigger();
        }

        TAsset* GetResult() const { return Result.Get(); }

        UE::Tasks::FTaskEvent Event;
        TWeakObjectPtr<TAsset> Result { nullptr };
    };
    const TSharedRef<FState> State = MakeShared<FState>(UE_SOURCE_LOCATION);

    // Stateの共有参照をコピーキャプチャしてロード結果をResultにセットする
    FSoftObjectPath Path = SoftPtr.ToSoftObjectPath();
    FStreamableManager& StreamableManager = UAssetManager::GetStreamableManager();
    StreamableManager.RequestAsyncLoad(Path, [SoftPtr, State]()
    {
        State->SetResult(SoftPtr.Get());
    });

    // Eventのトリガーを待つタスクを返す
    UE::Tasks::TTask<TAsset*> Task = UE::Tasks::Launch(UE_SOURCE_LOCATION,
        [State](){ return State->GetResult(); },
        UE::Tasks::Prerequisites(State->Event));

    return Task;
}
```
なんだか`TaskCompletionSource`みたいになってきました。`FTaskEvent` がまさにそれに該当します。

## 解説
`FTaskEvent`に依存するタスクを生成してそれを `return` しています。
アセットロード完了後に`Trigger`されることで、ロード結果をタスクの`Result`として返すことができます。

使い方は以下の通りです。
```cpp: 使い方
UPROPERTY(EditDefaultOnly)
TSoftObjectPtr<UMyDataAsset> MyDataAssetPtr;

void AActor::BeginPlay()
{
    using namespace UE::Tasks;

    // TaskA->TaskB という順で非同期に順番で実行する

    TTask<UMyDataAsset*> TaskA = LoadAssetAsyncTask(this->MyDataAssetPtr);

    TTask<UChildDataAsset*> TaskB = Launch(UE_SOURCE_LOCATION, [TaskA]() mutable -> UChildDataAsset*
    {
        //Prerequisites指定により、この時点でTaskAが完了しているのは確実なのでGetResultしてよい
        UMyDataAsset* Asset = TaskA.GetResult();
        return IsValid(Asset) ? Asset->ChildAsset : nullptr;
    },
    Prerequisites(TaskA), // TaskAが完了してからこのタスクを実行する
    ETaskPriority::Normal,
    EExtendedTaskPriority::GameThreadNormalPri); // <-ここ重要
}
```

TaskBはUObjectを触りたいのでゲームスレッドを指定しています。`EExtendedTaskPriority::GameThreadNormalPri`でゲームスレッドを名指しできます。`Task`の仕組みに乗りながらワーカースレッドやゲームスレッドを自由に行き来できます。いいじゃん。

さらに連鎖させましょう。

```cpp
void AActor::BeginPlay()
{
    using namespace UE::Tasks;

    // A->B->C という順で非同期に順番で実行する

    TTask<UMyDataAsset*> TaskA = LoadAssetAsyncTask(this->MyDataAssetPtr);

    TTask<UChildDataAsset*> TaskB = Launch(UE_SOURCE_LOCATION, [TaskA]() mutable -> UChildDataAsset*
    {
        UMyDataAsset* Asset = TaskA.GetResult();
        auto SubTask = LoadAssetAsyncTask(Asset->ChildAssetPtr);
        AddNested(SubTask);
        return Asset->ChildAsset;
    },
    Prerequisites(TaskA), // TaskAが完了してから実行する
    ETaskPriority::Normal,
    EExtendedTaskPriority::GameThreadNormalPri); // <-ここ重要

    FTask TaskC = Launch(UE_SOURCE_LOCATION, [TaskB]() mutable -> void
    {
        UChildDataAsset* Child = TaskB.GetResult();
        Child->GetHoge();
    },
    Prerequisites(TaskB), // TaskBが完了してから実行する
    ETaskPriority::Normal,
    EExtendedTaskPriority::GameThreadNormalPri);
}
```

フラットなままタスクを書き下すことができました。
最大のポイントはTaskBの中の `AddNested`です。

TaskAの結果を使ってさらに非同期ロードタスクを実行したいのですが、このままではネストしてしまいます。
そこで、TaskB自身にAddNestedでSubTaskを追加します。
これによって、TaskBはSubTaskが完了するまで完了しません。
TaskB自体は AddNestedのすぐ後に、 `return Asset->ChildAsset`でリザルトを書き込みます。しかしまだ完了しません。
Nestされたサブタスクが完了したら初めてTaskBは完了し、後続のTaskCが開始されるのです。

## Task詳解

`FTask` は `TTask<void>`のことです。TTaskはvoid型へのtemplate特殊化を持っています。
```cpp
using FTask = TTask<void>;
```

`FTask`および `TTask<TResult>`は軽量なハンドルオブジェクト。ただのハンドルなのでコピー可能です。Taskの実体は参照カウント式の共有ポインタとして管理されているので気軽にコピーしていいです。

`FTask`および `TTask<TResult>`をまとめて`Task`と呼ぶことにします。

### タスクの実行条件
タスクには`Prerequisites`で実行条件を定めることができます。実行条件とは、「実行条件Xが全て完了してからこのタスクを実行する」ことを意味します。

具体例を見た方が理解が早いでしょう。
`Task`は TArrayに格納することができて、依存タスク列として指定することができます。

```cpp: 事前条件

// TaskA,B,Cを並列で実行する
TArray<FTask> TaskList;
TaskList.Add(Launch(...)); // TaskA
TaskList.Add(Launch(...)); // TaskB
TaskList.Add(Launch(...)); // TaskC

// TaskZ は A,B,Cが全て完了してから実行される。
FTask TaskZ = Launch(..., Prerequisites(TaskList), ..);
```
もっとも頻繁に使うケースです。ユースケースは無限にありますが、通信周りが鉄板でしょう。

```cpp
TTask<FServerState> GetServerStateTask = NetworkService::GetServerStateAsync(); 

TTask<FLobbyResult> GetLobbyTask = Launch(...,[GetServerStateTask](){ 
    // サーバー状態確認
    FServerState State = GetServerStateTask.GetResult();
    if(サーバーダウン){ return サーバーダウンしてます }
    if(ServiceUnAvaibale){ return 現在プレーできません }

    // ロビー情報得る
    NetworkService::GetLobby(); 
}), Prerequisites(GetServerStateTask));

TTask<FLobbyResult> Join = Launch(...,[GetLobbyTask]()
{ 
    FLobbyResult Lobby = GetLobbyTask.GetResult();
    NetworkService::Join(Lobby); 
}), Prerequisites(GetLobbyTask));
```

実戦ではエラーハンドリングやUI同期など色々面倒ですが、雰囲気はわかると思います。
ラムダキャプチャで必要な情報を取得しつつ、上から下に流れるような処理が実装できます。

:::message
公式ドキュメントでは前提条件と翻訳されています。実行条件の方が分かりやすい
:::

### タスクの完了条件
タスクには`AddNested(ChildTask)` で完了条件を定めることができます。完了条件はタスク自身に子タスクを追加することで実装されます。完了条件とは、「完了条件Yが全て完了してからこのタスクを完了させる」を意味します。

```cpp: 完了条件
FTask TaskA = Launch(...[]()
{
    FTask SubTaskA1 =Lauch(...);
    FTask SubTaskA2 =Lauch(...);
    FTask SubTaskA3 =Lauch(...);
    AddNested(SubTaskA1, SubTaskA2, SubTaskA3);
});

//TaskAは すべてのサブタスクA1,A2,A3が完了してから完了する
```

これはタスクをまとめることができて非常に便利です。特に必要なリソースや情報をすべてまとめ上げてから下流工程に流すケースにおいて有用です。

```cpp
FTask LoadTask = Launc(...[]()
{
    // 非同期並列ですべてをロードする
    TTask<StaticMesh> MeshTask = LoadAssetAsyncTask<UStaticMesh>(MeshPtr);
    TTask<UNiagaraSystem> NiagaraTask = LoadAssetAsyncTask<UNiagaraSystem>(NiagaraPtr);
    AddNested(MeshTask, NiagaraTask);
});

FTask SpawnTask = Launch(..., []()
{
    // アセットを使う
},
Prerequisities(LoadTask));
```

C# の Taskと async-awaitほど流麗には実装できませんが、最低限スレッド指定機能や非同期待ち受け機能が備わっているため十分実戦でつかえます。

:::message
循環参照するとデッドロックします
:::

### FTaskEvent
`FTaskEvent` は さながらC#やjavascriptの`TaskCompletionSource` として利用できます。
Task化されていないありとあらゆる処理を Task化することができます。

上述の `LoadAssetAsyncTask`がまさにそれで、`FTaskEvent`を利用して コールバックをTaskへと繋ぎこんでいます。
これと同じ仕組みで通信機能やGameplayAbilityやなんでもかんでも Task化することができます。

# まとめ
連鎖した非同期処理を実装するためのモダンな実装を模索しました。
結果として、`UE::Tasks`を使うのが最強という結論に落ち着きました。名前付きスレッドを指定できるのが大変便利です。

`TaskSystem`は後発なだけあって、ほかの機構よりも改善が加えられているという所感です。
内部的にTaskGraphを使うなど、便利に使いやすくなるように整えられた非同期機構という印象です。

非同期周りの記事が2022年で終わっているので調査に苦労しました。
各種APIのオーバーヘッドの違いなども気になるところではありますが、当初の目的である非同期ロードチェインは達成できたのでゴールしちゃいます。

:::message
非同期周りはAI Agentに任せるとたまにデッドロックするのできちんとレビューしましょう
:::

# 参考資料
https://dev.epicgames.com/documentation/ja-jp/unreal-engine/tasks-systems-in-unreal-engine
https://dev.epicgames.com/documentation/ja-jp/unreal-engine/task-graph-insights-in-unreal-engine-5
https://www.docswell.com/s/EpicGamesJapan/5QMWWK-UE4_CEDECKYUSHU2021_MultiThread
https://qiita.com/EGJ-Takashi_Suzuki/items/83ac194e8a914a56b9d1
https://unrealengine.hatenablog.com/entry/2022/07/31/224504