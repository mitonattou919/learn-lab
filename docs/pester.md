# Pester 学習教材

> PowerShellは書けるが、テストを書いたことがない方向けの段階的学習教材

---

## Chapter 0: 概念・Why Pester？

### 1. テストとは何か

#### 1-1. 手動テストの限界

スクリプトを書いたとき、あなたはどうやって「正しく動く」と確認しますか？

多くの場合、こんな流れになります：

1. スクリプトを実行する
2. 出力を目で確認する
3. 問題なければ完了

これは「手動テスト」です。最初は十分に見えますが、すぐに問題が出てきます。

**よくある問題のシナリオ：**

```powershell
# バックアップスクリプト（backup.ps1）
function Backup-Files {
    param($Source, $Destination)
    Copy-Item -Path $Source -Destination $Destination -Recurse
    Write-Host "バックアップ完了: $Source -> $Destination"
}
```

このスクリプトに後から機能を追加したとします。**既存の動作が壊れていないか**、毎回手動で全パターンを確認しますか？

- コピー元が存在しない場合は？
- コピー先のディスクが満杯の場合は？
- ファイル名に特殊文字が含まれる場合は？

パターンが増えるほど手動確認は現実的でなくなります。

#### 1-2. 自動テストの考え方

自動テストとは、「期待する動作」をコードとして記述しておき、機械に検証させることです。

```
期待: Backup-Files を呼んだとき、ファイルがコピーされること
期待: コピー元が存在しなければ、エラーを投げること
```

このような「期待」をコードで書いておけば、スクリプトを変更するたびに自動で検証できます。

#### 1-3. テストがもたらすもの

| 手動テスト | 自動テスト |
|---|---|
| 毎回人間が確認 | 一度書けば何度でも再実行 |
| 見落としが起きる | 記述した条件は必ず検証 |
| 時間がかかる | 秒単位で完了 |
| 変更のたびに怖い | 安心してリファクタリングできる |

---

### 2. Pesterとは何か

#### 2-1. 位置づけ

**Pester**はPowerShell用のテストフレームワークです。

- PowerShell向けに設計されており、PowerShellスクリプトのテストに最適化されている
- Windows PowerShell 5.1 および PowerShell 7.x の両方で動作する
- Windows 10/11にはデフォルトで同梱されている（バージョンは古い場合がある）
- 現在の安定版は **Pester 5.x系**（このコースでは5.x系を対象とする）

#### 2-2. Pesterのインストール・バージョン確認

```powershell
# インストール済みバージョンを確認
Get-Module -Name Pester -ListAvailable

# 最新版をインストール（または更新）
Install-Module -Name Pester -Force -SkipPublisherCheck

# バージョン確認
Import-Module Pester -PassThru | Select-Object -ExpandProperty Version
```

> **注意:** Windows同梱のPesterは3.x系であることが多いです。このコースでは5.x系が必要なので、上記コマンドでインストールしてください。

#### 2-3. Pesterの基本的な見た目

Pesterで書いたテストはこのような形をしています：

```powershell
Describe "Get-Greeting 関数のテスト" {
    It "名前を渡すと挨拶文を返す" {
        $result = Get-Greeting -Name "太郎"
        $result | Should -Be "こんにちは、太郎さん"
    }

    It "名前が空のときエラーを投げる" {
        { Get-Greeting -Name "" } | Should -Throw
    }
}
```

日本語で読むとこうなります：

```
【説明】Get-Greeting 関数のテスト
  ・名前を渡すと挨拶文を返す  → 検証: 結果が "こんにちは、太郎さん" であること
  ・名前が空のときエラーを投げる  → 検証: 例外が発生すること
```

英語の構文がそのまま「仕様書」として読めるのがPesterの特徴です。

---

### 3. Pesterを使うべき理由（実務での価値）

#### 3-1. 変更に強くなる

スクリプトに手を加えるとき、テストがあれば「壊していない」という確信を持てます。これを**リグレッション（退行）防止**と呼びます。

#### 3-2. 仕様をコードで表現できる

「このスクリプトは何をするのか」をテストが示します。コメントと違い、テストは実際に実行されるので古くなりません。

#### 3-3. 設計が自然と良くなる

テストを書きにくいコードは、たいていの場合「設計が複雑すぎる」コードです。Pesterでテストを書こうとすると、自然と関数を小さく分けるようになります。

#### 3-4. チームでの共有が容易

CI/CDパイプライン（GitHub Actions等）にPesterを組み込むことで、プルリクエストのたびに自動でテストを実行できます。

---

### 4. テストファイルの命名規則

Pesterにはファイル名の慣習があります：

```
スクリプト本体:  Get-Greeting.ps1
テストファイル:  Get-Greeting.Tests.ps1
```

`.Tests.ps1` というサフィックスがPesterのテストファイルの目印です。Pesterはこのパターンのファイルを自動的に検出します。

```
プロジェクト構成の例:
MyProject/
├── src/
│   ├── Get-Greeting.ps1
│   └── Backup-Files.ps1
└── tests/
    ├── Get-Greeting.Tests.ps1
    └── Backup-Files.Tests.ps1
```

---

### 5. テストを実行する（先取り体験）

#### 5-0. ディレクトリ構造を用意する

```
PesterDemo/
├── Get-Greeting.ps1
└── Get-Greeting.Tests.ps1
```

```powershell
New-Item -ItemType Directory -Name PesterDemo
Set-Location PesterDemo
```

#### 5-1. テスト対象のスクリプトを作成

```powershell
# Get-Greeting.ps1
function Get-Greeting {
    param(
        [string]$Name
    )
    return "こんにちは、${Name}さん"
}
```

#### 5-2. テストファイルを作成

```powershell
# Get-Greeting.Tests.ps1
BeforeAll {
    . $PSScriptRoot/Get-Greeting.ps1
}

Describe "Get-Greeting" {
    It "名前を渡すと挨拶文を返す" {
        $result = Get-Greeting -Name "太郎"
        $result | Should -Be "こんにちは、太郎さん"
    }
}
```

