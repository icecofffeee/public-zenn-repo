---
title: "UE5:Unreal EngineのUE_LOGFMTについての研究"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: true
---

# はじめに
Unreal Engine 5.2 より `UE_LOGFMT` が登場しました。
`%d`, `%s`のようなフォーマット指定子を使うことなく位置指定子 `{}`でType付けされたフォーマットを実行することができます。
つよつよなログ機構なので`UE_LOG`の上位互換となることができます。

本稿では`UE_LOGFMT`の基本的な使い方のほかにカスタム型のLOGFMT対応について調べました。
かなり以前からある機能であるにもかかわらず、だれも具体的なカスタムclassへのLOGFMT対応について触れてくれません。
そのせいで `2026/01`現在であってもAIはちゃらんぽらんな回答を返します。

ならやるしかねぇよなぁ。

# UE_LOGFMT

[公式ドキュメント](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/logging-in-unreal-engine#ue-logfmt)

位置パラメータと名前付きパラメータの二通りに対応しています。

## 忙しい人のための使い方サンプル

```cpp
#include "Logging/StructuredLog.h"

void AActor::BeginPlay()
{
    UE_LOGFMT(LogTemp, Verbose, "BeginPlayだよ");

    UE_LOGFMT(LogTemp, Display, "Actor :{Name}", this->GetName());

    TSoftObjectPtr<UObject> Ptr;
    UE_LOGFMT(LogTemp, Warning, "Failed to load asset:{Asset}", Ptr->GetSoftObjectPath());
}
```

## include と依存モジュール
```cpp
#include "Logging/StructuredLog.h"
```
をインクルードします。

`Core`モジュールにいるので`Build.cs`へ追加の依存を足す必要はありません。

## 位置パラメータを使った方法

`{0}`, `{1}` のように`{}`を位置指定子として指定できます。
この位置指定子に与える名前をフィールド名と言います。
例えば `{0}` としたときそのフィールド名は`0`です

フィールド名は`unique non-empty` 名であることが要求されます。


```cpp: 位置指定
FName Name = TEXT("Foo");
int Value = 42;
UE_LOGFMT(LogTemp, Log, "Name={0}, Value={1}", Name, Value);
// "LogTemp Log Name=Foo, Value=42" とログ出力される
```

ユニークであるということは同じ引数を2か所に表示できないということです。
コンパイルは通りますが、`check`で止まります。
```cpp: フィールド名がユニークじゃない
int Value = 42;
UE_LOGFMT(LogTemp, Log, "{0} == {0}", Value);
// 実行時エラー checkで止まる
// ただし "42 == 42"というログ出力自体はちゃんと出る
```


### 位置パラメータと引数の対応関係
位置パラメータの登場順と引数の順番が対応しています。フィールド名は一切関係がないことに注意が必要です。`{0}`と`{1}`はさもパラメータパックのindexかのような見た目ですが、ただの名前でありindexではありません。`{1}`と`{0}`の順番を入れ替えても効果は変わらず、与えた引数の順です。注意してください。

入れ替えた事例です。
```cpp: 数値指定
FName Name = TEXT("Name");
int Value = 42;
UE_LOGFMT(LogTemp, Log, "Value={1}, Name={0}", Name, Value);
// "Value=Foo, Name=42" とログ出力される。期待と異なり入れ替わらない
```

ログの出力結果を見るとValueとNameが入れ替わっていません。
ただの名前ですので、位置パラメータでは任意の文字列でも使用可能です。
フィールド名と実引数名はいっさい関係がありません。順番のみが考慮されます。

```cpp: 引数の順番間違えた例
// Name, Valueと与えるべきところをValue,Nameと与えている！
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}", Value, Name);
// "Name=42, Value=Foo" とログ出力される
```

このときログの意味は壊れていますが、`%s`のようなフォーマット指定子を使わないため、安全にシリアライズされます。クラッシュしません。最悪、ログが出ないかログがヘンテコになるだけでクラッシュしないのは助かります。

もしフィールド名と実引数を対応付けしたい場合は、後述の名前付きパラメータを利用します。

## 名前付きパラメータを使った方法

名前付きパラメータでは、引数に名前と値のタプルを与えます。
タプルの0番目にはフォーマット文字列で指定したフィールド名を、タプルの1番目にはログ出力したい値を与えます。

```cpp
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}",
            ("Name", Name),
            ("Value", Value));
// "Name=Foo, Value=42" とログ出力される
```

名前指定なのでログフィールドや引数の順番を入れ替えても正しく表示されます。
```cpp
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}",
             ("Value", Value),
             ("Name", Name));
// "Name=Foo, Value=42" とログ出力される
```

可読性の観点や指定順のミスを防ぎやすい点に優れています。手で書くと面倒くさいのが玉に瑕ですが、AI Agentに実装させるときはこちらの方が安全かと思います。

### 同名禁止

フィールド名はユニークである必要があります。内部的に`TMap`に格納されるためです。

```cpp
const FName ClientName = TEXT("Client");
const FName ServerName = TEXT("Server");
// 実行時エラー checkfで止まる
UE_LOGFMT(LogTemp, Log, "Server={Name}, Client={Name}",
         ("Name", ServerName),
         ("Name", ClientName));

// 正しくはユニーク名にする
UE_LOGFMT(LogTemp, Log, "Server={ServerName}, Client={ClientName}",
         ("ServerName", ServerName),
         ("ClientName", ClientName));
```

## `TEXT`マクロ不要
`UE_LOGFMT`では フォーマット文字列にたいして　`TEXT`マクロを使う必要がありません。`UE_LOGFMT`マクロ内で`TEXT`マクロが付与されるため、つけてしまうとコンパイルエラーになってしまいます。

```cpp
// old style
UE_LOG(LogTemp, Log, TEXT("日本語でおｋ"));

// new style
UE_LOGFMT(LogTemp, Log, "日本語でおｋ");
```

`printf`スタイルに近づいてより読みやすく書きやすくなったのではないでしょうか。

## 組み込み型対応済み

`UE_LOGFMT`は UE組み込み型にかなり対応してくれています。

プリミティブ
* int32
* uint32
* float
* double
* bool

文字列系：
* FString
* FName
* FGuid

ポインタ系:
* `TObjectPtr<T>`
* `TSoftObjectPtr<T>`

コンテナ:
* `TArray<T>`
* `TMap<T>`


:::message
`FVector`, `FTransform`などの数学系構造体が非対応です
:::

## FString の デリファレンス不要
`UE_LOG` では `%s`に与える文字列は`FString::operator*()` を用いて`c-style`文字列として与える必要がありました。

```cpp
const FString Str = TEXT("エラーが発生しました");
UE_LOG(LogTemp, Log, TEXT("%s"), *Str);
// *Str の "*" がキモイ
```

とくに `FString::Format`を使うと汚くなっていました。
`TEXT`がいるせいで、閉じ括弧`)`の数がよくわからないですね。

```cpp
const int errorCode = -1;
UE_LOG(LogTemp, Log, TEXT("%s"), *FString::Format(TEXT("エラーが発生しました:%d"), errorCode));
// *FString::Format の "*"がキモイ
```

`UE_LOGFMT`では型付けされてシリアライズされることによりこの不満点が解決されています。
`FString`も`FName`も直接与えてよいです。

```cpp
const FString ErrorMessageStr = TEXT("ストレージが不足しています");
UE_LOGFMT(LogTemp, Log, "エラーが発生しました {0}", ErrorMessageStr);

const int errorCode = -1;
UE_LOGFMT(LogTemp, Log, "{0}", FString::Format(TEXT("エラーが発生しました:%d"), errorCode));
```

激熱ですねぇ。

## セキュリティ対応
### Format String Vulnerability
有名なC言語の脆弱性ですが、ユーザー入力の文字列をフォーマット文字列として与えてはいけません。
```cpp
void ParseUserInput(const char* Buffer, int Length)
{
    // 脆弱性！
    printf(Buffer);

    // こっちは安全
    printf("%s", Buffer); 
}
```

詳しくは添付資料をご覧ください。
https://www.jpcert.or.jp/research/materials.html#Formatted_Output
https://www.jpcert.or.jp/research/2011/7-FormatString.pdf


`UE_LOGFMT` では型付けされたシリアライズを実行するためフォーマット文字列を必ず要求します。`constexpr` と `static_assert`を駆使することでこれができなくなっておりコンパイルエラーになります。

```cpp
void ParseUserInput(const char* Buffer, int Length)
{
    // できないよ
    UE_LOGFMT(LogTemp, Log, Buffer);

    // 安全
    UE_LOGFMT(LogTemp, Log, "{0}", Buffer);
}
```

### フォーマット指定子の指定ミスでクラッシュしない
`%s`にint を与えたり, `%d`に文字列を与えたりすると undefined behaviourで爆発します。
`UE_LOGFMT`ではそもそもフォーマット指定子を使わないので間違えません。
実際にあったバグをご紹介しましょう。

```cpp
int ErrorCode = CreateDevice(&Device);
if(ErrorCode != Success)
{
    // クラッシュ！
    UE_LOG(LogFoo, Error, TEXT("Device Not Found: %s"), ErrorCode);
    return;
}
```

バグに気付きましたか？ `%s` に `int ErrorCode`を与えている点がバグです。正しくは`%d`を使うべきです。

さて、このバグは市場で発火します。なぜなら開発環境では正しいデバイスが常に存在するからです。ユーザーからは特定の環境でゲームがクラッシュする、という報告がきます。どうやらDeviceがないときに発生するようですね。デバイスの接続を確認するとかデバイス設定を見直すとかドライバを更新するとか、見当違いの対策を練り始めてしまいます。

真因は違います。エラーが原因ではなくエラーハンドリングが原因なのです。
しかも、大抵が滅多に通らない（そしてテストされない）パスで発生します。よって市場へリリースされてしまうのです。このように真の原因を隠蔽してしまう恐れがあるため、フォーマット指定子の指定ミスは恐ろしいのです。

なお、フォーマット指定子のミスは未定義動作なのでクラッシュするとは限りません。クラッシュすれば御の字で、生動きする可能性は大いにあります。未定義動作ですから。

## 指定子と引数の数が一致しなくてもクラッシュしない
`UE_LOGFMT`では、引数と指定子の数が一致していないとき正しく無視されます。
コンパイルエラーにはなりませんが、実行時に`check`でエラーを出してくれます。
`DO_CHECK`なしビルドだと`check`で止まりませんが、クラッシュせずに進行します。


* フィールド数より引数が多い時あふれた分は無視される。
```cpp: 引数が多いとき
UE_LOGFMT(LogTemp, Log, "{0}", 1, 2);
// "1" が表示され "2"は無視されるっぽい。
```

* フィールド数より引数が少ないときは最後のパラメータが繰り返されるっぽい。
```cpp: 引数が少ないとき
UE_LOGFMT(LogTemp, Log, "{0} {1}", 3);
// "3 3" が表示される
```

## 対応していない型
数学系構造体が対応していません。肝心の`FVector`系がないのはつらいです。

* FVector
* FRotator
* FTransform
* FBox
* ...

```cpp: コンパイルエラー
void Test()
{
    // コンパイルエラー!
    FVector Location(1.11, 2.22, 3.33);
    UE_LOGFMT(LogTemp, Log, "Position:{Pos}", Location);
}
```

これらは`ToString()` や `ToCompactString()` 等を使います。

```cpp
void Test()
{
    FVector Location(1.11, 2.22, 3.33);
    // Position:X=1.110 Y=2.220 Z=3.330
    UE_LOGFMT(LogTemp, Log, "Position:{Pos}", Location.ToString());

    // Position:V(X=1.11, Y=2.22, Z=3.33)
    UE_LOGFMT(LogTemp, Log, "Position:{Pos}", Location.ToCompactString());

    // Position:1.1100000000000001 2.2200000000000002 3.3300000000000001
    UE_LOGFMT(LogTemp, Log, "Position:{X} {Y} {Z}", Location.X, Location.Y, Location.Z);
}
```

`UE_LOGFMT`では小数の桁数を指定できないため、明示的に指定する想定と受け取りました。
---

# カスタム型を UE_LOGFMT対応させる

本題です。`UE_LOGFMT`ではユーザー定義の`class`/`struct` を `UE_LOGFMT`に対応させることができます。`SerializeForLog`関数を定義するだけなのですが、その実装方法がよくわかりませんでしたので、本稿でまとめました。ユーザー型はなんでもよいのですが、いくつか例示します。

## 例1: Vec3型
`FVector`が`UE_LOGFMT`非対応なのでオレオレ`Vec3`を構造化ログ対応してみます。

```cpp: MyVec3.h
class FCbWriter;

struct FMyVec3
{
    float X = 0;
    float Y = 0;
    float Z = 0;

    friend void SerializeForLog(FCbWriter& Writer, const FMyVec3& Vec3);
};
```

```cpp: MyVec3.cpp
#include "Logging/StructuredLog.h"
#include "Serialization/CompactBinaryWriter.h"

void SerializeForLog(FCbWriter& Writer, const FMyVec3& Vec3)
{
    Writer.BeginObject();
    Writer.AddFloat(ANSITEXTVIEW("X"), Vec3.X);
    Writer.AddFloat(ANSITEXTVIEW("Y"), Vec3.Y);
    Writer.AddFloat(ANSITEXTVIEW("Z"), Vec3.Z);
    Writer.EndObject();
}
```
実装できました。使ってみましょう。

```cpp: user code
const FMyVec3 Vec{2,3,4};
UE_LOGFMT(LogTemp, Log, "{0}", Vec);
// {"X":2, "Y":3, "Z":4}
```

## 例2: カスタム型をもつカスタム型
さらにカスタム型をもつカスタム型を`UE_LOGFMT`に対応させてみましょう。

```cpp: Locator.h
// 位置と向きのみを持つロケータを表す構造体
struct FLocator
{
    FMyVec3 Position{};
    FMyVec3 Forward{};

    friend void SerializeForLog(FCbWriter& Writer, const FLocator& Locator);
};
```

```cpp: Locator.cpp
void SerializeForLog(FCbWriter& Writer, const FLocator& Locator)
{
    Writer.BeginObject();
    Writer << ANSITEXTVIEW("Position") << Locator.Position;
    Writer << ANSITEXTVIEW("Forward") << Locator.Forward;
    Writer.EndObject();
}
```

所有するメンバ型が`FCbWriter`に対応しているなら、そのまま委譲するだけでよいです。

サンプルの多様性のため`operator<<` を使用してみましたが、
`Writer::AddField`でもいいです。どっちも同じです。

```cpp: Locator.cpp
FCbWriter SerializeForLog(FCbWriter& Writer, const FLocator& Locator)
{
    Writer.BeginObject();
    Writer.AddField(ANSITEXTVIEW("Position"), Locator.Position);
    Writer.AddField(ANSITEXTVIEW("Forward"), Locator.Forward);
    Writer.EndObject();
}
```


## 例3: ValueObjectパターン
値オブジェクトは意味次第ではログ対応すると嬉しいです。

```cpp: .h
class FCbWriter;

// 角度[deg]を表す値オブジェクト
struct MYMODULE_API FAngleDegree
{
    int32 Value; //角度[deg]

    // UE_LOGFMT対応のためのFCbWriter用シリアライズ関数
    friend void SerializeForLog(FCbWriter& Writer, const FAngleDegree& Angle);
};
```

```cpp: .cpp
#include "Logging/StructuredLog.h"
#include "Serialization/CompactBinaryWriter.h"

void SerializeForLog(FCbWriter& Writer, const FAngleDegree& Angle)
{
    Writer.BeginObject();
    Writer.AddInteger(ANSITEXTVIEW("degree"), Angle.Value);
    Writer.EndObject();
}

void Test()
{
    FAngleDegree Angle{.Value = 180}; //180°
    UE_LOGFMT(LogTemp, Log, "{0}", Angle);
}
// {"degree": 180}
```

単位付き表示したいときはこんな感じ。


```cpp
// 日本円を表す値オブジェクト
struct FCurrencyYen
{
	uint64 Currency; //円

	FString ToString() const
	{
		return FString::Printf(TEXT("￥%llu"), Currency);
	}
};

void SerializeForLog(FCbWriter& Writer, const FCurrencyYen& Yen)
{
    Writer.BeginObject();
    Writer << ANSITEXTVIEW("Currency") << Yen.ToString();
    Writer.EndObject();
}

UE_LOGFMT(LogTemp, Log "{0}", FCurrencyYen(123));
// {"Currency": ￥123}
```

システム層で勝手に補助単位を付与するべきでないと思いますが、例ですのでご容赦ください。
構造化ログの目的は解析しやすくすることにあるので、 `＄`や`￥`を前提とするシステムもあるかもしれない。

## 例4： enum対応
`UE_LOGFMT`は`enum class`にも対応しています。

```cpp: enum.h
// ENUMを UE_LOGFMTに対応させるケース
UENUM()
enum class EMyErrorCode : uint8
{
    None,
    Success,
    Failed,

    Failed_InvalidParameter,
    Failed_NoResource,
};

// linkできるよう関数を定義する
MYMODULE_API void SerializeForLog(FCbWriter& Writer, const EMyErrorCode ErrorCode);
```

```cpp: enum.cpp
void SerializeForLog(FCbWriter& Writer, const EMyErrorCode ErrorCode)
{
    // 1. Enumを文字列としてシリアライズする例
    Writer.AddString(UEnum::GetValueAsString(ErrorCode));

    // 2. Enumを整数としてシリアライズする例
    // Writer.AddInteger(static_cast<int32>(ErrorCode));

    // 3. EnumをObjectとしてシリアライズする例
    // Writer.BeginObject();
    // Writer.AddInteger(ANSITEXTVIEW("ErrorCode"), static_cast<int32>(ErrorCode));
    // Writer.EndObject();
}

void Test()
{
    UE_LOGFMT(LogTemp, Log, "エラーが発生しました. エラーコード={ErrorCode}", EMyErrorCode::Failed_InvalidParameter);
    // 「エラーが発生しました. エラーコード=EMyErrorCode::Failed_InvalidParameter」と表示される
}
```
`enum`をログ出力するとき、数値にするか文字列にするか悩ましいところです。好きに設計してください。ただし、`enum`の`under_lying_type`が`uint16`だとしても `Writer.AddInteger`しかないので`int32`扱いされます。


## その他応用事例
基本的にはログ出力したいレコード型を対応しておくと幸せになれると思います。

* 対戦FPSのキルログ
* バトルイベント
* ゲーム成績
* 有償アイテムの購入履歴
* デバッグログ
* バグレポ

```cpp
// キルログ
struct FKillRecord
{
    FName InstigatorName; // 攻撃者
    FName VictimName; // 被害者
    FName WeaponName; // キルを取った武器名
};

void SerializeForLog(FCbWriter& Writer, const FKillRecord& KillRecord)
{
    Writer.BeginObject();
    Writer.AddName(ANSITEXTVIEW("Instigator"), KillRecord.InstigatorName);
    Writer.AddName(ANSITEXTVIEW("Victim"), KillRecord.VictimName);
    Writer.AddName(ANSITEXTVIEW("Weapon"), KillRecord.WeaponName);
    Writer.EndObject();
    // {Insitgator: アリス, Victim: ボブ, Weapon: ショットガン}
}
```

---
# 詳解
`UE_LOGFMT` は `template関数`と `constexpr`および `concept` を駆使したかなりハイカラな実装になっています。型付けされた関数オーバーロードを適切に呼び出すために、`template`関数をつかってシリアライズ関数の呼び出しを試みています。
[ADL:Argument-Dependent Lookup](https://en.cppreference.com/w/cpp/language/adl.html)による解決がなされることを期待しているのです。

```cpp
template<T>
void Write(FCbWriter& Writer, const void* Value)
{
    // 1. void SerializeForLog(FCbWriter&, const T&) 
    // もしくは
    // 2. void SerializeForLog(FCbWriter&, T) 
    // という関数呼び出しと解釈される。
    SerializeForLog(Writer, *(const T*)Value);
}
```

## ADLによる関数ルックアップ
ADLでは `SerializeForLog`関数に最もマッチするシグネチャをもつ関数を探しにいきます。
適切にリンクできる部分に定義されていればそれを使うように解決されます。

`FCbWriter`こと"`Compact Binary Writer`"がコア実装です。`FCbWriter`側でプリミティブ型や`FName`などの著名なUE組み込み型のサポートが実装されています。

`SerializeForLog`と `operator<<` はどちらか片方が実装されていればよいです。 ADLがある方を見つけてくれます。両方実装してもいいですが、先に見つかるのは`SerializeForLog`のようです。

ADLを使う以上、`SerializeForLog`の引数型は値渡し(`T`)でも参照渡し(`const T&`)でもどちらでもよいです。カスタム型の特性に応じて好きに選べます。以下のいずれかが実装されていたらOK。
* `void SerializeForLog(FCbWriter& Writer, const T Value);`
* `void SerializeForLog(FCbWriter& Writer, const T& Value);`
* `FCbWriter& operator<<(FCbWriter& Writer, const T Value);`
* `FCbWriter& operator<<(FCbWriter& Writer, const T& Value);`

迷ったら 参照渡しでよいでしょう。
実装も `Writer.Add**` のメソッドを呼ぶか `operator<<`で繋いでいくかは自由です。
強いて選ぶならば、 `FCbWriter::Add**` の方が呼び出し側で型を指定できるため小回りが利きそうです。

特にBlueprintが`uint32`を扱えないために泣く泣く`int32`で扱っている箇所で便利そうな匂いがします。

## FCbWriter::BeginObject/EndObject
こちらは構造化ログへのエントリに、オブジェクトとして追加するかどうかを決めます。`Json object`のようなものと理解しています。

### オブジェクトとして登録
```cpp: オブジェクトとして扱う場合
void SerializeForLog(FCbWriter& Writer, const FHoge& Value)
{
    Writer.BeginObject();
    Writer.AddInteger(ANSITEXTVIEW("フィールド名"), Value.IntValue);
    Writer.EndObject();
    // {フィールド名: 値} というログが出る
}
```

### 値として登録
オブジェクトではなく、値として扱う場合はフィールド名を指定せずに直接値を入れます。一回のみいれられます。
```cpp: 値として扱う場合
void SerializeForLog(FCbWriter& Writer, const FHoge& Value)
{    
    Writer.AddInteger(Value.IntValue);
    // 値がそのままログに出る
}
```

```cpp: ダメな例
void SerializeForLog(FCbWriter& Writer, const FHoge& Value)
{    
    // これはダメ。フィールドが複数あるならちゃんとフィールド名を指定しましょう
    Writer.AddInteger(Value.IntValue0);
    Writer.AddInteger(Value.IntValue1);

    // GOOD
    Writer.BeginObject();
    Writer.AddInteger(ANSITEXTVIEW("IntValue0"), Value.IntValue0);
    Writer.AddInteger(ANSITEXTVIEW("IntValue1"), Value.IntValue0);
    Writer.EndObject()
}
```

`FAngleDegree`のようにプリミティブ型のラッパーであり内部値をそのまま扱いたい場合は `BeginObject-EndObject`でくくらずに直接出力した方がいいこともあります。

一方で、ログの集計しやすさや読みやすさからオブジェクトとしてシリアライズして、
`{フィールド名:値}` のようにjsonライクなシリアライズが適切な場合もあります。

結局のところ、構造化ログは集計しやすさのために構造化しているのだから、録ったログの使い道からシリアライズ方法を選ぶのが良いと思いました。

# まとめ

* `UE_LOGFMT`すごく良い
* `UE_LOG`はフォーマット指定子の指定ミスが怖いから `UE_LOGFMT`使った方がいい
* `float`値の桁数が問題になる場合は `ToString`や`ToCompactString` を検討すること
* AIが生成したフォーマットから`%s`と`%d`をコードレビューするのは無理。見落とす
* `CLAUDE.md` や `copilot-instructions.md` に `UE_LOGFMT`を使うよう指示すると幸せ