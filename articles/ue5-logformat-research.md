---
title: "UE5:Unreal EngineのUE_LOGFMTについての研究"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: false
---

# はじめに
Unreal Engine 5.2 より `UE_LOGFMT` が登場しました。
`%d`, `%s`のようなフォーマット指定子を使うことなく位置指定子 `{}`でType付けされたフォーマットを実行することができます。
つよつよなログ機構なので`UE_LOG`の上位互換となることができます。乗り換えた方がよいのではないでしょうか。

本稿では`UE_LOGFMT`の基本的な使い方のほかにカスタム型のLOGFMT対応について調べました。
かなり以前からある機能であるにもかかわらず、だれも具体的なカスタムclassへのLOGFMT対応について触れてくれません。
そのせいで 2026/01月現在であってもAIはちゃらんぽらんな回答を返します。

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

## 位置パラメータを使った使い方

`{0}`, `{1}` のように`{}`を位置指定子として指定できます。
この位置指定子に与える名前をログフィールド名と言います。
例えば `{0}` としたときのそのフィールド名は`0`です

フィールド名は`unique non-empty` 名であることが要求されます。


```cpp: 位置指定
FName Name = TEXT("Foo");
int Value = 42;
UE_LOGFMT(LogTemp, Log, "Name={0}, Value={1}", Name, Value);
// "LogTemp Log Name=Foo, Value=42" とログ出力される
```

### ログフィールド名と引数の対応関係
フィールドの登場順と引数の順番が対応しています。名前は一切関係がないことに注意が必要です。

`{0}`と`{1}`はさもパラメータパックのindexかのような見た目ですが、ただの名前でありindexではありません。
`{1}`と`{0}`の順番を入れ替えても効果は変わらず、与えた引数の順です。注意してください。

入れ替えた事例です。
```cpp: 数値指定
FName Name = TEXT("Name");
int Value = 42;
UE_LOGFMT(LogTemp, Log, "Value={1}, Name={0}", Name, Value);
// "Value=Foo, Name=42" とログ出力される。期待と異なり入れ替わらない
```

ログの出力結果は変わりません。
ただの名前ですので、位置パラメータでは任意の文字列でも使用可能です。

ログフィールド名と実引数名はいっさい関係がありません。順番のみが考慮されます。

```cpp: 引数の順番間違えた例
// Name, Valueと与えるべきところをValue,Nameと与えている！
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}", Value, Name);
// "Name=42, Value=Foo" とログ出力される
```

このときログの意味は壊れていますが、`%s`のようなフォーマット指定子を使わないため、安全にシリアライズされます。クラッシュしません。最悪、ログが出ないかログがヘンテコになるだけでクラッシュしないのは助かります。

もしログフィールド名と実引数を対応付けしたい場合は、後述の名前付きパラメータを利用します。

## 名前付きパラメータを使った使い方

名前付きパラメータでは、引数に名前と値のタプルを与えます。
タプルの0番目にはフォーマット文字列で指定したフィード名を、タプルの1番目にはログ出力したい値を与えます。

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

可読性の観点や指定順のミスを防ぎやすい点に優れています。
手で書くと面倒くさいのが玉に瑕ですが、AI Agentに実装させるときはこちらの方が安全かと思います。面倒くささもないですしね。

### 同名禁止

パラメータ名はユニークである必要があります。内部的にTMapに格納されるためです。

```cpp
const FName ClientName = TEXT("Client");
const FName ServerName = TEXT("Server");
// 実行時エラー checkfで止まる
UE_LOGFMT(LogTemp, Log, "Server={Name}, Client={Name}", {("Name", ServerName), ("Name", ClientName)});

// 正しくはこうしてほしい
UE_LOGFMT(LogTemp, Log, "Server={ServerName}, Client={ClientName}", {("ServerName", ServerName), ("ClientName", ClientName)});
```

## `TEXT`マクロ不要
`UE_LOGFMT`では フォーマット文字列にたいして　TEXTマクロを使う必要がありません。UE_LOGFMTマクロ内でTEXTマクロが付与されるため、つけてしまうとコンパイルエラーになってしまいます。