#### 5-3. テストを実行

```powershell
Invoke-Pester -Path "./Get-Greeting.Tests.ps1" -Output Detailed
```

**出力例：**

```
Starting discovery in 1 files.
Discovery found 1 tests in 22ms.
Running tests.

Describing Get-Greeting
  [+] 名前を渡すと挨拶文を返す 15ms (11ms|4ms)

Tests completed in 53ms
Tests Passed: 1, Failed: 0, Skipped: 0
```

---

### 6. まとめ

| キーワード | 意味 |
|---|---|
| テスト | 期待する動作をコードで記述し、自動的に検証する仕組み |
| Pester | PowerShell向けのテストフレームワーク |
| `.Tests.ps1` | Pesterテストファイルの命名規則 |
| `Invoke-Pester` | テストを実行するコマンド |
| `Describe` | テストのグループ（後続フェーズで詳解） |
| `It` | 個別のテストケース（後続フェーズで詳解） |
| `Should` | 検証を行うコマンド（後続フェーズで詳解） |

---

### 練習問題

#### 問題 1
次のうち、Pesterのテストファイルとして正しい命名はどれですか？

```
a) MyScript.test.ps1
b) MyScript.Tests.ps1
c) Test-MyScript.ps1
d) MyScript_test.ps1
```

#### 問題 2
以下の関数に対するテストファイルを作成してください。

```powershell
# Add-Numbers.ps1
function Add-Numbers {
    param([int]$A, [int]$B)
    return $A + $B
}
```

テストファイル `Add-Numbers.Tests.ps1` を作成し、以下を検証してください：
- `Add-Numbers -A 3 -B 4` の結果が `7` であること
- `Add-Numbers -A 0 -B 0` の結果が `0` であること

#### 問題 3（考察）
あなたが今まで書いたスクリプト（または業務で使っているスクリプト）を思い浮かべてください。そのスクリプトに自動テストがあれば、どんな場面で役立ちますか？

---

### 練習問題の解答

#### 問題 1 の解答
**b) MyScript.Tests.ps1** が正解です。

#### 問題 2 の解答

```powershell
# Add-Numbers.Tests.ps1
BeforeAll {
    . $PSScriptRoot/Add-Numbers.ps1
}

Describe "Add-Numbers" {
    It "3 + 4 は 7 を返す" {
        $result = Add-Numbers -A 3 -B 4
        $result | Should -Be 7
    }

    It "0 + 0 は 0 を返す" {
        $result = Add-Numbers -A 0 -B 0
        $result | Should -Be 0
    }
}
```

---

## Chapter 1: 基礎構文

### 1. テストファイルの骨格

Pesterのテストファイルは以下の3つのブロックで構成されます。

```powershell
BeforeAll {
    . $PSScriptRoot/MyScript.ps1
}

Describe "テストグループの説明" {
    It "個別テストの説明" {
        $result = 何かの処理
        $result | Should -Be 期待値
    }
}
```

---

### 2. BeforeAll：テストの準備

#### 2-1. ドットソーシング

```powershell
BeforeAll {
    . $PSScriptRoot/Get-Greeting.ps1
}
```

| 書き方 | 意味 |
|---|---|
| `. ファイルパス` | ファイルを現在のスコープに読み込む |
| `$PSScriptRoot` | テストファイル自身があるディレクトリの絶対パス |

#### 2-2. BeforeAll の注意点（Pester 5.x）

```powershell
# 悪い例（Pester 5.x では動かない）
. $PSScriptRoot/Get-Greeting.ps1

Describe "Get-Greeting" {
    It "動くかな？" {
        Get-Greeting -Name "太郎"   # エラーになる
    }
}
```

```powershell
# 良い例
BeforeAll {
    . $PSScriptRoot/Get-Greeting.ps1
}

Describe "Get-Greeting" {
    It "動く" {
        Get-Greeting -Name "太郎"
    }
}
```

---

### 3. Describe：テストのグループ化

```powershell
Describe "グループの説明文" {
    # ここに It ブロックを書く
}
```

#### 3-1. ネスト（入れ子）

```powershell
Describe "Get-Greeting" {

    Describe "正常系" {
        It "名前を渡すと挨拶文を返す" { ... }
        It "長い名前でも動く" { ... }
    }

    Describe "異常系" {
        It "空文字を渡すとエラーになる" { ... }
    }
}
```

#### 3-2. Context

`Context` は `Describe` と機能的に同じですが、**条件・状況を表す**のに使う慣習があります。

```powershell
Describe "Get-Greeting" {
    Context "名前が指定されているとき" {
        It "挨拶文を返す" { ... }
    }

    Context "名前が空のとき" {
        It "エラーを投げる" { ... }
    }
}
```

---

### 4. It：個別テストケース

```powershell
It "テストの説明文" {
    # 検証コード
}
```

| 状態 | 記号 | 意味 |
|---|---|---|
| 成功 | `[+]` 緑 | 検証が全て通った |
| 失敗 | `[-]` 赤 | 検証が1つでも失敗した |
| スキップ | `[!]` 黄 | `-Skip` で意図的にスキップ |

---

### 5. Should：値の検証

```powershell
実際の値 | Should -マッチャー 期待値
```

#### 5-1. -Be：完全一致

```powershell
$result = 1 + 1
$result | Should -Be 2
```

#### 5-2. -BeNullOrEmpty

```powershell
$value = ""
$value | Should -BeNullOrEmpty
```

#### 5-3. -Throw：例外の検証

```powershell
{ throw "エラー" } | Should -Throw
{ throw "ファイルが見つかりません" } | Should -Throw "*ファイルが見つかりません*"
```

---

### 6. 動作するサンプル

#### テスト対象

