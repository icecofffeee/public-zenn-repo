---
title: "UE5:Unreal Engineのポインタについてまとめた 中編 "
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: true
---

# はじめに

[UE5:Unreal Engineのポインタについてまとめた 前編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr) の続きです。
中編では、マネージドポインタについて記載します。

* [UE5:Unreal Engineのポインタについてまとめた 前編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr)
* [UE5:Unreal Engineのポインタについてまとめた 中編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr-second)
* [UE5:Unreal Engineのポインタについてまとめた 後編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr-third)

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

## Hard Object Pointer

```cpp
UCLASS()
class UMyObject : public UObject
{
    GENERATED_BODY()

    UPROPERTY()
    UObject* Pointer{};
}
```
古い形式のUObjectを指すポインタです。
使いません。必ず`TObjectPtr<T>`に乗り換えましょう。

`UPROPERTY()` を付与することで、リフレクションシステムに登録された生のポインタです。GCのマークアンドスイープで索引されるようなっています。

このポインタは`UMyObject`クラスのインスタンスが、`Pointer`が指す`UObject`を"保持"もしくは"依存"していることを示します。このポインタはGarbage Collectionシステムで走査対象になります。Garbage Collectionシステムはオブジェクトへの全てのハードポインタが`nullptr`になるか、オブジェクトが明示的に`Destroy()`がされない限り、そのオブジェクトを破棄しません。全てのハードポインタです。ハードオブジェクトポインタを含みます。

:::message
よくある誤解ですが、`UPROPERTY`は所有権ではなく到達可能性(Reachability)を保障するものです。ルートから到達可能であるからGC回収を妨げる、という機能であって絶対の所有権を持つわけではありません。参照先のオブジェクトがどこかで明示的に`MarkAsGarbage()`されたら、GC回収されてしまい`Pointer`には`nullptr`がセットされます。
:::

:::message
`AActor`と`UActorComponent`に限っては更に挙動が変わります(後述)
:::


ハードオブジェクトポインタは static メンバー変数にできません。
コンパイルエラーになります。
`UPROPERTY() static inline UObject* Pointer{};`
![static member](/images/ue5-unrealcpp-ptr-second/uproperty_static_pointer.png)

### Hard Object Pointerは使わない
もう使いません。必ず`TObjectPtr<T>`に乗り換えましょう。
大事なことなので重ねて言及しました。

:::message alert
ハードオブジェクトポインタが紛れ込むと`Incremental Garbage Collection`が使えなくなります。全てのハードオブジェクトポインタは`UPROPERTY`な`TObjectPtr`に乗り換えましょう。
:::

### Hard Object Pointerの使い方
使いませんが、一応使い方について述べます。

#### Hard Object Pointerのデリファレンス
デリファレンスするときは必ず `IsValid()`を用いて死活チェックしてから利用します。どこで`MarkAsGarbage`されるか分からないので、基本的に`IsValid()`した方が安全です。ゲーム内に限らず、エディター、テストコード、プロファイリングなど`MarkAsGarbage`で破棄したくなるタイミングはちらほら出てきますから、油断なりません。特にエディタ上でAssetを`Force Delete`したときや、レベルからアクターを`Delete`したときが危ないです。そのアセットへのハード参照はnullptrにセットされますが、Slateでキャプチャしたりすると、タイミングによっては怪しいことになります。ワイルドな参照はダングリングポインタになります。

```cpp
if(IsValid(Pointer))
{
    Pointer->DoSomething();
}
```
パフォーマンス稼ぎのために`IsValid()`を外したくなるときがありますが、クラッシュすると時間を奪われるので個人的にお勧めしてません。

