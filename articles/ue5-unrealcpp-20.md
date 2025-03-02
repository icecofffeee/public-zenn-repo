---
title: "UE5:Unreal C++20 についての研究"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ue5", "cpp"]
published: false
---

# Unreal C++ とは
ゲームエンジンであるUnreal Engine における C++の特殊な使い方を指した便宜上の呼称です。
Unreal Engine はC++で実装されていますが、`UCLASS`, `UPROPERTY`といった独自のマクロやリフレクションシステムなどを備えており非常に特殊な記述が求められます。Unreal Engine のお作法に則った特殊なC++実装が必要とされるということさして、Unreal C++という呼び方がなされます。非公式の呼称です。


# 背景
Unreal Engine 5.3 にて C++20 対応がなされました。C++20の機能が全部つかえるぞぉということはなく、Unreal C++ に適合した書き方が必要となってきます。
本稿ではどのようにすればモダンなUnreal C++が実装できるかの研究成果を発表します。

# constexpr
C++20にて `constexpr` の制限が概ね取り払われてとても直感的に`constexpr if` を記述することが可能になりました。
型trait群では、std名前空間にあるものの他、UnreanEngine用のtraitがいくつか用意されています。

## constexpr コンストラクタ
いくつかの組み込み型において constexprコンストラクタが実装されています。
既存の実装を壊さないように少し工夫されているため、意図的に使わないと気づけません。なおほとんどの型にはconstexpr コンストラクタが実装されていないため中々使いどころがありません。

対応しているものは以下の通りです。
```cpp
constexpr int32 Zero = 0; //標準的なものは全部大丈夫
constexpr int32 Supported[] = { 48000, 441000}; //標準的なものは全部大丈夫
constexpr uint32 Max = TNumericLimits<uint32>::Max(); // constexpr関数も使える
constexpr float value = 1.0f;
constexpr bool bIsDebugBuild = false;
constexpr TCHAR SupportedLetter[] = TEXT("abcdefghijklmnopqrstuvwxyz"); //固定長配列
constexpr const TCHAR* Message[] = TEXT("teisuu string");//文字列リテラル
```

UENUM使えます。
```cpp
UENUM()
enum class EHoge : uint8
{
    Hoge (UMETA(DisplayName = "HOGE")),
}
constexpr EHoge ConstExprHoge = EHoge::Hoge; // enum class, UENUMもOK
constexpr EObjectFlags NoSaveFlags = RF_Transient | RF_DuplicateTransient; // ただのenum もok
```

FVectorだけなぜか一部対応していますが、ゼロベクトルぐらいしか作れません。
残念ながら単位ベクトルは作れません。

```cpp
// UE::Math::TVector 型は第2引数にTVectorConstInitを与えるとconstexprコンストラクタになる 
constexpr FVector ZeroVector(0, UE::Math::TVectorConstInit{});
constexpr FVector OneVector(1, UE::Math::TVectorConstInit{}); 

//コンパイルエラー. こんなコンストラクタはない
constexpr FVector UpVector = FVector(0,1,0,UE::Math::TVectorConstInit{});
```

一部構造体は対応しています。
```cpp
constexpr FLinearColor MyRed = FLinearColor(0.89f, 0, 0);
constexpr FAsciiSet Number = "0123456789";
static inline constexpr FGuid TelemetryID = FGuid(0xdeadbeaf, 0xdeadbeaf, 0xdeadbeaf, 0xdeadbeaf);
```
特筆すべきは`FStringView`です。文字列リテラルをコンパイル時定数として引き回せるようになりました。`TEXTVIEW`マクロを使うことで得られます。`std::string_view`と同じ感じで使うことができます。

```cpp
static constexpr FStringView kDefaultMessage = TEXTVIEW("Message Dayo");
```

ただし、utf8文字列リテラルはUE5.5時点では未対応です。
`UTF8TEXTVIEW` マクロはあるのですがconstexprにはできません。c++20の`char8_t`に対応されるまでダメらしいです。

## constexprにできない型

TArray, TMap, など 定数式にしたいやつらがいますができません。
定数テーブルを作りたいのですが、まだ早いです。

```cpp
constexpr TArray<int> DataTable = {0, 1, 2, 3}; //ダメ
```

# constinit
初期化問題を回避するためには constinit は有効です。static initialization order fiascoとして有名です。
特に外部リンケージを持つ extern なグローバル変数などで初期化を保障できるのはありがたいです。
constinit は 定数初期化コンストラクタができないといけません。