```cpp
// old style
UE_LOG(LogTemp, Log, TEXT("日本語でおｋ"));

// new style
UE_LOGFMT(LogTemp, Log, "日本語でおｋ");
```

printfスタイルに近づいてより読みやすく書きやすくなったのではないでしょうか。

## 組み込み型対応済み

UE_LOGFMTは UE組み込み型にかなり対応してくれています。

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


```message
`FVector`, `FTransform`などの数学系構造体が非対応です
```

## FString の デリファレンス不要
UE_LOG では `%s`に与える文字列は`FString::operator*()` を用いてc-style文字列として与える必要がありました。

```cpp
const FString Str = TEXT("エラーが発生しました");
UE_LOG(LogTemp, Log, TEXT("%s"), *Str);
```

とくに FString::Formatを使うと汚くなっていました。
```cpp
const int errorCode = -1;
UE_LOG(LogTemp, Log, TEXT("%s"), *FString::Format(TEXT("エラーが発生しました:%d"), errorCode));
```

TEXTがいるせいで、閉じ括弧`)`の数がよくわからんです。


UE_LOGFMTでは型付けされてシリアライズされることによりこの不満点が解決されています。
`FString`も`FName`も直接与えてよいです。

```cpp
const FString ErrorMessageStr = TEXT("ストレージが不足しています");
UE_LOG(LogTemp, Log, "エラーが発生しました {0}", ErrorMessageStr);

const int errorCode = -1;
UE_LOG(LogTemp, Log, "{0}", FString::Format(TEXT("エラーが発生しました:%d"), errorCode));
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


`UE_LOGFMT` では型付けされたシリアライズを実行するためフォーマット文字列を必ず要求します。
constexpr と static_assertを駆使することでこれができなくなっておりコンパイルエラーになります。

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

さて、このバグは市場で発火します。なぜなら開発環境では正しいデバイスが常に存在するからです。
ユーザーからは特定の環境でゲームがクラッシュする、という報告がきます。どうやらDeviceがないときに発生するようですね。
デバイスの接続を確認するとかデバイス設定を見直すとかドライバを更新するとか、検討違いの対策を練り始めてしまいます。

真因は違います。エラーが原因ではなくエラーハンドリングが原因なのです。
しかも、大抵が滅多に通らない（そしてテストされない）パスで発生します。よって市場へリリースされてしまうのです。
このように真の原因を隠蔽してしまう恐れがあるため、フォーマット指定子の指定ミスは恐ろしいのです。

なお、未定義動作なのでクラッシュするとは限りません。クラッシュすれば御の字で、生動きする可能性は大いにあります。未定義動作ですから。





## 指定子と引数の数が一致しなくても無視される
UE_LOGFMTでは、引数と指定子の数が一致していないとき正しく無視されます。(多分)

```cpp: 引数が多いとき
UE_LOGFMT(LogTemp, Log, "{0}", 1, 2);
// 1 が表示される
```

```cpp: 引数が少ないとき
UE_LOGFMT(LogTemp, Log, "{0} {1}", 1);
// 1 が表示される
```

## 対応していない型
数学系構造体が対応していません。肝心のFVector系がないのはつらい

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

これらは`ToString()` や `ToCompactString()` 等を使ってしのぎます。

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

# カスタム型を UE_LOGFMT対応させる

本題です。`UE_LOGFMT`ではユーザー定義のclass/struct を UE_LOGFMTに対応させることができます。

Serialize関数を定義するだけなのですが、その実装方法がよくわかりませんでしたので、本稿でまとめました。

ユーザー型はなんでもよいのですが、いくつか例示します。

## 例1: Vec3型
FVectorが非対応なのでオレオレVec3を構造化ログ対応してみます。

```cpp: MyVec3.h
class FCbWriter;

struct FMyVec3
{
    float X = 0;
    float Y = 0;
    float Z = 0;

    friend void SerializeForLog(FCbWriter& Writer, const FMyVec3& Vec3)
}
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
さらにカスタム型をもつカスタム型をUE_LOGFMTに対応させてみましょう。