```cpp
    Object->MarkAsGarbage(); // どこかでゴミマークされたとする
    //...

    // まだ GCが走っていない状態では
    // operator bool や == nullptr は
    // MarkAsGarbageなオブジェクトに対して trueを返してしまう
    if(Object != nullptr){} // 非nullptrだからゴミだけどアクセスしちゃうよ
    if(Object){}  // 非nullptrだからゴミだけどアクセスしちゃうよ
```
#### Hard Object Pointerの生成
NewObjectで作ります。
```cpp
    UObject* Object = NewObject<UObject>();
```
作りたての状態では、参照チェインに含まれていないので、どこかのUPROPERTYに参照させておかねばなりません。
```cpp
    UObject* Object = NewObject<UObject>();
    this->Pointer = Object;
```
もしくは RootSet に明示的に登録するかです。

```cpp
    UObject* Object = NewObject<UObject>();
    Object->AddToRoot();
```

#### Hard Object Pointerの破棄
マネージドなので破棄はGC回収を期待します。参照を外すときは素直に`nullptr`をセットします。GC回収を遅延させないように明示的にnullptr代入をすることが大事です。親(Outer)からthisへの参照が残っていたとしてもthisから`Pointer`への参照を明示的に外しておけば`UObject`が1つ早期に回収されることが期待されます。
```cpp
void ReleaseReference()
{
    Pointer = nullptr;
}
```
自前で破棄したいときは`MarkAsGarbage()`か `ConditionalBeginDestroy()`を使います。
`MarkAsGarbage`は到達可能性に関わらず、次のGCタイミングで回収されます。通常通りのライフサイクルを通るので安心です。

```cpp
void ReleaseReference()
{
    // 推奨
    if(IsValid(Pointer)
    {
        Pointer->RemoveFromRoot(); //root setから除いておかないとMarkAsGarbageできないよ
        Pointer->MarkAsGarbage();
        Pointer = nullptr;
    }

    // 超難しい
    if(IsValid(Pointer)
    {
        Pointer->ConditionalBeginDestroy(); // BeginDestroyを呼ぶ
        Pointer = nullptr;
    }
}
```
`ConditionalBeginDestroy`はすぐさま`BeginDestroy`を呼び出そうとするので危険です。ライフサイクルを守らないので、UEのプロじゃない限り使わない方がいいです。
※ `AActor`と `UActorComponent`の破棄は別(後述)

### Hard Object Pointerの注意点
#### アセットを消すとnullptrになる
`UPROPERTY() UObject*` が何らかのアセットを参照している場合、エディタ上でそのアセットを `Force Delete`した場合 nullptrにセットされます。そのため、アセットを参照したい場合はnullptrになり得ることを前提に実装しなければなりません。Slateや UEditorSubsystem とかエディタ周りでアセット参照を握る場合は気を付けましょう。

---
## TObjectPtr
```cpp
UCLASS()
class UMyObject : public UObject
{
    GENERATED_BODY()

    UPROPERTY()
    TObjectPtr<UObject> Pointer;
}
```

`UObject`派生型を指す基本のポインタです。参照を保持して参照チェインによるGCを妨げたい場合は`UPROPERTY() TObjectPtr`にするべきです。`TObjectPtr`は`ハード参照`です。`強い参照`ではありません。`ハード参照`は参照を保持しますが、所有はしません。参照を絶対所有したい場合は `TStrongObjectPtr`を使います。逆にGCを妨げたくない場合は `TWeakObjectPtr`を使います。

`TObjectPtr` は生のアドレスではなく内部でオブジェクトへのハンドルとして持っています。パッケージビルドされた段階で生ポインタに変換されます。`sizeof(TObjectPtr<T>)`は生ポと同じです。なので値渡しでOKです。

`TObjectPtr` は Incremental Garbage Collection に対応しています。
ユーザーコード側で全てのハードオブジェクトポインタを `TObjectPtr`に置き換えられるなら Incremental GCを利用することができます。

また、`TObjectPtr`はCook時に遅延ロードに対応しているため、Cook時間の面において有利です。

