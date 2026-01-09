---
title: "UE5:Unreal EngineのUE_LOGFMTについての研究"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ue5, cpp, unrealengine, unrealengine5]
published: false
---

# はじめに
Unreal Engine 5.2 より `UE_LOGFMT` が登場しました。
こちらのログ機構は構造化ログを出すのに便利です。
`%d`, `%s`のようなフォーマット指定子を使うことなく位置指定子 `{}`でType付けされたフォーマットを実行することができます。
つよつよなログ機構なので基本的に乗り換えた方がよいでしょう。

本稿では基本的な使い方のほかにカスタム型のLOGFMT対応について調べました。
かなり以前からある機能であるにもかかわらず、だれも具体的なカスタムclassへのLOGFMT対応について触れてくれません。
そのせいで 2026/01月現在であってもAIはちゃらんぽらんな回答を返します。

ならやるしかねぇよなぁ。

# UE_LOGFMT

[公式ドキュメント](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/logging-in-unreal-engine#ue-logfmt)

位置パラメータと名前付きパラメータの二通りに対応しています。

## 忙しい人のための使い方サンプル

```cpp
#include "Logging/StructeredLog.h"

void AActor::BeginPlay()
{
    UE_LOGFMT(LogTemp, Verbose, "BeginPlayだよ");

    TSoftObjectPtr<UObject> Ptr;
    UE_LOGFMT(LogTemp, Warning, "Failed to load asset:{Asset}", Ptr->GetSoftObjectPath());
}
```

## include と依存モジュール
```cpp
#include "Logging/StructeredLog.h"
```
をインクルードします。

Coreモジュールにいるので`Build.cs`へ追加の依存を足す必要はありません。

## 位置パラメータを使った使い方

`{0}`, `{1}` のように`{}`を位置指定子として指定できます。


```cpp: 数値指定
FName Name = TEXT("Foo");
int Value = 42;
UE_LOGFMT(LogTemp, Log, "Name={0}, Value={1}", Name, Value);
// "LogTemp Log Name=Foo, Value=42" とログ出力される
```

`{0}{1}`はさもパラメータパックのindexかのような見た目ですが、ただの名前でありindex順は関係ありません。
`{1}{0}`と順番を入れ替えても効果は変わらず、与えたパラメータの順です。

```cpp: 数値指定
FName Name = TEXT("Name");
int Value = 42;
UE_LOGFMT(LogTemp, Log, "Name={1}, Value={0}", Name, Value);
// "LogTemp Log Name=Foo, Value=42" とログ出力される
```

ただの名前ですので、位置パラメータでは任意の文字列でも使用可能です。

```cpp
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}", Name, Value);
// "LogTemp Log Name=Foo, Value=42" とログ出力される
```

引数名とはいっさい関係がないので、引数の順番を入れ替えるとログの内容がかわります。
```cpp
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}", Value, Name);
// "LogTemp Log Name=42, Value=Foo" とログ出力される
```

このとき`%s`のようなフォーマット指定子を使わないため、安全に文字列としてシリアライズされます。

## 名前付きパラメータを使った使い方

名前付きパラメータでは、引数として名前と値のタプルを与えます。
```cpp
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}", ("Name", Name), ("Value", Value));
// "LogTemp Log Name=Foo, Value=42" とログ出力される
```

名前指定できているのでパラメータの順番は入れ替えても正しく表示されます。
```cpp
UE_LOGFMT(LogTemp, Log, "Name={Name}, Value={Value}", {("Value", Value), ("Name", Name)});
// "LogTemp Log Name=Foo, Value=42" とログ出力される
```

### 同名禁止

パラメータ名はユニークである必要があります。TMapに格納されるためそれはそう。
```cpp
const FName ClientName = TEXT("Client");
const FName ServerName = TEXT("Server");
UE_LOGFMT(LogTemp, Log, "Server={Name}, Client={Name}", {("Name", ServerName), ("Name", ClientName)});
```

## `TEXT`マクロ不要
`UE_LOGFMT`では フォーマット文字列にたいして　TEXTマクロを使う必要がありません。UE_LOGFMTマクロ内でTEXTマクロが付与されるため、つけてしまうとコンパイルエラーになってしまいます。

```cpp
// old style
UE_LOG(LogTemp, Log, TEXT("日本語でおｋ"));

// new style
UE_LOGFMT(LogTemp, Log, "日本語でおｋ");
```

printfスタイルに近づいてより読みやすくわかりやすくなったのではないでしょうか？

## 組み込み型対応済み

UE_LOGFMTは UE組み込み型にかなり対応してくれています。

* int
* float
* double
* bool
* uint8
* char*

* FString
* FName
* FGuid

* `TArray<T>`
* `TMap<T>`

* `TObjectPtr<T>`

```cpp

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

TEXTがいるせいで、閉じ括弧`)`の数とかよくわからんですね。


UE_LOGFMTでは型付けされてシリアライズされるのでこれが解決されています。
`FString`も`FName`も直接与えてよいです。

```cpp
const FString Str = TEXT("エラーが発生しました");
UE_LOG(LogTemp, Log, "{0}", Str);