```powershell
# Calculator.ps1
function Add-Numbers {
    param([int]$A, [int]$B)
    return $A + $B
}

function Divide-Numbers {
    param([int]$Dividend, [int]$Divisor)
    if ($Divisor -eq 0) {
        throw "ゼロ除算はできません"
    }
    return $Dividend / $Divisor
}
```

#### テストファイル

```powershell
# Calculator.Tests.ps1
BeforeAll {
    . $PSScriptRoot/Calculator.ps1
}

Describe "Add-Numbers" {
    Context "正常な入力のとき" {
        It "正の数を足すと正しい合計を返す" {
            $result = Add-Numbers -A 3 -B 4
            $result | Should -Be 7
        }

        It "負の数を含む場合も正しく計算する" {
            $result = Add-Numbers -A -5 -B 3
            $result | Should -Be -2
        }

        It "両方ゼロのとき 0 を返す" {
            $result = Add-Numbers -A 0 -B 0
            $result | Should -Be 0
        }
    }
}

Describe "Divide-Numbers" {
    Context "正常な入力のとき" {
        It "10 ÷ 2 は 5 を返す" {
            $result = Divide-Numbers -Dividend 10 -Divisor 2
            $result | Should -Be 5
        }
    }

    Context "ゼロ除算のとき" {
        It "例外を投げる" {
            { Divide-Numbers -Dividend 10 -Divisor 0 } | Should -Throw "*ゼロ除算はできません*"
        }
    }
}
```

---

### 7. テストが失敗したときの読み方

```
[-] わざと失敗させる 18ms (15ms|3ms)
 Expected: 99
 But was:  2
 at $result | Should -Be 99, ...
```

| 項目 | 意味 |
|---|---|
| `Expected` | テストで期待した値 |
| `But was` | 実際に返ってきた値 |
| `at ...` | 失敗した行 |

---

### 8. まとめ

```
BeforeAll { }    → テスト前の準備
Describe { }     → テストのグループ化
Context { }      → 状況による分類
It { }           → 個別テストケース
Should -Be       → 完全一致の検証
Should -BeNullOrEmpty → null/空の検証
Should -Throw    → 例外発生の検証
```

---

### 練習問題

#### 問題 1
次のテストコードには問題があります。何が問題で、どう直せばよいですか？

```powershell
. $PSScriptRoot/MyScript.ps1

Describe "MyFunction" {
    It "正しい値を返す" {
        $result = MyFunction -Input "test"
        $result | Should -Be "TEST"
    }
}
```

#### 問題 2
以下の関数に対するテストを書いてください。

```powershell
# StringUtils.ps1
function ConvertTo-UpperCase {
    param([string]$Text)
    if ([string]::IsNullOrEmpty($Text)) {
        throw "テキストが空です"
    }
    return $Text.ToUpper()
}
```

---

### 練習問題の解答

#### 問題 1 の解答
`Describe` の外でドットソーシングしている（Pester 5.x では動作しない）。`BeforeAll` の中に移動する。

#### 問題 2 の解答

```powershell
BeforeAll {
    . $PSScriptRoot/StringUtils.ps1
}

Describe "ConvertTo-UpperCase" {
    Context "有効なテキストのとき" {
        It "'hello' を渡すと 'HELLO' を返す" {
            $result = ConvertTo-UpperCase -Text "hello"
            $result | Should -Be "HELLO"
        }

        It "'PowerShell' を渡すと 'POWERSHELL' を返す" {
            $result = ConvertTo-UpperCase -Text "PowerShell"
            $result | Should -Be "POWERSHELL"
        }
    }

    Context "無効なテキストのとき" {
        It "空文字を渡すと例外を投げる" {
            { ConvertTo-UpperCase -Text "" } | Should -Throw "*テキストが空です*"
        }

        It "null を渡すと例外を投げる" {
            { ConvertTo-UpperCase -Text $null } | Should -Throw "*テキストが空です*"
        }
    }
}
```

---

## Chapter 2: アサーション（Should マッチャー）

### 1. アサーションとは

**アサーション（assertion）**とは「この値はこうであるべき」という主張をコードで表現したものです。

```powershell
実際の値 | Should -マッチャー名 [期待値]
```

---

### 2. 否定：-Not

すべてのマッチャーは `-Not` を前に置くことで否定になります。

```powershell
$result | Should -Not -Be 0
$result | Should -Not -BeNullOrEmpty
{ ... }  | Should -Not -Throw
```

---

### 3. 等値・比較系マッチャー

#### 3-1. -Be：完全一致

```powershell
42 | Should -Be 42
"Hello" | Should -Be "Hello"
"Hello" | Should -Not -Be "hello"
```

#### 3-2. -BeLike：ワイルドカード一致

```powershell
"Hello, World" | Should -BeLike "Hello*"
"Hello, World" | Should -BeLike "*World"
```

> `-BeLike` は大文字小文字を**区別しません**。区別したい場合は `-BeLikeExactly` を使います。

#### 3-3. -Match：正規表現マッチ

```powershell
"error: file not found" | Should -Match "error:"
"user@example.com"      | Should -Match "^\w+@\w+\.\w+$"
```

> `-Match` は大文字小文字を**区別しません**。区別したい場合は `-MatchExactly` を使います。

#### 3-4. 数値比較

```powershell
10 | Should -BeGreaterThan 5
10 | Should -BeGreaterOrEqual 10
5  | Should -BeLessThan 10
5  | Should -BeLessOrEqual 5
```

---

### 4. 型・存在確認系マッチャー

#### 4-1. -BeOfType

```powershell
42          | Should -BeOfType [int]
"hello"     | Should -BeOfType [string]
Get-Date    | Should -BeOfType [DateTime]
,@(1, 2, 3) | Should -BeOfType [array]
```

> **配列を検証するときは先頭に `,` を付ける**（パイプラインの自動展開を防ぐ）

#### 4-2. -BeNullOrEmpty

```powershell
$null | Should -BeNullOrEmpty
""    | Should -BeNullOrEmpty
@()   | Should -BeNullOrEmpty
```

