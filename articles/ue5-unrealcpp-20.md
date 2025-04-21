---
title: "UE5:Unreal C++20 についての研究"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ue5", "cpp"]
published: true
---

# Unreal C++ とは
ゲームエンジンであるUnreal Engine における C++の特殊な使い方を指した便宜上の呼称です。
Unreal Engine はC++で実装されていますが、`UCLASS`, `UPROPERTY`といった独自のマクロやリフレクションシステムなどを備えており非常に特殊な記述が求められます。Unreal Engine のお作法に則った特殊なC++実装が必要とされるということさして、Unreal C++という呼び方がなされます。非公式の呼称です。


# 背景
Unreal Engine 5.3 にて C++20 対応がなされました。C++20の機能が全部つかえるぞぉということはなく、Unreal C++ に適合した書き方が必要となってきます。
本稿ではどのようにすればモダンなUnreal C++が実装できるかの研究成果を発表します。

# 検証環境
UE 5.5.4
Visual Studio 2022 (17.10)
MSVC v143 x64/x86 ビルドツール v14.38-17.8
Windows 11 (x64)

# constexpr
C++20にて `constexpr` の制限が概ね取り払われてとても直感的に`constexpr if` を記述することが可能になりました。
型trait群では、std名前空間にあるものの他、UnrealEngine用のtraitがいくつか用意されています。

## constexpr コンストラクタ
UEのいくつかの組み込み型において `constexpr`コンストラクタが実装されています。
既存の実装を壊さないように少し工夫されているため、意図的に使わないと気づけません。なおほとんどの型には`constexpr` コンストラクタが実装されていないため中々使いどころがありません。

対応しているものは以下の通りです。
```cpp
constexpr int32 Zero = 0; //プリミティブ型は全部大丈夫
constexpr int32 SupportedFrequency[] = { 48000, 44100}; //配列もok
constexpr uint32 Max = TNumericLimits<uint32>::Max(); // constexpr関数も使える
constexpr float value = 1.0f;
constexpr TCHAR SupportedLetter[] = TEXT("abcdefghijklmnopqrstuvwxyz"); //固定長配列
constexpr const TCHAR* Message = TEXT("teisuu string");//文字列リテラル
#if _DEBUG
constexpr bool bIsDebugBuild = true; // constexpr ifで使用できる
#else
constexpr bool bIsDebugBuild = false; // constexpr ifで使用できる
#endif
```

何気に 配列サイズの推論の強化もされています。

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

`TVector<T>`だけ一部対応しており、`FVector`等の基幹クラスで`constexpr`が使えるのですが、ゼロベクトルぐらいしか作れません。残念ながら単位ベクトルは作れません。UE5.5.3の時点では引数指定型の`constexpr`コンストラクタが実装されていないからです。

```cpp
// UE::Math::TVector 型は第2引数にTVectorConstInitを与えるとconstexprコンストラクタになる 
constexpr FVector ZeroVector(0, UE::Math::TVectorConstInit{});
constexpr FVector OneVector(1, UE::Math::TVectorConstInit{}); 

//コンパイルエラー. こんなコンストラクタはない
//なぜ定義してくれなかったのだ...
constexpr FVector UpVector = FVector(0,1,0,UE::Math::TVectorConstInit{});
```

その他に一部構造体は`constexpr`対応しています。
```cpp
constexpr FLinearColor MyRed = FLinearColor(0.89f, 0, 0);
constexpr FAsciiSet Number = "0123456789";
static inline constexpr FGuid TelemetryID = FGuid(0xdeadbeaf, 0xdeadbeaf, 0xdeadbeaf, 0xdeadbeaf);
```
特筆すべきは`FStringView`です。文字列リテラルをコンパイル時定数として引き回せるようになりました。`TEXTVIEW`マクロを使うことで得られます。`std::string_view`と同じ感じで使うことができます。

```cpp
static constexpr FStringView kDefaultMessage = TEXTVIEW("Message Dayo");
```

ただし、utf8文字列リテラルはUE5.5.3時点では未対応です。`UTF8TEXTVIEW` マクロはあるのですが`constexpr`にはできません。c++20の`char8_t`に対応されるまでダメらしいです。


## constexprにできない型

`TArray`, `TMap` など 定数式にしたいやつらがいますができません。
コンパイル時定数テーブルを作りたいのですが、まだ早いです。残念。

```cpp
constexpr TArray<int> DataTable = {0, 1, 2, 3}; //ダメ
```

# constinit
`constinit`とはコンパイル時初期化です。`constexpr` がコンパイル時定数であるのに対して、こちらは`mutable`な変数に使用できます。