```cpp: Locator.h
// 位置と向きのみを持つロケータを表す構造体
struct FLocator
{
    FMyVec3 Position{};
    FMyVec3 Forward{};

    friend FCbWriter SerializeForLog(FCbWriter& Writer, const FLocator& Locator)
}
```

```cpp: Locator.cpp
FCbWriter SerializeForLog(FCbWriter& Writer, const FLocator& Locator)
{
    Writer.BeginObject();
    Writer << ANSITEXTVIEW("Position") << Locator.Position;
    Writer << ANSITEXTVIEW("Forward") << Locator.Forward;
    Writer.EndObject();
}
```

所有するメンバ型が`FCbWriter`に対応しているなら、そのまま委譲するだけでよいです。

ちなみに サンプルの多様性のため`operator<<` を使用してみましたが、
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
}
```

```cpp: .cpp
#include "Logging/StructuredLog.h"
#include "Serialization/CompactBinaryWriter.h"

void SerializeForLog(FCbWriter& Writer, const FAngleDegree& Angle)
{
   	Writer.BeginObject();
	Writer.AddFloat(ANSITEXTVIEW("degree"), Angle.Value);
	Writer.EndObject();
}

void Test()
{
    FAngleDegree Angle{.Value = 180}; //180°
    UE_LOGFMT(LogTemp, Log "{}", Degree);
}
// 180
```

単位付き表示したいときはこんな感じ。


```cpp
// 日本円を表す値オブジェクト
struct FCurrencyYen
{
    uint64 Currency; //円

    FString ToString()
    {
        return FString::Format("￥%llu");
    }
}

FCbWriter Serialize(FCbWriter& Writer, const FCurrencyYen& Yen)
{
    Writer.BeginObject();
    Writer << ANSITEXTVIEW("Currency") << Yen.ToString();
    Writer.EndObject();
}

UE_LOGFMT(LogTemp, Log "{0}", FCurrencyYen(123));
// {"Currency": ￥123}
```

システム層で勝手に補助単位文字を付与するべきでないので、例として微妙だったかもしれません。

## 例4： enum対応
`UE_LOGFMT` は enum classにも対応しています。

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

// linkできるよう関数を定義すればよい
MYMODULE_API void SerializeForLog(FCbWriter& Writer, const EMyErrorCode ErrorCode);
```

```cpp: enum.cpp
void SerializeForLog(FCbWriter& Writer, const EMyErrorCode ErrorCode)
{
	// Enumを文字列としてシリアライズする例
	Writer.AddString(UEnum::GetValueAsString(ErrorCode));
	
	// Enumを整数としてシリアライズする例
	//Writer.AddInteger(static_cast<int32>(ErrorCode));
}

void Test()
{
    UE_LOGFMT(LogTemp, Log, "エラーが発生しました. エラーコード={ErrorCode}", EMyErrorCode::Failed_InvalidParameter);
    // 「エラーが発生しました. エラーコード=EMyErrorCode::Failed_InvalidParameter」と表示される
}
```
enumをログ出力するとき、数値にするか文字列にするか悩ましいところですが、設計してください。


## その他応用事例
何でもかんでも対応すればいいという訳ではありません。基本的にはログ出力したいレコード型を対応しておくと幸せになれると思います。

* 対戦FPSのキルログ (プレーヤーAがプレーヤーBを武器Cで倒した)
* バトルイベント
* ゲーム成績
* 有償アイテムの購入履歴
* デバッグログ

```cpp
// キルログ
struct FKillRecord
{
    FName InstigatorName; // 攻撃者
    FName VictimName; // 被害者
    FName WeaponName; // キルを取った武器名
}

FCbWriter Serialize(FCbWriter& Writer, const FKillRecord& KillRecord)
{
    Writer.BeginObject();
    Writer.AddName(ANSITEXTVIEW("Instigator"), KillRecord.InstigatorName);
    Writer.AddName(ANSITEXTVIEW("Victim"), KillRecord.VictimName);
    Writer.AddName(ANSITEXTVIEW("Weapon"), KillRecord.WeaponName);
    Writer.EndObject();
    // {Insitgator: アリス, Victim: ボブ, Weapon: ショットガン}
}
```