```cpp
constinit extern int GFrameCount = 0; // 0初期化される
constinit extern const int kMaxSize = 1024; // これは必ず初期化されるconst変数 (定数ではない)
constexpr int MAX_SIZE = 1024; // これはコンパイル時定数
constinit FVector GlobalVector = FVector(0, UE::Math::TVectorConstInit);

static constinit FMySingleton* pSingleton = nullptr;

void Main()
{
    GFrameCount++; // const intではないので書き換えられる
    GlobalVector.X++;// constinitは初期化するだけ、const FVectorではないので書き換えられる

    std::array<int, MAX_SIZE> arr; // OK. コンパイル時定数
}
```

constinit はあくまで初期化をコンパイル時に保障してくれる機能であって、定数化するものではないことに注意。
constと constexpr と constinit の違いをそれぞれ説明できないと使いこなすことは難しい。

結論：
mutable なグローバル変数は constinit で良いと思います。public で mutable な static変数も定数初期化を実装して constinit static で定義するのが良いと思います。
とはいえ、グローバル変数はそうそう使うべきでないし、static変数はクソなので、前提からお勧めしません。
グローバルなアクセスが行いたいだけなら、UEなんだからWorldSubsystemとか使えばいいじゃない。

結果使いどころはほとんどないと思います。


# FStringView
C++に存在する`std::string_view`のUnrealC++版が `FStringView`です。
c++17から使えてましたがconstexpr コンストラクタが使えるので言及しておきます。

文字列を引数に受けるときは可能ならば`FStrinView`を使うようにすると統一的なAPIを実装できます。
使い方はほぼほぼ `std::string_view`と同じです。
implicit operator が実装されているので暗黙的に各種文字列型から変換されます。
関数呼び出しを行う側はとくに FStringViewであることを意識せず文字列を渡せるでしょう。

FStringViewは軽量な構造体なので値渡しで十分です。下手に参照渡しにするよりも値渡しの方がいいです。
```cpp
// dirname コマンドみたいな関数
FStringView GetDirectoryName(const FStringView InStr)
{
    int Index = 0;
    if(InStr.FindLastChar(TEXT('/'), Index))
    {
        FStringView SubStr = InStr.SubStr(0, Index);
        return SubStr;
    }
}

void Use()
{
    constexpr FStringView Path = TEXTVIEW("Game/Map/MyLevel");
    FStringView DirName = GetDirectoryName(Path);

    FStringView DirName = GetDirectoryName(Path);
}
```
このように一切の文字列コピーを挟まずに部分文字列を指す区間を得ることができました。
残念ながら FindLastCharといった関数がconstexprでないので、コンパイル時に定めることは中々できません。

また LogやPrintfに渡したいときは直接は無理なのでいったんFStringにします。

# FString::Format
`std::format` がC++20でお目見えしました。それのUnrealC++版です。
やっとC#やpythonのように`{}`によるフォーマット指定子が使えるようになりました。
これで文字列を`%d`で読み込んでクラッシュするような事件が減りますね。

ただし、UnrealC++なので使い方が少々特殊です。

```cpp
int hour = 12;
int minute = 34;
int second = 56;
FString String = FString::Format(TEXT("Timestamp {0}:{1}:{2}"), {hour, minute, second});// "Timestamp 12:34:56" 
FString String = FString::Format(TEXT("Timestamp {0}:{1}:{2}"), hour, minute, second);// コンパイルエラー {}で括るべし
```

第1引数にはフォーマット文字列を与えます。第2引数は {}で括ってあげる必要があります。
これは`FString::Format`はパラメータパックを受け取らないからです。
第2引数を `FStringFormatOrderedArguments`型にしないといけません。こいつの正体は `TArray<FStringFormatArg>`型です。
{}初期化による暗黙的なコンストラクタ呼び出しで対応できます。

名前指定フォーマットも可能です。

```cpp
FStringFormatNamedArgument Args; //ただの TMap<FString, FStringFormatArg>
Args.Add(TEXT("Hour"), 12);
Args.Add(TEXT("Minute"), 34);
Args.Add(TEXT("Second"), 56);
FString String = FString::Format(TEXT("Timestamp {Hour}:{Minute}:{Second}"), Args);// "Timestamp 12:34:56" 
```

`FStringFormatNamedArgument`型を使います。こいつの正体は`TMap<FString, FStringFormatArg>`型です。

残念ながらFStingFormatArgの制限により受付可能な型が非常に少ないです。
プリミティブ型とFString, FStringViewなど文字列型だけです。
FVectorなどの著名な型は直接入れられないので、ToStringで変換してあげる必要があります。

```cpp
FVector Vec(1,2,3);
FString String = FString::Format(TEXT("Vec {0}"), {Vec.ToString()});
FString String = FString::Format(TEXT("Vec {0}"), {Vec});//コンパイルラー

```

# consteval 
進捗だめです。

constevalは実質使えません。
`UE_CONSTEVAL`が存在しますが、一部のコンパイラのみ対応しています。
ゲームエンジンとしては全てのコンパイラに対応していないとマルチプラットフォームが保障できなくなるので、封印されています。