`constinit`は初期化問題を回避するために有効です。静的変数の初期化順に伴うバグは`static initialization order fiasco`として有名です。特に外部リンケージを持つ `extern` なグローバル変数などの初期化を保障できるのはありがたいです。

`constinit` は 定数初期化コンストラクタができないといけません。UE5.5時点では定数初期化コンストラクタを持つstructが中々存在しないので、使いどころは少ないですが、
使える場合は`constinit`を使った方が安全でしょう。

```cpp
constinit extern int GFrameCount = 0; // 0初期化される
constinit extern const int kMaxSize = 1024; // これは必ず初期化されるconst変数 (定数ではない)
constexpr int MAX_SIZE = 1024; // これはコンパイル時定数
constinit FVector GlobalVector = FVector(0, UE::Math::TVectorConstInit);

static constinit FMySingleton* pSingleton = nullptr;

void Main()
{
    GFrameCount++; // constinitは初期化するだけ、const intではないので書き換えられる
    GlobalVector.X++;// constinitは初期化するだけ、const FVectorではないので書き換えられる

    std::array<int, MAX_SIZE> arr; // OK. コンパイル時定数
}
```

`constinit` はあくまで初期化をコンパイル時に保障してくれる機能であって、定数化するものではないことに注意してください。
`const`と `constexpr` と `constinit` の違いをそれぞれ説明できないと使いこなすことは難しいです。

## constinit の結論：
実際問題、UEでは使いどころはほとんどないと思います。

グローバルなアクセスが行いたいなら、`WorldSubsystem`等の別の手段を検討したらいいと思います。他の手段を検討して、それでも`extern`や `public static`を使用したいときに`constinit`が使えると安全に初期化できます。

`mutable` なグローバル変数は `constinit` で良いと思います。`public` で `mutable` な `static`変数も定数初期化を実装して `constinit static` で定義するのが良いと思います。
しかしながら、グローバル変数はそうそう使うべきでないし、static変数はやシングルトンは悪なので、そもそもの使用をお勧めしません。

`constinit`に頼るような時点で何か黒魔術的なことをしようとしている気がします。

## 具体例
```cpp
// .cpp
struct FBarWidgetRegistrator
{
    FBarWidgetRegistrator()
    {
        // SlateApplicationの初期化前に呼ばれると危険
        // こういうことはやらない
        FSlateApplication::Get().RegisterWidget(...);
    }
}
static FBarWidgetRegistrator s_BarWidgetInstance; //コンストラクタ呼び出し
```
上記例では、 UnrealEngineの `FSlateApplication`のシングルトンに対して、静的変数の初期化タイミングでアクセスしてしまっています。
static 変数 `s_BarWidgetInstance`の初期化は、実行可能ファイルの場合はメイン関数開始前、モジュールの場合はモジュールがロード時に行われます。
このタイミングでは`FSlateApplication`が初期化されているか、わかりません。初期化前だと`FSlateApplication::Get()`がnullptrを返すか、未定義動作となります。
`Get`内部で`check()`しているのでそこで止まることが期待されるのですが、`static`な `FSlateApplication::CurrentApplication`が確実に他の使用箇所より早く初期化されるかというのはちょっとわかりませんでした。こういうことが安全にできるように`constinit`を使おうとするとエンジンのあちらこちらを改造しないといけないので現実的ではありません。やっぱり使えませんでした。

なお、静的メモリ領域は0クリアされてから構築される、ということが保障されているので、上記コード例に関してはギリギリセーフだと思います。



# FStringView
C++に存在する`std::string_view`のUnrealC++版が `FStringView`です。
c++17から使えていた機能なので本稿の趣旨と外れますが、`constexpr` コンストラクタが使えるので言及しておきます。

文字列を引数に受けるときは可能ならば`FStrinView`を使うようにすると統一的なAPIを実装できます。

使い方はほぼほぼ `std::string_view`と同じです。
`implicit operator` が実装されているので暗黙的に各種文字列型から変換されます。
関数呼び出しを行う側はとくに`FStringView`であることを意識せず文字列を渡せるでしょう。

