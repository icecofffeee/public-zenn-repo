---
title: "UE5:Unreal Engineのポインタについてまとめた 後編 "
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: false
---

# はじめに

[UE5:Unreal Engineのポインタについてまとめた 前編](https://zenn.dev/allways/articles/ue5-unrealcpp-ptr) の続きです。
後編では、マネージドポインタについて記載します。

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

# 詳解

## 前提
マネージドポインタに関して前提条件を先に解説します。

### マネージドポインタは全てゲームスレッド利用
スレッドセーフではありません。なぜならガベージコレクションがメインスレッドで行われるからです。マネージドポインタを使用する場合は、上記ポインタの種類によらず必ずゲームスレッドで利用します。`AActor`や`UActorComponent`のライフサイクル関数内で利用する限りにおいてはゲームスレッドなので、普段は問題にならないはずです。

レンダースレッドや、ワーカースレッドで`UObject`へアクセスするのはいけません。
以降の説明は原則ゲームスレッドでの利用です。

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
:::message alert
基本的につかいません。理由は`Incremental Garbage Collection`が使えなくなるからです。
全てのハードオブジェクトポインタは`UPROPERTY`な`TObjectPtr`に乗り換えましょう。
:::

`UPROPERTY()` を付与することで、リフレクションシステムに登録し、GCのマークアンドスイープで索引されるようにした生ポインタです。

このポインタは`UMyObject`クラスのインスタンスが、`Pointer`が指す`UObject`を"保持"もしくは"依存"していることを示します。このポインタはGarbage Collectionシステムで走査対象になります。Garbage Collectionシステムはオブジェクトへの全てのハードポインタが`nullptr`になるか、オブジェクトが明示的に`Destroy()`がされない限り、そのオブジェクトを破棄しません。

:::message
よくある誤解ですが、`UPROPERTY`はGC回収を妨げるものの絶対の所有権を持つわけではありません。参照先のオブジェクトがどこかで明示的に`MarkAsGarbage()`されたら、GC回収されてしまい`Pointer`には`nullptr`がセットされます。また、`AActor`と`UActorComponent`に限っては更に挙動が変わります(後述)
:::

ハードオブジェクトポインタは static メンバー変数にできません。
コンパイルエラーになります。
`UPROPERTY() static inline UObject* Pointer{};`
![static member](/images/ue5-unrealcpp-ptr-second/uproperty_static_pointer.png)

### Hard Object Pointerの使い方
デリファレンスするときは必ず `IsValid()`を用いて死活チェックしてから利用します。先述の通りどこでDestroyされるか分からないので、基本的に`IsValid()`した方が安全です。パフォーマンス稼ぎのためにIsValid()を外したくなるときがありますが、クラッシュすると時間を奪われるので個人的にお勧めしてません。
```cpp
if(IsValid(Pointer))
{
    Pointer->DoSomething();
}
```

```cpp
    UObject* Object = NewObject<UObject>();
    Object->MarkAsGarbage(); // どこかでゴミマークされたとする
    //...

    // まだ GCが走っていないとすると、
    // operator bool や == nullptr はあんまり良くない
    // MarkAsGarbageなオブジェクトに対して trueを返してしまう
    if(Object != nullptr){}
    if(Object){}  // 非nullptrだからゴミだけどアクセスしちゃうよ
```

参照を外すときは素直に nullptrをセットします。GC回収を妨げないように明示的にnullptr代入をすることが大事です。
```cpp
void ReleaseReference()
{
    Pointer = nullptr;
}
```
どうしても自前で破棄したいときは`MarkAsGarbage()`か `ConditionalBeginDestroy()`を使います。
`MarkAsGarbage`は次のGCタイミングで回収されます。通常通りのライフサイクルを通るので安心です。`ConditionalBeginDestroy`はすぐさま`BeginDestroy`を呼び出そうとするので危険です。

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

### アセットを消すとnullptrになる
`UPROPERTY() UObject*` が何らかのアセットを参照している場合、エディタ上でそのアセットを `Force Delete`した場合 nullptrにセットされます。
そのため、アセットを参照したい場合はnullptrになり得ることを前提に実装しなければなりません。

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

基本形です。参照を保持して参照チェインによるGCを妨げたい場合は UPROPERTY() TObjectPtrにするべきです。
`TObjectPtr`は`ハード参照`です。`強い参照`ではありません。`ハード参照`は参照を保持しますが、所有はしません。参照を絶対所有したい場合は `TStrongObjectPtr`を使います。
逆にGCを妨げたくない場合は `TWeakObjectPtr`を使います。

`TObjectPtr` は Incremental Garbage Collection に対応しています。
ユーザーコード側で全てハードオブジェクトポインタを `TObjectPtr`に置き換えられるなら Incremental GCを利用することができます。

また、`TObjectPtr`はCook時に遅延ロードに対応しているため、Cook時間が有利です。

### ハード参照はインスタンス生成時にアセットロードしてしまう
`ハードオブジェクトポインタ`および、`TObjectPtr`はハード参照です。ハード参照はAssetを参照した場合、最初のクラスインスタンスが生成されたタイミングで一緒にロードされます。
そのため大量のアセットを`TObjectPtr`で参照してしまうと、大きなスパイクが発生したり、ロード時間が伸びる問題が発生します。

これを回避するには `TSoftObjectPtr`なるソフト参照を使用します。

※`Class Default Object` のときはロードされません

### TObjectPtrの使い方
`operator bool`を利用してnullptr比較が可能です。
ただし、前述の通り`T*`が生存していることに興味がある局面では`IsValid`を使う方が賢明です。
```cpp
if(Pointer){ /* pointerは非nullptr*/ }
if(IsValid(Pointer)){ /* pointerは非nullptr かつ Garbageでない*/ }
```

`T*`をデリファレンスしたいときは `GetValid()`を使って生ポにします。ifスコープの利用が賢明です。
```cpp
// GetValid<T>は IsValidなら T* をさもなければ nullptrを返すtemplate関数
if(T* Ptr = GetValid(Pointer))
{
    Ptr->DoSomething();
    Ptr->DoFooBar();

    //何回も operator->を呼び出すのは無駄なので
    // Pointer->DoSomething(); 
    // Pointer->DoFooBar(); 
}
```

生ポインタと違って、`TObjectPtr`はテンプレートクラスです。なので型情報を保持しています。
そのためCast を型安全かつconstexprに行えます。つまりコンパイル時に間違ったキャストを弾いてくれるのです。すばら。

is-a関係であるかに興味がある局面では`IsA<T>`を使用し、Castして利用したい場合は`Cast<T>`を使用します。
`Cast<T>`は TObjectPtr用の特殊化実装が存在するため型安全かつconstexprです。これがc++20の力だ！
```cpp
TObjectPtr<UObject> BasePtr = Pointer; // TObjectPtr<UMyBase>からTObjectPtr<UObject>へのアップキャスト
if(BasePtr.IsA<UMyObject>()){ /*その型の派生型であることに意味があるとき. ログ出すときとか.型タグ使うときとか*/ }
if(UMyObject* Obj = Cast<UMyObject>(BasePtr))
{
    Obj->DoSometing();
}

```

このようにTObjectPtrは生ポ、ハードオブジェクトポインタよりも価値が高いので、このままreturn してよいです。
生ポと TObjectPtrのサイズは(64bit環境において)同じなので TObjectPtrをreturnして問題ありません。
生ポをreturn しないようにしましょう。

```cpp
TObjectPtr<UObject> GetMyObject() const
{
    return Pointer; //TObjectPtr<UMyBase>からTObjectPtr<UObject>へのアップキャスト
}
```