### TObjectPtrの使い方
#### TObjectPtrのデリファレンス
`T*`をデリファレンスしたいときは `GetValid()`を使って生ポにします。ifスコープの利用が賢明です。これは何回も`operator ->`や`operator *`を利用するのは無駄だからです。`ResolveObjectHandle`の呼び出しもその都度行われます。
```cpp
// GetValid<T>は IsValidなら T* を,さもなければnullptrを返すtemplate関数
if(T* Ptr = GetValid(Pointer))
{
    Ptr->DoSomething();
    Ptr->DoFooBar();

    //何回も operator->を呼び出すのは無駄です
    // Pointer->DoSomething(); 
    // Pointer->DoFooBar(); 
}
```

`operator bool`を利用して`nullptr`比較が可能です。
ただし、`T*`が生存していることに意味がある局面では`IsValid`を使う方が賢明です。
```cpp
if(Pointer){ /* pointerは非nullptr だが Garbageかはわからない*/ }
if(IsValid(Pointer)){ /* pointerは非nullptr かつ Garbageでない*/ }
```
#### TObjectPtrの生成
`implicit` に生ポから変換できます。`operator=`や普通にコンストラクタを使えばよいです。

```cpp
UObject* RawPtr = NewObject<UObject>();

TObjectPtr<UObject> Obj0 = RawPtr;
TObjectPtr<UObject> Obj1(RawPtr); //生ポをはめることもできる
TObjectPtr<UObject> Obj2(nullptr); //明示的なnullptrコンストラクタもあるよ
TObjectPtr<UObject> Obj3(); //デフォルトコンストラクタはnullptr
```

普通は`AActor`等のコンストラクタ内でセットするか、エディタのDetailsビューからセットすると思います。
```cpp
AMyActor::AMyActor()
{
    Pointer = CreateDefaultSubObject<T>(TEXT("Hogehoge"));
}
```
#### TObjectPtrの破棄
nullptr設定するだけです。これは`operator=(TYPE_OF_NULLPTR)`がちゃんと実装されているからです。

```cpp
void ReleaseReference()
{
    Pointer = nullptr;
}
```
Pointerが指すオブジェクトのGC回収を早めたいならnullptrをセットするのもありです。

#### TObjectPtrのキャスト
生ポインタと違って、`TObjectPtr`はテンプレートクラスです。なので型情報を保持しています。
そのため`Cast`を型安全かつ`constexpr`に行えます。つまりコンパイル時に間違ったキャストを弾いてくれるのです。すばら。

`is-a`関係であるかに興味がある局面では`IsA<T>`を使用し、`Cast`して利用したい場合は`Cast<T>`を使用します。`Cast<T>`は `TObjectPtr`用のtemplate実装が存在するため型安全です。
```cpp
// .h
UPROPERYT() TObjectPtr<UMyBase> Pointer{};

// .cpp
void Main()
{
    // TObjectPtr<UMyBase>からTObjectPtr<UObject>へのアップキャスト
    // 静的な型チェックは operator= でやってくれている
    TObjectPtr<UObject> BasePtr = Pointer;

    // その型の派生型であることに意味があるときはIsA<T>() 
    if(BasePtr.IsA<UMyObject>())
    {
         /*ログ出すときとか.型タグ使うときとか*/ 
    }

    // もっぱらコレ
    if(UMyObject* Obj = Cast<UMyObject>(BasePtr))
    {
        Obj->DoSometing();
    }
}
```
#### TObjectPtrの関数の引数・戻り値利用
`TObjectPtr`は生ポ、ハードオブジェクトポインタよりも価値が高いので、このまま`return`してよいです。生ポと `TObjectPtr`のサイズは同じなので `TObjectPtr`を`return`して問題ありません。生ポはスコープを超えて使うべきでないので、`return`しないようにしましょう。

```cpp
// .h
UPROPERYT() TObjectPtr<UMyBase> Pointer{};

// C++ Nativeの世界は生ポ滅ぶべし
TObjectPtr<UObject> UseObject_ForNative(TObjectPtr<UObject> Other)
{
    return Pointer; 
}
```