#### 4-3. -BeTrue / -BeFalse

```powershell
$true  | Should -BeTrue
$false | Should -BeFalse
```

---

### 5. コレクション系マッチャー

#### 5-1. -Contain

```powershell
$fruits = @("apple", "banana", "cherry")
$fruits | Should -Contain "banana"
$fruits | Should -Not -Contain "grape"
```

#### 5-2. -HaveCount

```powershell
$items = @("a", "b", "c")
$items | Should -HaveCount 3
```

#### 5-3. -BeIn

```powershell
$allowedRoles = @("admin", "user", "guest")
"admin" | Should -BeIn $allowedRoles
```

---

### 6. ファイル・パス系マッチャー

#### 6-1. -Exist

```powershell
"C:\Windows" | Should -Exist
"C:\存在しないフォルダ" | Should -Not -Exist
```

#### 6-2. -FileContentMatch

```powershell
"C:\Logs\app.log" | Should -FileContentMatch "ERROR"
```

---

### 7. 例外系マッチャー

```powershell
{ throw "何らかのエラー" } | Should -Throw
{ throw "ファイルが見つかりません" } | Should -Throw "*ファイル*"
```

---

### 8. マッチャー選択の指針

```
確認したいこと                      使うマッチャー
─────────────────────────────────────────────────
値が完全に等しい                    -Be
文字列がパターンに一致する           -BeLike（ワイルドカード）
文字列が正規表現に一致する           -Match
値が配列の中にある                  -BeIn
配列が値を含む                      -Contain
配列の要素数                        -HaveCount
値が null または空                  -BeNullOrEmpty
値が true / false                   -BeTrue / -BeFalse
型が正しい                          -BeOfType
数値の大小                          -BeGreaterThan 等
ファイルが存在する                  -Exist
例外が発生する                      -Throw
上記すべての否定                    -Not -マッチャー
```

---

### 9. まとめ

| カテゴリ | マッチャー |
|---|---|
| 等値 | `-Be` `-BeExactly` `-BeLike` `-Match` |
| 数値比較 | `-BeGreaterThan` `-BeLessThan` `-BeGreaterOrEqual` `-BeLessOrEqual` |
| 型・存在 | `-BeOfType` `-BeNullOrEmpty` `-BeTrue` `-BeFalse` |
| コレクション | `-Contain` `-HaveCount` `-BeIn` |
| ファイル | `-Exist` `-FileContentMatch` |
| 例外 | `-Throw` |
| 否定 | 全マッチャーに `-Not` を付けられる |

---

## Chapter 3: セットアップ／ティアダウン

### 1. なぜセットアップ／ティアダウンが必要か

```powershell
# 準備コードが各 It に重複している（悪い例）
Describe "ファイル操作のテスト" {
    It "ファイルを読める" {
        New-Item -Path "C:\Temp\test.txt" -Value "hello"
        # テスト...
        Remove-Item -Path "C:\Temp\test.txt"
    }

    It "ファイルを書き換えられる" {
        New-Item -Path "C:\Temp\test.txt" -Value "hello"   # 重複
        # テスト...
        Remove-Item -Path "C:\Temp\test.txt"               # 重複
    }
}
```

---

### 2. 4つのブロック

| ブロック | 実行タイミング |
|---|---|
| `BeforeAll` | `Describe`内の**全テストの前に1回** |
| `AfterAll` | `Describe`内の**全テストの後に1回** |
| `BeforeEach` | `Describe`内の**各 `It` の前に毎回** |
| `AfterEach` | `Describe`内の**各 `It` の後に毎回** |

---

### 3. BeforeAll / AfterAll

```powershell
Describe "データベース接続テスト" {
    BeforeAll {
        $script:connection = Open-DatabaseConnection -Server "localhost"
    }

    AfterAll {
        Close-DatabaseConnection -Connection $script:connection
    }

    It "クエリを実行できる" {
        $result = Invoke-Query -Connection $script:connection -Query "SELECT 1"
        $result | Should -Not -BeNullOrEmpty
    }
}
```

#### $script: スコープについて

```powershell
Describe "スコープのサンプル" {
    BeforeAll {
        $script:value = "共有する値"   # $script: を付ける
        $localValue   = "ローカルな値" # これは It から見えない
    }

    It "script スコープの変数は見える" {
        $script:value | Should -Be "共有する値"
    }
}
```

> **重要:** `BeforeAll` 内の変数は `$script:変数名` で定義し、`It` 内でも `$script:変数名` でアクセスします。

---

### 4. BeforeEach / AfterEach

```powershell
Describe "ファイル操作テスト" {
    BeforeEach {
        $script:testFile = Join-Path $TestDrive "test.txt"
        Set-Content -Path $script:testFile -Value "初期内容"
    }

    AfterEach {
        if (Test-Path $script:testFile) {
            Remove-Item $script:testFile
        }
    }

    It "ファイルの内容を読める" {
        $content = Get-Content -Path $script:testFile
        $content | Should -Be "初期内容"
    }

    It "書き換えても次のテストには影響しない" {
        Set-Content -Path $script:testFile -Value "新しい内容"
        $content = Get-Content -Path $script:testFile
        $content | Should -Be "新しい内容"
    }
}
```

#### $TestDrive

`$TestDrive` はPesterが自動で用意する一時ディレクトリです。テスト終了後に自動削除されます。

---

### 5. ネストしたブロックの実行順序

```
外側 BeforeAll
  内側 BeforeAll
    外側 BeforeEach → 内側 BeforeEach → テスト A → 内側 AfterEach → 外側 AfterEach
    外側 BeforeEach → 内側 BeforeEach → テスト B → 内側 AfterEach → 外側 AfterEach
  内側 AfterAll
外側 AfterAll
```

---

### 6. 使い分けの判断基準

```
準備コストが高い（DB接続、ファイル生成など）
かつ テスト間で状態を共有してよい
    → BeforeAll / AfterAll

各テストが独立した状態で動く必要がある
    → BeforeEach / AfterEach
```