# 詳解
`UE_LOGFMT` は `template関数`と `constexpr`および `concept` を駆使したかなりハイカラな実装になっています。型付けされた関数オーバーロードを適切に呼び出すために、template関数をつかってシリアライズ関数の呼び出しを試みています。
[ADL:Argument-Dependent Lookup](https://en.cppreference.com/w/cpp/language/adl.html)による解決がなされることを期待しているのです。

```cpp
template<T>
void Write(FCbWriter& Writer, const void* Value)
{
    // void SerializeForLog(FCbWriter&, const T&) という関数呼び出し
    SerializeForLog(Writer, *(const T*)Value);
}
```

ADLでは SerializeForLog関数に最もマッチするシグネチャをもつ関数を探しにいきます。
適切にリンクできる部分に定義されていればそれを使うように解決されます。

`FCbWriter`こと"`Compact Binary Writer`"がコア実装です。`FCbWriter`側でプリミティブ型や`FName`などの著名なUE組み込み型のサポートが実装されています。

`SerializeForLog`と `operator<<` はどちらか片方が実装されていればよいです。 ADLがある方を見つけてくれます。両方実装してもいいですが、先に見つかる `SerializeForLog`が優先されるようです。
完全に同じ意味を持つ関数を２つ実装するのはDRY原則違反なので、片方でよいでしょう。

値渡しでも参照渡しでもどちらでもよいです。カスタム型の特性に応じて参照渡しにするもよし、値渡しにしてもよいです。
迷ったら 参照渡しでよいと思います。

以下のいずれかが実装されていたらOK。
* `void SerializeForLog(FCbWriter& Writer, const T Value);`
* `void SerializeForLog(FCbWriter& Writer, const T& Value);`
* `FCbWriter& operator<<(FCbWriter& Writer, const T Value);`
* `FCbWriter& operator<<(FCbWriter& Writer, const T& Value);`

実装も `Writer.Add**` のメソッドを呼ぶか `operator<<`で繋いでいくかは自由です。
しいて選ぶならば、 `FCbWriter::Add**` の方が呼び出し側で型を指定できるため小回りが利きそうです。

特にBPがuint32を扱えないために泣く泣くint32で扱っている箇所で便利そうな匂いがします。

## FCbWriter::BeginObject
こちらは構造化ログへのエントリに、オブジェクトとして追加するかどうかを決めます。JsonObjectのようなものと理解しています。

### オブジェクトとして登録
```cpp: オブジェクトとして扱う場合
void SerializeForLog(FCbWriter& Writer, const FHoge& Value)
{
    Writer.BeginObject();
    Writer.AddInteger(ANSITEXTVIEW("<フィールド名>"), Value.IntValue);
    Writer.EndObject();
}
```

### 値として登録
オブジェクトではなく、値として扱う場合はフィールド名を指定せずに直接値を入れます。一回のみいれられます。
```cpp: 値として扱う場合
void SerializeForLog(FCbWriter& Writer, const FHoge& Value)
{    
    Writer.AddInteger(Value.IntValue);
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

一方で、ログの集計しやすさや読みやすさからオブジェクトとしてシリアライズして、`{フィード名:値}` のようにjsonライクなシリアライズが適切な場合もあります。

結局のところ、構造化ログは集計しやすさのために構造化しているのだから、録ったログの使い道からシリアライズ方法を選ぶのがええんではないでしょうか。

## まとめ

* `UE_LOGFMT`すごく良い
* `UE_LOG`はフォーマット指定子の指定ミスが怖いから `UE_LOGFMT`使った方がいい
* 値の精度が問題になる場合は `ToString`や`ToCompactString` を検討すること
* AIが生成したフォーマットから`%s`と`%d`をコードレビューするのは無理。見落とす
* CLAUDE.md や copilot-instructions.md に `UE_LOGFMT`を使うよう指示すると幸せ