`UFUNCTION`では`TObjectPtr`は使えないのでしゃあなしで生ポにします。

```cpp
// UFunctionはしょうがないので生ポにする
UFUNCTION(BlueprintCallable)
UObject* UseObject_ForBP(UObject* Other) const
{
    // なんか使う
    // ...
    return Pointer; 
}
```

`UFUNCTION`とそれ以外でいちいち関数分けてられない、というのはごもっともなので、ほどほどにバランスをとってください。

### TObjectPtrの注意点
#### ハード参照はインスタンス生成時にアセットロードしてしまう
`ハードオブジェクトポインタ`および、`TObjectPtr`はハード参照です。ハード参照はAssetを参照した場合、最初のクラスインスタンスが生成されたタイミング[^1]で一緒にロードされます。
そのため大量のアセットを`TObjectPtr`で参照してしまうと、大きなスパイクが発生したり、ロード時間が伸びる問題が発生します。

これを回避するには `TSoftObjectPtr`を使用します。
インスタンス参照は`TObjectPtr`, アセット参照は`TSoftObjectPtr`を使うと覚えておくとよいでしょう。

[^1]: `Class Default Object` のときはロードされません

## TSoftObjectPtr
```cpp
UPROPERTY() TSoftObjectPtr<UObject> Pointer;
```
ソフトオブジェクトポインタはアセットへのハード参照をしないポインタです。実行時に自動ではロードしません。中身は`FSoftObjectPath`と`FWeakObjectPtr`のペアです。つまりアセットへのパスとロードしたアセットへの弱参照を保持しています。

ただの`FString`と異なりリダイレクタやエディタ統合に対応しています。型Tを持っているため、特定のアセットしか設定できないようになっています。Detailsビューがちゃんと対応されており、ハード参照とほぼ変わらずにアセットを設定できます。


### TSoftObjectPtrの使い方

#### TSoftObjectPtrの生成
普通は`UPROPERTY(EditDefaultsOnly)`にしてエディターから設定します。
```cpp
    UPROPERTY(EditDefaultsOnly)
    TSoftObjectPtr<UStaticMesh> Mesh;
```
C++でハードコードするときはパスを指定します。

```cpp
void Main()
{
    TSoftObjectPtr<UStaticMesh> Mesh = FSoftObjectPath("/Game/Path/To/Mesh");
}
```

#### TSoftObjectPtrの破棄
パスと弱参照しかもっていないので別に破棄する必要はありませんが、Resetで同じインスタンスを使いまわせます。
```cpp
void Main()
{
    TSoftObjectPtr<UStaticMesh> Mesh = FSoftObjectPath("/Game/Path/To/Mesh");
    Mesh.Reset();
}
```

#### TSoftObjectPtrのロード
同期ロードはメンバメソッドを直接叩きます。ロード済みのインスタンス参照は自身で保持する必要があります。TSoftObjectPtrはロードしたものへの弱参照しかもっていないので、GC回収されちゃいます。
```cpp
UPROPERTY(EditDefaultsOnly)
TSoftObjectPtr<UStaticMesh> Mesh;

UPROPERTY()
TObjectPtr<UStaticMesh> LoadedMesh;

void Main()
{
    TSoftObjectPtr<UStaticMesh> MeshAsset = FSoftObjectPath("/Game/Path/To/Mesh");
    UStaticMesh* Mesh = MeshAsset.LoadSynchronous();

    // ここでGCされるとMeshは回収されてしまう
    // なので UPRRPERTY()なLoadedMeshに保持しておく
    LoadedMesh = Mesh;
}
```

#### TSoftObjectPtrの非同期ロード
非同期ロードは公式ドキュメントの通りです。
[アセットの非同期ロード](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/asynchronous-asset-loading-in-unreal-engine)