`FStringView`は軽量な構造体なので値渡しで十分です。下手に参照渡しにするよりも値渡しの方がいいです。
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
    FStringView DirName = GetDirectoryName(Path); // "Game/Map/"
    FStringView ParentDirName = GetDirectoryName(DirName); //"Game/"
}
```
上記コードでは一切の文字列コピーを挟まずに部分文字列を指す区間を得ることができました。
残念ながら `FStringView::FindLastChar`関数やその他メンバ関数が`constexpr`関数でないので、`FStringView`を使用する関数を`constexpr`関数にすることはできませんでした。

`UE_LOG`や`Printf`に渡したいときは直接直接渡せないのでいったん`FString`に戻します。

UE5.5時点では少々取り回しに難点がありますが、中間操作をコピーレスで高速にできるというのは文字列処理周りで恩恵があると思います。とくに文字列パーサーのように元の入力文字列を一切書き換えないreadonly な関数の場合、`FStringView`で受けると引数の型を柔軟に受けつつ、高速かつ効率的な処理を実装できると思います。

# FString::Format
`std::format` がC++20でお目見えしました。それのUnrealC++版です。
やっとC#やpythonのように`{}`によるフォーマット指定子が使えるようになりました。
これで文字列を`%d`で読み込んだり、`%s` に`int`を与えてクラッシュするような事件が減ります。

ただし、UnrealC++なので使い方が少々特殊です。

```cpp
int hour = 12;
int minute = 34;
int second = 56;
// "Timestamp 12:34:56" となる
FString String = FString::Format(TEXT("Timestamp {0}:{1}:{2}"), {hour, minute, second});

// コンパイルエラー.第2引数は {}で括るべし
FString String = FString::Format(TEXT("Timestamp {0}:{1}:{2}"), hour, minute, second);
```

第1引数にはフォーマット文字列を与えます。第2引数は `{}`で括ってあげる必要があります。
`FString::Format`はパラメータパックを受け取らず、パラメーター構造体を挟むことで対応しています。
この`{}`の正体は第2引数の`FStringFormatOrderedArguments`型のコンストラクタです。`{}`初期化による暗黙的なコンストラクタ呼び出しで対応しています。ユーザーからは`initializer_list`を与えたように見える、書き味となっています。

`FStringFormatOrderedArguments`の正体は `TArray<FStringFormatArg>`型ですから、`intializer_list`そのままと言ってもいいでしょう。

残念ながら`FStringFormatArg`の制限により受付可能な型が非常に少ないです。プリミティブ型と`FString`, `FStringView`など文字列型だけです。`FVector`などの著名な型は直接入れられないので、`ToString`で変換してあげる必要があります。

```cpp
FVector Vec(1,2,3);
FString String = FString::Format(TEXT("Vec {0}"), {Vec.ToString()});

//コンパイルラー. 不便......
FString String = FString::Format(TEXT("Vec {0}"), {Vec});
```

それでも、プリミティブ型のフォーマット指定子を間違える事件がなくなるので効能は高めです。実例はエラーコードのパースミスです。

## フォーマット指定子の誤りは未定義動作

```cpp
if(UNLIKELY(hasError))
{
    // フォーマット指定子を間違えたため、エラー発生時のみクラッシュする
    // %s に intを与えたらだめ
    UE_LOG(LogTemp, Error, TEXT("Error :%s"), EErrorCode::HogeError);
    return;
}
```
滅多に発生しないエラーはテストに漏れガチです。エラーが発生することでクラッシュするため、そのエラーが原因だと誤判断されがちですが、実際はエラーハンドリングのエラーです。
直接のエラー原因を隠蔽してしまうため、やっかいです。

このように、`FString::Format`では型安全性が向上し、誤ったフォーマット指定によるランタイムクラッシュをコンパイルエラーにすることができます。`{0}`によるフォーマット指定子を利用する方が安全です。


## FString::Formatの名前指定フォーマット
名前指定フォーマットも可能です。

```cpp
FStringFormatNamedArgument Args; //ただの TMap<FString, FStringFormatArg>
Args.Add(TEXT("Hour"), 12);
Args.Add(TEXT("Minute"), 34);
Args.Add(TEXT("Second"), 56);
FString String = FString::Format(TEXT("Timestamp {Hour}:{Minute}:{Second}"), Args);// "Timestamp 12:34:56" 
```

`FStringFormatNamedArgument`型を使います。こいつの正体は`TMap<FString, FStringFormatArg>`型です。

ただのマップなのでユーザー側で任意の名前指定フォーマットを作成できます。工夫すれば、プロジェクト固有の名前指定フォーマットが作れるでしょう。


# consteval 
進捗だめです。

UE5.5.3 時点では `consteval`は実質使えません。
`UE_CONSTEVAL`が存在しますが、一部のコンパイラのみ対応しています。
ゲームエンジンとしては全てのコンパイラに対応していないとマルチプラットフォームが保障できなくなるので、封印されているようです。

https://cpprefjp.github.io/lang/cpp20/immediate_functions.html

# ビットフィールドのメンバ変数初期化
`Non static data member initializer`:NSDMI がビットフィールドでも許可されました。
UE5でも使えます。どんどん使いましょう。

ゲーム開発ではデータサイズ削減のためにビットフィールド変数を多用することが多いでしょう。定義時にそのままインラインでかけるようになりました。いちいちコンストラクタで初期化する必要がなくなるので助かります。複数コンストラクタがある場合は初期化漏れをおこしがちなので、NSDMIを使った方がいいです。

今までコメントに`uint8 bShouldCallTick:1; // detault true`とコメントに記載する他ありませんでしたが、これからは自明に書けます。