---

### 7. よくある間違いと対策

#### $script: を忘れる

```powershell
# 悪い例
BeforeAll {
    $config = Load-Config   # $script: がない
}
It "設定が読める" {
    $config | Should -Not -BeNullOrEmpty   # $config は $null → 失敗
}
```

```powershell
# 良い例
BeforeAll {
    $script:config = Load-Config
}
It "設定が読める" {
    $script:config | Should -Not -BeNullOrEmpty
}
```

---

### 8. まとめ

```
BeforeAll  → 1回だけ実行（重い初期化に使う）
AfterAll   → 1回だけ実行（後片付けに使う）
BeforeEach → 各 It の前に毎回実行（状態のリセット）
AfterEach  → 各 It の後に毎回実行（クリーンアップ）

$script:変数名  → BeforeAll/BeforeEach から It に値を渡すスコープ
$TestDrive      → Pester が用意する自動クリーンアップ付き一時ディレクトリ
```

---

## Chapter 4: モック

### 1. なぜモックが必要か

実務のスクリプトには必ず「外部への依存」があります。

```powershell
function Send-AlertMail {
    param([string]$Message)
    $body = "アラート: $Message"
    Send-MailMessage -To "admin@example.com" -Subject "Alert" -Body $body -SmtpServer "smtp.example.com"
}
```

このスクリプトをテストしようとすると：
- メールサーバーが必要
- テストのたびに実際のメールが送信される
- ネットワーク障害でテストが不安定になる

これらの問題を解決するのが**モック（Mock）**です。

---

### 2. Mock の基本

```powershell
Mock コマンド名 { 代わりに実行するコード }
```

```powershell
Describe "Send-AlertMail のテスト" {
    BeforeAll {
        . $PSScriptRoot/AlertMailer.ps1
    }

    It "Send-MailMessage が呼ばれる" {
        Mock Send-MailMessage {}

        Send-AlertMail -Message "ディスク容量不足"

        Should -Invoke Send-MailMessage -Times 1
    }
}
```

---

### 3. Should -Invoke：呼び出しの検証

```powershell
Should -Invoke コマンド名 -Times 1      # 1回だけ呼ばれた
Should -Invoke コマンド名 -Times 2      # 2回呼ばれた
Should -Invoke コマンド名 -Times 0      # まったく呼ばれていない
Should -Invoke コマンド名               # 呼ばれたかどうかだけ確認
```

#### 引数の検証：-ParameterFilter

```powershell
Should -Invoke Send-MailMessage -Times 1 -ParameterFilter {
    $To -eq "admin@example.com"
}
```

---

### 4. 戻り値を持つモック

```powershell
Mock Get-Date { return [DateTime]"2026-01-01" }

Mock Get-Process {
    return [PSCustomObject]@{
        Name = "notepad"
        Id   = 1234
    }
}

# 例外を投げるモック
Mock Invoke-RestMethod { throw "接続タイムアウト" }
```

#### 引数に応じて異なる値を返す

```powershell
Mock Get-ServerConfig {
    return [PSCustomObject]@{ MaxConnections = 100 }
} -ParameterFilter { $ServerName -eq "prod" }

Mock Get-ServerConfig {
    return [PSCustomObject]@{ MaxConnections = 10 }
} -ParameterFilter { $ServerName -eq "dev" }
```

---

### 5. モックの対象になるもの

```powershell
Mock Get-Date       { return [DateTime]"2026-01-01" }
Mock Get-Content    { return "偽のファイル内容" }
Mock Test-Path      { return $true }
Mock New-Item       {}
Mock Remove-Item    {}
```

別モジュールの関数をモックするときは `-ModuleName` を指定：

```powershell
Mock Get-ADUser { return [PSCustomObject]@{ Name = "TestUser" } } `
    -ModuleName ActiveDirectory