`FStreamableManager` を使って RequestAsyncLoadを呼びます。ロード完了のコールバックで受け取ります。

早くコルーチンか`TFuture`対応来てくれぇ!

### TSoftObjectPtrの死活チェック

3種類あります。これは3状態を識別するものです。それぞれの状態は排他です。

1. `IsNull()` - 有効なパスを指していない
2. `IsPending()` - 有効なパスを指しているが、ロードしていない、もしくはロード中、もしくはGC回収済み
3. `IsValid()` - 有効なパスを指しており、ロード済み

有効なパスを指していないがロード済みという状態はありえません。

```cpp
TSoftObjectPtr Ptr = FSoftObjectPath("Game/Path/To/Asset");

if(Ptr.IsNull()){} // パスが間違っている or パスが空文字

if(Ptr.IsPending(){} // パスは合っている. 未ロード or ロード中

if(Ptr.IsValid()){} // ロード完了

```

`UPROPERTY()`な`SoftObjectPtr`をちゃんとEditorから設定できているならば`IsPending()`となります。`IsValid()`は内部の`WeakObjectPtr`が活きている間は`true`を返すということなので、将来的にGC回収されたら`IsPending()`へと戻ります。`ResetWeakPtr()`で`IsValid()`から`IsPending()`へ明示的に戻せますが、普通そんなことしないでしょう。

```cpp
if(Ptr.IsValid())
{
    Ptr.ResetWeakPtr(); //内部の弱参照をResetしてIsPendingに戻す.
    check(Ptr.IsPending());

    //もいっかいロードしたら別インスタンスが得られる
    UObject* Loaded2 = Ptr.LoadSynchronus(); 
}

```

`IsValid()`は内部で`dynamic_cast`を用いており重めです。ロード完了したかどうかはロード済みのアセットの有無で確認した方が賢明でしょう。

```cpp
UPROEPRTY() TSoftObjectPath<UObject> SoftObjectPtr;
UPROPERTY() TObjectPtr<UObject> Loaded;

void BeginPlay()
{
    this->Loaded = SoftObjectPtr.LoadSynchronus();

    if(SoftObjectPtr.IsValid()){} // これは少し重め

    if(IsValid(Loaded)){} // ロード済みかどうかはロード済みObjectの死活チェックで良いよね
}
```

## TWeakObjectPtr
UObjectのGCを妨げることがない弱参照です。

```cpp
UPROPERTY()
TWeakObjectPtr<UObject> WeakObject;
```

中身はObjectのIndexとシリアルNoを持つ8byteの値型です。`T*` 自体は持っておらずポインタではないのですが、ポインタとして振る舞います。
GCを妨げないということで、有れば使い、なかったら使わないというようなニーズを満たすときに重宝します。

### TWeakObjectPtrの使い方
#### TWeakObjectPtrの生成
`UObject*`や `TObjectPtr`、`TStrongObjectPtr`から暗黙的に変換できます。

```cpp
TObjectPtr<UObject> Object = NewObject<UObject>();
TWeakObjectPtr<UObject> Weak = Object;
```

明示的に作りたい場合は `MakeWeakObjectPtr<T>`を使います。
生ポもTObjectPtrも両方対応しています。

```cpp
TObjectPtr<UObject> Object = NewObject<UObject>();
UObject* RawPtr = NewObject<UObject>();
TWeakObjectPtr<UObject> WeakFromTObj = MakeWeakObjectPtr(Object);
TWeakObjectPtr<UObject> WeakFromRaw = MakeWeakObjectPtr(RawPtr);
```

#### TWeakObjectPtrの破棄
`Reset`を使うか、`nullptr`をセットします。
```cpp
TObjectPtr<UObject> Object = NewObject<UObject>();
TWeakObjectPtr<UObject> Weak = Object;
Weak.Reset();
Weak = nullptr;
```

#### TWeakObjectPtrの死活チェック
活きているかどうかに興味がある局面においては`IsValid()`を使用します。