```cpp
USTRUCT()
struct FMyBuffState
{
    GENERATED_BODY()

    UPROPERTY() uint8 bHasPoison : 1 = false;
    UPROPERTY() uint8 bHasPowerUp : 1 = false;
    UPROPERTY() uint8 bHasPowerDown: 1 = false;
    UPROPERTY() uint8 bMovable: 1 = true; // 移動可能フラグ. デフォルトtrue. 移動不可デバフを受けたらfalseとか.
}

//カラーフォーマットRGB565
struct FRGB565
{
    uint16 R : 5 = 0;
    uint16 G : 6 = 0;
    uint16 B : 5 = 0;
}
```

https://cpprefjp.github.io/lang/cpp20/default_member_initializers_for_bit_fields.html


# 指示付き初期化
便利な`{}`による集成体初期化が、進化して指示がつけられるようになりました。
UE5.3以降なら使えます。どんどん使いましょう。`USTRUCT` でも動作します。

```cpp
USTRUCT()
struct FHogeRequestParam
{
    GENERATED_BODY()

    UPROPERTY() FVector Location{ FVector::ZeroVector };
    UPROPERTY() FRotator Rotation{EForceInit};
    UPROPERTY() int Value{};
}

static void DoRequest(const FHogeRequestParam& Req){...}

void Main()
{
    FHogeRequestParam Param
    {
        .Location = FVector(1,2,3),
        .Rotator = FRotator(0,0,0),
        .Value = 100,
    };

    DoRequest(Param);

    TArray<FHogeRequestParam> Params;
    Param.Emplace(
        FHogeRequestParam {
        .Location = FVector(1,2,3),
        .Rotator = FRotator(0,0,0),
        .Value = 100,
       });
}
```

 `UObject`派生型は `NewObject<T>()`で生成するため、使えません。

指示付き初期化のおかげで、引数つきコンストラクタをわざわざ定義することなく、読みやすい初期化が行えます。逆にコンストラクタをユーザー宣言(`user-declared`)すると、指示付き初期化を行えません。

指示を省略したフィールドについては集成体初期化のデフォルト初期化が採用されます。
プリミティブ型の場合0, false, nullptrです。構造体の場合は、デフォルトコンストラクタです。

UE5の`FVector::FVector()`は何もしないコンストラクタなので、`FVector::X,Y,Z`は未初期化となります。そのため、FVectorやFRotatorへの指示の省略には気を付けてください。`Non static data member initialize`を使って、メンバ変数宣言時に初期化しておきましょう。デフォルトコンストラクタがNSDMIを使用するようになります。

## 初期化 vs 代入

初期化なので、コンストラクトした後に値を代入するのとは訳が違います。
```cpp
    // 初期化
    const FHogeRequestParam Param0
    {
        .Location = FVector(1,2,3),
        .Rotator = FRotator(0,0,0),
        .Value = 100,
    };
    // ここでParam0は必ず初期化済み

    // 代入
    FHogeRequestParam Param1;
    // Param1はこの時点ではデフォルトコンストラクタ初期化されている
    // 以下は operator=() による一時オブジェクトの作成と代入
    // 代入なので順番は自由。メンバ宣言順じゃなくていい。
    Param1.Value = 100;
    Param1.Rotator = FRotator(0,0,0);
    Param1.Location = FVector(1,2,3);
```

初期化の場合は、初期化フェーズで値が初期化されるので`const`変数で受けることができ頑健です。
代入の場合は、初期化したあとに値の再代入が行われますから、`immutable`にできません。通常、デフォルトコンストラクタが実行されてから、代入されるわけですから処理的には無駄です。サンプル程度のコードでは最適化を有効にしたら概ね同じになると推測します。（確認してません、へへ）

やはりconst変数にできるという方が可読性や意味論的にも利点が多いでしょう。