```

---

### 6. 動作するサンプル

#### テスト対象

```powershell
# SystemMonitor.ps1
function Get-DiskAlert {
    param(
        [string]$DriveLetter = "C",
        [int]$ThresholdPercent = 80
    )

    $disk = Get-PSDrive -Name $DriveLetter -ErrorAction Stop
    $usedPercent = [math]::Round(($disk.Used / ($disk.Used + $disk.Free)) * 100, 1)

    if ($usedPercent -ge $ThresholdPercent) {
        $message = "ディスク使用率が ${usedPercent}% に達しました"
        Write-EventLog -LogName Application -Source "SystemMonitor" `
            -EventId 1001 -EntryType Warning -Message $message
        return [PSCustomObject]@{
            AlertTriggered = $true
            Message        = $message
        }
    }

    return [PSCustomObject]@{
        AlertTriggered = $false
        Message        = $null
    }
}
```

#### テストファイル

```powershell
# SystemMonitor.Tests.ps1
BeforeAll {
    . $PSScriptRoot/SystemMonitor.ps1
}

Describe "Get-DiskAlert" {

    Context "使用率が閾値未満のとき" {
        BeforeAll {
            Mock Get-PSDrive {
                [PSCustomObject]@{ Name = "C"; Used = 50GB; Free = 50GB }
            }
            Mock Write-EventLog {}
        }

        It "AlertTriggered が false である" {
            $result = Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            $result.AlertTriggered | Should -BeFalse
        }

        It "Write-EventLog は呼ばれない" {
            Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            Should -Invoke Write-EventLog -Times 0
        }
    }

    Context "使用率が閾値以上のとき" {
        BeforeAll {
            Mock Get-PSDrive {
                [PSCustomObject]@{ Name = "C"; Used = 90GB; Free = 10GB }
            }
            Mock Write-EventLog {}
        }

        It "AlertTriggered が true である" {
            $result = Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            $result.AlertTriggered | Should -BeTrue
        }

        It "Write-EventLog が1回呼ばれる" {
            Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            Should -Invoke Write-EventLog -Times 1
        }

        It "Write-EventLog に正しいイベント ID が渡される" {
            Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            Should -Invoke Write-EventLog -Times 1 -ParameterFilter {
                $EventId -eq 1001
            }
        }
    }
}
```

---

### 7. よくある落とし穴

#### Mock は Describe の中に書く

```powershell
# 悪い例
Mock Get-Date { return [DateTime]"2026-01-01" }

Describe "テスト" {
    It "日付が固定される" {
        (Get-Date).Year | Should -Be 2026   # 動かない
    }
}
```

#### Should -Invoke は It の中に書く

```powershell
# 悪い例：BeforeAll に書くと常に 0 回
BeforeAll {
    Mock Send-MailMessage {}
    Should -Invoke Send-MailMessage -Times 1   # まだ呼ばれていない
}

# 良い例
It "メールが送信される" {
    Send-AlertMail -Message "test"
    Should -Invoke Send-MailMessage -Times 1
}
```

---

### 8. まとめ

```
Mock コマンド名 { 代替コード }
    → 指定したコマンドを偽物に差し替える

Mock コマンド名 { 代替コード } -ParameterFilter { 条件 }
    → 特定の引数のときだけ偽物に差し替える

Should -Invoke コマンド名 -Times N
    → N 回呼ばれたことを確認

Should -Invoke コマンド名 -Times N -ParameterFilter { 条件 }
    → 特定の引数で N 回呼ばれたことを確認
```

---

## Chapter 5: 実務パターン

### 1. このフェーズの目的

Phase 0〜4で学んだ知識を統合し、実務でよく遭遇するシナリオに対応できるようにします。

| テーマ | 実務での例 |
|---|---|
| ファイル操作 | ログローテーション、設定ファイルの読み書き |
| レジストリ | インストール確認、設定値の読み書き |
| 外部コマンド | ping、robocopy、外部CLIツールの呼び出し |

---

### 2. パターン 1：ファイル操作

#### テスト対象

```powershell
# LogRotator.ps1
function Invoke-LogRotation {
    param(
        [string]$LogDirectory,
        [int]$RetentionDays = 30,
        [string]$ArchiveDirectory
    )

    if (-not (Test-Path $LogDirectory)) {
        throw "ログディレクトリが見つかりません: $LogDirectory"
    }

    $cutoffDate = (Get-Date).AddDays(-$RetentionDays)
    $oldLogs = Get-ChildItem -Path $LogDirectory -Filter "*.log" |
               Where-Object { $_.LastWriteTime -lt $cutoffDate }

    $result = [PSCustomObject]@{
        ArchivedCount = 0
        DeletedCount  = 0
        Errors        = @()
    }

    foreach ($log in $oldLogs) {
        try {
            if ($ArchiveDirectory) {
                if (-not (Test-Path $ArchiveDirectory)) {
                    New-Item -Path $ArchiveDirectory -ItemType Directory | Out-Null
                }
                Move-Item -Path $log.FullName -Destination $ArchiveDirectory
                $result.ArchivedCount++
            } else {
                Remove-Item -Path $log.FullName
                $result.DeletedCount++
            }
        }
        catch {
            $result.Errors += $log.Name
        }
    }

    return $result
}
```

#### テストコード

```powershell
# LogRotator.Tests.ps1
BeforeAll {
    . $PSScriptRoot/LogRotator.ps1
}

Describe "Invoke-LogRotation" {

    BeforeAll {
        $script:logDir     = Join-Path $TestDrive "logs"
        $script:archiveDir = Join-Path $TestDrive "archive"
        New-Item -Path $script:logDir -ItemType Directory
    }

    BeforeEach {
        Get-ChildItem $script:logDir | Remove-Item
        if (Test-Path $script:archiveDir) {
            Remove-Item $script:archiveDir -Recurse
        }
    }

    Context "ログディレクトリが存在しないとき" {
        It "例外を投げる" {
            { Invoke-LogRotation -LogDirectory (Join-Path $TestDrive "nonexistent") } |
                Should -Throw "*ログディレクトリが見つかりません*"
        }
    }

    Context "古いログが存在するとき（削除モード）" {
        BeforeEach {
            $oldFile = Join-Path $script:logDir "old.log"
            Set-Content -Path $oldFile -Value "古いログ"
            (Get-Item $oldFile).LastWriteTime = (Get-Date).AddDays(-35)

            $newFile = Join-Path $script:logDir "new.log"
            Set-Content -Path $newFile -Value "新しいログ"
        }

        It "古いログファイルが削除される" {
            Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            Join-Path $script:logDir "old.log" | Should -Not -Exist
        }

        It "新しいログファイルは残る" {
            Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            Join-Path $script:logDir "new.log" | Should -Exist
        }

        It "DeletedCount が 1 になる" {
            $result = Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            $result.DeletedCount | Should -Be 1
        }
    }

    Context "古いログが存在するとき（アーカイブモード）" {
        BeforeEach {
            $oldFile = Join-Path $script:logDir "archive_target.log"
            Set-Content -Path $oldFile -Value "アーカイブ対象"
            (Get-Item $oldFile).LastWriteTime = (Get-Date).AddDays(-40)
        }

        It "アーカイブディレクトリが自動作成される" {
            Invoke-LogRotation -LogDirectory $script:logDir `
                               -RetentionDays 30 `
                               -ArchiveDirectory $script:archiveDir
            $script:archiveDir | Should -Exist
        }

        It "古いログがアーカイブに移動する" {
            Invoke-LogRotation -LogDirectory $script:logDir `
                               -RetentionDays 30 `
                               -ArchiveDirectory $script:archiveDir
            Join-Path $script:archiveDir "archive_target.log" | Should -Exist
        }

        It "ArchivedCount が 1 になる" {
            $result = Invoke-LogRotation -LogDirectory $script:logDir `
                                         -RetentionDays 30 `
                                         -ArchiveDirectory $script:archiveDir
            $result.ArchivedCount | Should -Be 1
        }
    }
}
```

---

### 3. パターン 2：レジストリ

#### テスト対象

```powershell
# AppInstallChecker.ps1
function Get-InstalledAppInfo {
    param([string]$AppName)

    $registryPaths = @(
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall",
        "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"
    )

    foreach ($path in $registryPaths) {
        $app = Get-ChildItem -Path $path -ErrorAction SilentlyContinue |
               Get-ItemProperty |
               Where-Object { $_.DisplayName -like "*$AppName*" } |
               Select-Object -First 1

        if ($app) {
            return [PSCustomObject]@{
                DisplayName     = $app.DisplayName
                Version         = $app.DisplayVersion
                Publisher       = $app.Publisher
                Found           = $true
            }
        }
    }

    return [PSCustomObject]@{ Found = $false }
}
```

#### テストコード（モック戦略）

```powershell
# AppInstallChecker.Tests.ps1
BeforeAll {
    . $PSScriptRoot/AppInstallChecker.ps1
}

Describe "Get-InstalledAppInfo" {

    Context "アプリが見つかるとき" {
        BeforeAll {
            Mock Get-ChildItem {
                return [PSCustomObject]@{ PSPath = "HKLM:\...\{FAKE-GUID}" }
            }
            Mock Get-ItemProperty {
                return [PSCustomObject]@{
                    DisplayName    = "MyApp 2.0.1"
                    DisplayVersion = "2.0.1"
                    Publisher      = "My Company"
                }
            }
        }

        It "Found が true である" {
            $result = Get-InstalledAppInfo -AppName "MyApp"
            $result.Found | Should -BeTrue
        }

        It "DisplayName にアプリ名が含まれる" {
            $result = Get-InstalledAppInfo -AppName "MyApp"
            $result.DisplayName | Should -BeLike "*MyApp*"
        }
    }

    Context "アプリが見つからないとき" {
        BeforeAll {
            Mock Get-ChildItem { return @() }
            Mock Get-ItemProperty { return @() }
        }

        It "Found が false である" {
            $result = Get-InstalledAppInfo -AppName "存在しないアプリ"
            $result.Found | Should -BeFalse
        }
    }
}
```

---

### 4. パターン 3：外部コマンド

外部コマンド（ping、robocopyなど）は直接モックできません。**ラッパー関数に分離してモック化**するのが定石です。

```powershell
# NetworkDiagnostics.ps1
function Invoke-PingCommand {
    param([string]$Target, [int]$TimeoutMs)
    & ping -n 1 -w $TimeoutMs $Target 2>&1
    return $LASTEXITCODE
}

function Test-NetworkConnectivity {
    param([string[]]$Targets, [int]$TimeoutMs = 1000)

    $results = @()
    foreach ($target in $Targets) {
        Invoke-PingCommand -Target $target -TimeoutMs $TimeoutMs
        $success = ($LASTEXITCODE -eq 0)

        $results += [PSCustomObject]@{
            Target  = $target
            Success = $success
        }
    }
    return $results
}
```

```powershell
# NetworkDiagnostics.Tests.ps1
BeforeAll {
    . $PSScriptRoot/NetworkDiagnostics.ps1
}

Describe "Test-NetworkConnectivity" {

    Context "疎通できるとき" {
        BeforeAll {
            Mock Invoke-PingCommand {
                $script:LASTEXITCODE = 0
                return @("Reply from 192.168.1.1")
            }
        }

        It "Success が true" {
            $results = Test-NetworkConnectivity -Targets @("192.168.1.1")
            $results[0].Success | Should -BeTrue
        }
    }

    Context "疎通できないとき" {
        BeforeAll {
            Mock Invoke-PingCommand {
                $script:LASTEXITCODE = 1
                return @("Request timed out.")
            }
        }

        It "Success が false になる" {
            $results = Test-NetworkConnectivity -Targets @("10.0.0.99")
            $results[0].Success | Should -BeFalse
        }
    }
}
```

---

### 5. 実務で使えるテクニック集

#### 5-1. TestCases：パラメータ化テスト

```powershell
Describe "ConvertTo-FileSizeString" {
    BeforeAll {
        function ConvertTo-FileSizeString {
            param([long]$Bytes)
            if ($Bytes -ge 1GB) { return "{0:N1} GB" -f ($Bytes / 1GB) }
            if ($Bytes -ge 1MB) { return "{0:N1} MB" -f ($Bytes / 1MB) }
            if ($Bytes -ge 1KB) { return "{0:N1} KB" -f ($Bytes / 1KB) }
            return "$Bytes B"
        }
    }

    $testCases = @(
        @{ Bytes = 500;        Expected = "500 B"  }
        @{ Bytes = 2048;       Expected = "2.0 KB" }
        @{ Bytes = 3145728;    Expected = "3.0 MB" }
        @{ Bytes = 1073741824; Expected = "1.0 GB" }
    )

    It "<Bytes> バイトは <Expected> になる" -TestCases $testCases {
        $result = ConvertTo-FileSizeString -Bytes $Bytes
        $result | Should -Be $Expected
    }
}
```

#### 5-2. -Tag：テストの選択実行

```powershell
Describe "ユーザー管理" {
    It "ユーザーを作成できる" -Tag "unit" { ... }
    It "Active Directory に同期される" -Tag "integration", "ad" { ... }
}

# タグを指定して実行
Invoke-Pester -Path "." -Tag "unit"
Invoke-Pester -Path "." -ExcludeTag "ad"
```

#### 5-3. PesterConfiguration

```powershell
$config = New-PesterConfiguration
$config.Run.Path              = "./tests"
$config.Output.Verbosity      = "Detailed"
$config.TestResult.Enabled    = $true
$config.TestResult.OutputPath = "./TestResults.xml"
$config.Filter.Tag            = @("unit")

Invoke-Pester -Configuration $config
```

---

### 6. テストの品質チェックリスト

```
[ ] 正常系（ハッピーパス）のテストが書かれているか
[ ] 異常系（エラー、例外）のテストが書かれているか
[ ] 境界値（0件、1件、最大値、最小値）のテストが書かれているか
[ ] 外部依存（ファイル・レジストリ・外部コマンド）がモック化されているか
[ ] $TestDrive を使って実際のファイルシステムを汚染していないか
[ ] BeforeAll/BeforeEach の使い分けが適切か
[ ] It の説明文を読むだけで何を確認しているか分かるか
[ ] テストが互いに独立していて、実行順序に依存していないか
[ ] AfterAll/AfterEach で後片付けが行われているか
```

---

### 7. まとめ：フェーズ全体の振り返り

```
Chapter 0: テストとは何か、Pesterの位置づけ
Chapter 1: Describe / It / Should の骨格
Chapter 2: Should マッチャーの全体像と使い分け
Chapter 3: BeforeAll/AfterAll/BeforeEach/AfterEach によるセットアップ・ティアダウン
Chapter 4: Mock による外部依存の切り離し、Should -Invoke による呼び出し検証
Chapter 5: ファイル操作・レジストリ・外部コマンドの実務パターン
           TestCases によるパラメータ化、Tag による選択実行、PesterConfiguration
```

**次のアクション：**
- 手元にある実務スクリプトに対してテストを書いてみる
- CI/CDパイプラインにPesterを組み込む（GitHub Actions等）
- `Invoke-Pester -Configuration $config` でXMLレポートを出力し、テスト結果を可視化する

---

### 練習問題

#### 問題 1：TestCases を使ったパラメータ化

```powershell
function ConvertTo-NetworkClass {
    param([string]$IpAddress)
    $firstOctet = [int]($IpAddress -split "\.")[0]
    if ($firstOctet -le 127) { return "A" }
    if ($firstOctet -le 191) { return "B" }
    if ($firstOctet -le 223) { return "C" }
    return "D以上"
}
```

テストケース：`10.0.0.1`→`"A"` / `128.0.0.1`→`"B"` / `192.168.1.1`→`"C"` / `224.0.0.1`→`"D以上"`

#### 問題 2：ファイル操作のテスト

```powershell
# ConfigManager.ps1
function Export-Config {
    param([hashtable]$Config, [string]$Path)
    $Config | ConvertTo-Json -Depth 5 | Set-Content -Path $Path -Encoding UTF8
}

function Import-Config {
    param([string]$Path)
    if (-not (Test-Path $Path)) {
        throw "設定ファイルが見つかりません: $Path"
    }
    return Get-Content -Path $Path -Raw | ConvertFrom-Json
}
```

#### 問題 3：モックと実ファイルの組み合わせ

```powershell
# ReportGenerator.ps1
function New-DailyReport {
    param([string]$OutputDirectory)
    $today      = Get-Date
    $fileName   = "report_{0}.txt" -f $today.ToString("yyyyMMdd")
    $outputPath = Join-Path $OutputDirectory $fileName
    $content = "日次レポート`n生成日時: $today"
    Set-Content -Path $outputPath -Value $content -Encoding UTF8
    return $outputPath
}
```

---

### 練習問題の解答

#### 問題 1 の解答

```powershell
BeforeAll {
    . $PSScriptRoot/NetworkUtils.ps1
}

Describe "ConvertTo-NetworkClass" {
    $testCases = @(
        @{ IP = "10.0.0.1";      Expected = "A"    }
        @{ IP = "127.0.0.1";     Expected = "A"    }
        @{ IP = "128.0.0.1";     Expected = "B"    }
        @{ IP = "191.255.255.0"; Expected = "B"    }
        @{ IP = "192.168.1.1";   Expected = "C"    }
        @{ IP = "223.255.255.0"; Expected = "C"    }
        @{ IP = "224.0.0.1";     Expected = "D以上" }
    )

    It "<IP> はクラス <Expected> に分類される" -TestCases $testCases {
        $result = ConvertTo-NetworkClass -IpAddress $IP
        $result | Should -Be $Expected
    }
}
```

#### 問題 2 の解答

```powershell
BeforeAll {
    . $PSScriptRoot/ConfigManager.ps1
}

Describe "Export-Config と Import-Config" {
    BeforeEach {
        $script:configPath = Join-Path $TestDrive "config.json"
        $script:testConfig = @{ AppName = "MyApp"; Version = "2.0" }
    }

    It "Export-Config を実行するとファイルが作成される" {
        Export-Config -Config $script:testConfig -Path $script:configPath
        $script:configPath | Should -Exist
    }

    It "Export 後に Import すると元のデータが復元される" {
        Export-Config -Config $script:testConfig -Path $script:configPath
        $loaded = Import-Config -Path $script:configPath
        $loaded.AppName | Should -Be "MyApp"
    }

    It "存在しないパスを Import-Config に渡すと例外が投げられる" {
        { Import-Config -Path (Join-Path $TestDrive "ghost.json") } |
            Should -Throw "*設定ファイルが見つかりません*"
    }
}
```

#### 問題 3 の解答

```powershell
BeforeAll {
    . $PSScriptRoot/ReportGenerator.ps1
    Mock Get-Date { return [DateTime]"2026-03-06 10:00:00" }
}

Describe "New-DailyReport" {
    BeforeEach {
        $script:outputDir = Join-Path $TestDrive "reports"
        New-Item -Path $script:outputDir -ItemType Directory -Force
    }

    It "返り値のパスにファイルが存在する" {
        $path = New-DailyReport -OutputDirectory $script:outputDir
        $path | Should -Exist
    }

    It "ファイル名に今日の日付 (20260306) が含まれる" {
        $path = New-DailyReport -OutputDirectory $script:outputDir
        $path | Should -BeLike "*20260306*"
    }

    It "ファイルの内容に '日次レポート' が含まれる" {
        $path = New-DailyReport -OutputDirectory $script:outputDir
        $path | Should -FileContentMatch "日次レポート"
    }
}
```