```cpp
if(Weak.IsValid())
{
    UE_LOG(LogTemp, Display, TEXT("まだ活きている"));
}
else
{
    UE_LOG(LogTemp, Display, TEXT("もう死んだか、最初からnullptrか、Staleした"));
}
```
`IsValid` には引数が2個ついています。`bEvenIfPendingKill` と `bThreadSafeTest`です。

`bEvenIfPendingKill=true`はガベージマーク済みなゴミオブジェクトを指していてもtrueを返します。この段階では有効なアドレスを指しているため、デリファレンスしてもアクセス違反にはなりません。よくわからないなら`false`にしてください。

`bThreadSafeTest=true`はUObjectのルートからの到達可能性チェックを行わず`GObjectArray[i]`が`nullptr`かどうかだけをチェックします。GCのマークフェーズの間でも`nullptr`かチェックしたいときに使います。
どちらも普通のユーザーは使いません。

```cpp
constexpr bool bEvenIfPendingKill = true;
constexpr bool bThreadSafeTest = true;
if(Weak.IsValid(bEvenIfPendingKill, bThreadSafeTest))
{
    // ゴミかもしれないが、メモリ領域は活きている
}
```

ワーカースレッドから行える最強の死活チェックは`Pin`です。
```cpp
if(TStringObjectPtr Strong = Weak.Pin())
{
    // スレッドセーフにまだ活きていることが確実
}
else
{
    // すでに死んでいる
}
```


#### TWeakObjectPtrのデリファレンス
弱参照なのでいつ死ぬかわかりません。そのため使用する度にチェックする必要があります。
基本的には `if`スコープで `Pin`もしくは`Get`を使います。
ゲームスレッドで利用する場合は `Get`で問題ありません。

```cpp
if(UObject* Ptr = Weak.Get())
{
    Ptr->DoSomething();
}
```
`Get`では中身が活きているかどうかをチェックした上で、有効なときのみ有効なアドレスを返し、死んでいる場合は`nullptr`を返します。

`Get(bEvenIfPendingKill)`ではガベージマーク済みなオブジェクトであっても有効なアドレスを返します。通常使いません。

```cpp
TObjectPtr<UObject> Object = NewObject<UObject>();
TWeakObjectPtr<UObject> Weak = Object;
Object->MarkAsGarbage();
if(UObject* Ptr = Weak.Get(/*bEvenIfPendingKill*/true))
{
    Ptr->DoSomething(); //まだGCされてないからぎりぎりセーフ
}
```

`Pin`を使うと弱参照から強参照に変換してGCを確実に妨げることができます。`Pin`はスレッドセーフです。
内部では `FGCScopeGuard`という`class`を用いてGCとの排他制御を行っています。もしPin止めが間に合わなかったときはnullptrが帰ります。
```cpp
// Slateの実行コンテキスト等において使う
if(TStrongObjectPtr<UObject> Ptr = Weak.Pin())
{
    Ptr->DoSometing(); //絶対GCされてない
}
```

`Pin(bEvenIfPendingKill)`ではガベージマーク済みなオブジェクトであっても有効なアドレスを返します。
こちらは強参照であるため例えガベージマーク済みであっても、強参照が活きている限りGCを妨げます。通常使いません。

### `Pin`は スレッドセーフだが`T*`はスレッドセーフではない
`WeakObjectPtr::Pin`はスレッドセーフです。GCに対してLockを取りつつ確実に`IsValid`な `TStrongObjectPtr`を取得できます。

ですが、肝心の`T*`に関してはそれがスレッドセーフかどうかは実装によります。ほとんどの`UObject`派生型はスレッドセーフじゃありません。
```cpp
if(T* Ptr = Weak.Pin())
{
    Ptr->DoSometing_ConCurrent(); //自前でスレッドセーフな関数を実装するべし
    int Value = Ptr->GetValue_ConCurrent(); //std::atomicやCriticalSection等でスレッドセーフにするべし
}
```