const int errorCode = -1;
UE_LOG(LogTemp, Log, "{0}", FString::Format(TEXT("エラーが発生しました:%d"), errorCode));
```

激熱ですねぇ。

## セキュリティ対応
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
そのため、原理的にこれができなくなっておりコンパイルエラーになります。

```cpp
void ParseUserInput(const char* Buffer, int Length)
{
    // できないよ
    UE_LOGFMT(LogTemp, Log, Buffer);

    // 安全
    UE_LOGFMT(LogTemp, Log, "{0}", Buffer);
}
```

## フォーマット指定子の指定ミスでクラッシュしない
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

FVector等が非対応なのはおそらく出力精度を指定できないからです。
ToString() や ToCompactString() 等を使え、ということなのでしょう。（知らんけど）

```cpp
const FVector Location = Actor->GetActorLocation();

// コンパイルエラー!
UE_LOGFMT(Logtemp, Log "Position:{Pos}", Location);

// Positon:(1.00, 2.01, 3.02) のように表示される
UE_LOGFMT(Logtemp, Log "Position:{Pos}", Location.ToString());

// Positon:(1.00, 2.01, 3.02) のように表示される
UE_LOGFMT(Logtemp, Log "Position:{Pos}", Location.ToCompactString());

// Positon:(1.000000001, 1.99999998f, 3.00000001f) double精度表示される
UE_LOGFMT(Logtemp, Log "Position:{X} {Y} {Z}", Location.X, Location.Y, Location.Z);

```

# カスタム型を UE_LOGFMT対応させる

本題です。ユーザー定義のclass/struct を UE_LOGFMTに対応させることができます。

Serialize関数を定義するだけなのですが、その実装方法がよくわかりません。
ここでまとめました。

ユーザー型はなんでもよいのですが、いくつか例示します。
```cpp
struct FMyVec3
{
    float X;
    float Y;
    float Z;
}
```

```cpp: MyVec3.h
struct FMyVec3
{
    float X;
    float Y;
    float Z;

    friend FCbWriter Serialize(FCbWriter& Writer, const FMyVec3& Vec3)
}
```

```cpp: MyVec3.cpp
FCbWriter Serialize(FCbWriter& Writer, const FMyVec3& Vec3)
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
UE_LOGFMT(LogTemp, Log, "{}", Vec);
// {X:2, Y:3, Z:4}
```

## 応用編
さらにカスタム型をもつカスタム型をUE_LOGFMTに対応させてみましょう。

```cpp: Locator.h
// 位置と向きのみを持つロケータを表す構造体
struct FLocator
{
    FMyVec3 Position;
    FMyVec3 Foward;

    friend FCbWriter Serialize(FCbWriter& Writer, const FLocator& Locator)
}
```

```cpp: Locator.cpp
FCbWriter Serialize(FCbWriter& Writer, const FLocator& Locator)
{
    Writer.BeginObject();
    Writer << ANSITEXTVIEW(Position) << Locator.Position;
    Writer << ANSITEXTVIEW(Foward) << Locator.Forward;
    Writer.EndObject();
}
```

所有するメンバ型が`FCbWriter`に対応しているなら、そのまま委譲するだけでよいです。

## ValueObjectパターンによき
値オブジェクトは意味次第ではログ対応すると嬉しいです。

```cpp
struct FCurrencyYen
{
    int64 Currency; //円

    FString ToString()
    {
        return FString::Format("￥%d");
    }
}

FCbWriter Serialize(FCbWriter& Writer, const FCurrencyYen& Yen)
{
    Writer.BeginObject();
    Writer << ANSITEXTVIEW(Currency) << Yen.ToString();
    Writer.EndObject();
}

UE_LOGFMT(LogTemp, Log "{}", FCurrencyYen(123));
// {Currency: ￥123}

```

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
    // {Insitgator: 太郎, Victim: 次郎, Weapon: ショットガン}
}
```