`Slate`の世界は`UObject`管理外なので、ラムダキャプチャした`UObject`がGCされてしまわないように`WeakObjectPtr`で参照を引き回したり、`StrongObjectPtr`で確実に所有権を握る必要がでてきます。Unreal Editor拡張を実装するにあたって`Slate`は避けて通れず、また`UObject`も触らないわけにはいかないので、マネージドポインタを触る場合は特に気を付ける必要があります。


## TStrongObjectPtr
マジの強いオブジェクト参照です。GCを必ず妨げます。`RefCounted`フラグを立てて、明示的な参照カウント方式に切り替えます。
たとえ`Unreachable`フラグが立っている到達不可能なオブジェクトであってもGCを妨げます。

:::message
参照カウント式なので、循環参照には十分気を付けてください。
:::

:::message
強い参照なので後片付けを忘れないように注意してください。メンバー変数として保持するときは`EndPlay`等の適切なタイミングで `Reset()`する必要があります。
:::

ゲーム内では`TObjectPtr`で十分なので登場頻度は低いと思います。
`TWeakObjectPtr`の`Pin`から得ることが大半でしょう。

もっぱら Editor拡張で使われます。
典型例はアセットのコンバートやバリデーション等です。処理中のアセットがGCされたり削除されると例外だすので、事前に強い参照を保持してから望むというものです。

### TStrongObjectPtrの使い方

#### TStrongObjectPtrの生成
`WeakObjectPtr<T>::Pin` から貰います。もしくは明示的にコンストラクタを呼びます。
```cpp
TObjectPtr<UObject> Obj = NewObject<UObject>();
TWeakObjectPtr<UObject> Weak = Obj;
TStrongObjectPtr<UObject> StrongFromObject(Obj);
TStrongObjectPtr<UObject> StrongFromWeak = Weak.Pin();
```

#### TStrongObjectPtrの破棄
`Reset()`を呼び出すか, `nullptr`をセットします。
参照カウント式なので確実に破棄しましょう。デストラクタで自動的に`Reset`されるためで一時変数などは大丈夫でしょう。
メンバ変数に持つ場合は寿命が伸びる可能性があるので適切なライフスコープで破棄してください。

```cpp
TStrongObjectPtr<UObject> Strong = Weak.Pin();
Strong.Reset();   // 参照カウントを減らし、内部ポインタをnullptrにする
Strong = nullptr; // 参照カウントを減らし、内部ポインタをnullptrにする
```
全ての`TStrongObjectPtr`から解放されたオブジェクトは従来のマークアンドスイープ方式でGCされます。

#### TStrongObjectPtrのデリファレンス
`Get()`で取得します。強参照であるので、非nullptrであるのならば確実に活きています。
そのため`operator->`や `operator*`を直接使っても問題ありません。覚えることを減らすためほかのポインタ同様`Get`を使えばいいと思います。

```cpp
TStrongObjectPtr<UObject> Strong = Weak.Pin();
if(UObject* Ptr = Strong.Get())
{
    Ptr->DoSomething();
}
```

#### TStrongObjectPtrの死活チェック
`IsValid()`でチェックします。強参照であるので、非`nullptr`であるのならば確実に活きています。
`Pin`止めが間に合わなかったときは`nullptr`が返るので必ず一度はチェックしましょう。
チェック済みの`StringObjectPtr`は ずっと非`nullptr`なので以降のチェックは省けます。
```cpp
TStrongObjectPtr<UObject> Strong = Weak.Pin();
if(Strong.IsValid())
{
    // ピン止めが間に合った
}
else
{
    // ピン止めが間に合わなかった or Weakが最初からnullptrだった
}
```

### TStrongObjectPtr更に詳しく
内部的に `UObject::AddRef`/`ReleaseRef()`を呼び出します。これにより、`EInternalObjectFlags::RefCounted`フラグが立ちます。
このフラグは `EInternalObjectFlags_GarbageCollectionKeepFlags` に含まれているため、GCのスイープフェーズで回収されません。

## Wild Pointer (Raw Pointer)
```cpp
UCLASS()
class UMyObject : public UObject
{
    GENERATED_BODY()

    UObject* Pointer {nullptr}; // UPROPERTY()がない！
}
```
`UPROPERTY()`がついていないため、リフレクションシステムから辿ることができない野生のポインタです。このポインタが指すオブジェクトはIsValidなのかGC回収済みなのか全くわかりません。

### Wild Pointer の使い方
ありません。使ってはいけません。ダンリングポインタになるのでクラッシュするか、最悪生動きします。

### Wild Pointer の死活チェック
できません。`IsValid()`ではチェックできません。`IsValidLowLevel()`でもチェックできません。

`IsValidLowLevel`は 活きているかのチェックではなく、ダングリングポインタの恐れがあるかのチェックに使用します。デバッグ用です。誤判定する可能性があるため、デリファレンスしてアクセスしてはいけません。
```cpp
if(!IsValidLowLevel(Object))
{
    // ログを出すなどの為に使用する
    UE_LOG(LogTemp, Error, TEXT("ダングリングポインタかもしれない!!!!"));

    // これはダメ↓ GetNameの呼び出しでクラッシュする可能性が高い
    // UE_LOG(LogTemp, Error, TEXT("%s"), Obj->GetName());
}
```

### Wild Pointer 何故ダメか
このポインタを使ったときの最悪のシナリオとしては、GCに回収された無効オブジェクトが再利用されて有効な別のUObject派生型へと転生して使用されることです。
低い可能性ではありますが、同一アドレスに別のUObjectが生まれる可能性があるのです。
例：
```cpp
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

    void BeginPlay()
    {
        WildPrimitive = NewObject<PrimitiveComponent>();

        // このスコープ内においてのみ活きていることが保障できる
    }

    void Tick()
    {
        // この時点で WildPrimitiveはもう死んでいる可能性がある
        // ...


        if(IsValid(WildPrimitive)) // チェックできてない意味のないチェック
        {
        }
    }

    void EndPlay( ... )
    {
        if(IsValid(WildPrimitive)) // チェックできてない意味のないチェック
        {
            WildPrimitive->MarkAsGarbage();// クラッシュの危険
            WildPrimitive = nullptr;
        }
    }

    UPrimitiveComponent* WildPrimitive {nullptr};
}
```

この例では`WildPrimitive`に`PrimitiveComponent`を差しました。通常 `PrimitiveComponent`型であることを期待するはずです。プログラム上でも合っています。
ただし、GCされた場合、`UPROPERTY()`ではないので`nullptr`に書き戻されることもありません。相変わらずポインタは無効領域を差しています。
このとき、デリファレンスすると割り当てられたメモリ領域外にアクセスして未定義動作を起こします。運がよければクラッシュして問題が顕在化します。

運が悪いと、別の`NewObject<T>`が実行され、低い確率でたまたま同アドレスに新たな`T`が割り当てられたとします。`T`もまた`UObject`派生型であるというせいで、`UObjectBase::InternalIndex`などにはギリギリアクセスできてしまいます。そのため`IsValidLowLevel`などは転生した`T`に対して`true`となるときがあります。が、当然自身は`PrimitiveComponent`であると思い込んでいるので、そのsizeでアクセスするし、vtableもそれだと思い込んでいます。謎領域をreadしながら生動きする可能性があります。デバッグビルドだとアクセス違反例外が出ると思います。最適化により例外チェック機構が無効化されていると出ないこともあります。
いずれにせよ、生動きにより問題が顕在化しない、もしくは深刻化する可能性が大いにあります。

生ポ絶対ダメ！

# つづく
予想通り書くことが多かったので後編に続きます。
