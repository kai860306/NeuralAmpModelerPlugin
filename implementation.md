# 開発手順 / 実装計画書 (implementation.md)

- バージョン: v0.8
- 更新日: 2026-07-14
- 対応文書: spec.md v0.8 / design.md v0.4
- 変更点(v0.8): **P0 を手作業で全完了し GO 判定(2026-07-14)**。P0 の全チェックボックスを完了([x])化。P0(Mac)の重複・順序の乱れを整理(Xcode 導入を P0-1 へ統合 / 「やり直し」ブロックを P0-3 に一本化 / オーディオ設定を P0-5 へ集約 / submodule 取得手順を単一正順に圧縮)。P0-2 / P0-6 を **fork-first(Fork → 自分の fork を clone)を主手順**に変更し二度手間を排除。submodule の存在確認一覧に **eigen** を追加(実クローンで submodule は 4 つ)。P1 に **「製品リポジトリ化の方針」節**を新設(fork ベース化 / 依存の SHA 固定 / upstream 追従方針 / Core バージョン判断)。
- 変更点(v0.7): P0-6 にWindowsでのStandalone Appビルド手順を追加。Developer PowerShell for VS 2022を使用し、クローン、CRLF対策、依存関係取得、MSBuildによるRelease/x64ビルド、起動・音出し確認までをコマンドラインで実行する手順を記録。
- 変更点(v0.6): P0 のmacOS環境構築手順を、リポジトリ直下の `setup_container.sh` を使用する手順に更新。`~/Desktop/developer` を基準にしたクローン、Xcode初期設定、Standalone Appのビルド・起動、確認コマンドを追加。`setup_container.sh` と `NeuralAmpModeler/scripts` 配下の配布用ビルドスクリプトの役割分担を明記。
- 変更点(v0.5): **主開発機を macOS に確定**。§0.3 / P0 / P1 / P5 / P6 を「mac 主体 + Windows はフェーズ末確認」の運用に更新。P1 に CI 整備タスクを追加。
- 変更点(v0.4): 確定した仕様(グローバルテンポ / リンクウィジェット / Mono 入力)をタスクに反映。P3 にキャビ 3 種 + リンクロジックを追加。
- 変更点(v0.3): 全フェーズを「あなた(手作業 / ビルド / テスト)」と「Claude Code(コーディング)」の役割分担付きステップに詳細化。P0 に環境構築 / NAM モデル入手手順、P7 にテスト信号の再生・録音 / NAM 学習(素材制作)を正式フェーズとして追加。
- 変更点(v0.2): フレームワークを JUCE から iPlug2 に変更。

---

## 0. 全体像

### 0.1 フェーズ一覧

| フェーズ | 内容 | 主担当 | 目安 |
|---|---|---|---|
| P0 | 環境構築 + フォーク・スパイク(音出し Go/No-Go) | **あなた** | 1〜2 日 |
| P1 | 製品化の土台(リネーム / 基盤把握 / Notices) | Claude Code + あなた | 1 週 |
| P2 | シグナルチェーン拡張(ゲート / IR 2 マイク) | Claude Code | 2 週 |
| P3 | アンプ 3ch + トーンスタック + プリセット | Claude Code | 2 週 |
| P4 | ペダル・EQ・ECHO・REVERB | Claude Code | 3〜4 週 |
| P5 | ユーティリティ(チューナー / メトロ / Transpose / MIDI) | Claude Code | 2〜3 週 |
| P6 | 仮 UI 完成(design.md 準拠)= **個人利用アプリ完成** | Claude Code + あなた | 2〜3 週 |
| P7 | **素材制作**(機材調達 → テスト信号の再生・録音 → NAM 学習 → IR 収録) | **あなた** | 機材調達含め 2〜4 週 |
| P8 | 商用化ゲート(素材入れ替え / 本番 UI / 署名 / QA) | 両方 | — |

- P0〜P6 = Phase A(個人利用・検証)。P7〜P8 = Phase B(商用化)。
- 権利面: Phase A では TONE3000 等の公開モデル / 無料 IR を**個人利用の範囲で**使用(ビルド同梱・配布・無償公開もしない)。モデル / IR は最初から `.gitignore`。

### 0.2 Claude Code 運用ルール

- リポジトリ直下に **`CLAUDE.md`** を置き、以下を常時参照させる: spec.md / design.md / implementation.md の場所、「音声スレッドで new / delete / ファイル IO / ロック禁止」等の DSP 原則、ビルドコマンド、`.gitignore` 方針(モデル / IR を含めない)、公式 NeuralAmpModelerPlugin のソースを一次資料とすること
- **1 タスク = 1 ブランチ = 1 PR** の粒度で依頼する(下記各フェーズの「Claude Code タスク」がそのままタスク分割)
- あなたのレビュー手順(毎 PR 共通): ①ビルドが通る ②Standalone で音出し・該当機能の手動テスト ③CPU 使用率が悪化していないか ④diff にモデル / IR 等の素材が混入していないか
- Claude Code には各タスクで「変更対象ファイル」「受け入れ条件」を必ず伝える(各フェーズに記載)

### 0.3 クロスプラットフォーム運用(Windows / macOS)

**ソースコードは単一。ビルドは各 OS のマシン上で行う**(Win: Visual Studio の .sln / mac: Xcode の .xcworkspace。実用上クロスコンパイルはしない)。macOS は Intel + Apple Silicon の Universal Binary でビルドする。

- **CI を早期に整備する(P1)**: GitHub Actions で Win / mac 両方を毎 PR 自動ビルドする。これにより「コンパイルが通るか」は常時両 OS で保証され、手元での二重ビルドは不要になる
- **主開発機は macOS**(AU の即時ビルド・auval 検証ができ、署名 / 公証環境を先行整備できるため)。日常開発・Claude Code の実行・DAW 確認は mac で行い、**Windows(副 OS)での機能確認は各フェーズの完了条件時**にまとめて行う。ただし以下の OS 差が大きい領域は、実装したその場で Windows でも確認する:
  - オーディオデバイス層(Win: ASIO / WASAPI ↔ mac: CoreAudio)— P0, P5
  - ファイルパス / ユーザーフォルダ(プリセット・モデルの保存先、パス区切り)— P3
  - MIDI デバイス — P5
  - UI の HiDPI / Retina 表示とフォント描画 — P6
  - プラグイン形式(VST3 は両方 / AU は mac のみ・auval 検証)— P1 以降のフェーズ末
- **フェーズ別の Windows 確認**: P0 = mac で環境確立 → P0-6 で Windows も一度ビルド・音出し / P1〜P4 = CI ビルド + フェーズ末に Windows で機能確認 / P5〜P6 = Windows での確認必須(ASIO・WASAPI / MIDI / HiDPI 表示)/ P7 = OS 非依存 / P8 = 両方フル QA
- ⚠️ **iPlug2 特有の注意: ソースファイルを追加したら Xcode と Visual Studio の両プロジェクトに登録が必要**(CMake 一元管理ではないため片方だけだと他方でリンクエラー)。CLAUDE.md に「ファイル追加時は .xcodeproj と .vcxproj の両方を更新する」ルールを明記し、CI がこの漏れを検出する
- その他: `.gitattributes` で改行コードを統一(LF)。#include のファイル名は大文字小文字を正確に書く(OS によりファイルシステムの扱いが異なる)。CPU プロファイルは Apple Silicon / Intel / Windows でそれぞれ記録(SIMD 経路が異なり NAM の負荷が変わる)。Windows の ASIO 対応には Steinberg ASIO SDK のライセンス同意が必要(無償。P8 チェックリスト参照)

---

## P0 — 環境構築 + フォーク・スパイク(あなたの手作業、1〜2 日)

**ゴール: 公式プラグインが自分の環境でビルドでき、公開 NAM モデルで音が出ることを確認する(フレームワーク採用の Go/No-Go)。このフェーズはコーディング不要。**

### P0-1. 開発ツールのインストールと Xcode 初期設定

**主開発機(Mac)**:
- [x] App Store から完全版 Xcode をインストールし、一度 GUI から起動して追加コンポーネントの導入を完了する → `xcode-select --install`
- [x] `xcodebuild` が単体の Command Line Tools ではなく完全版 Xcode を参照するように切り替える

```bash
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch
sudo xcodebuild -license accept
```

`xcodebuild -license accept` で、すでにライセンスへ同意済みと表示された場合は、そのまま次へ進む。

- [x] Xcode の参照先が完全版 Xcode になっていることを確認する

```bash
xcode-select -p
```

以下が表示されることを確認する。

```text
/Applications/Xcode.app/Contents/Developer
```

`/Library/Developer/CommandLineTools` が表示される場合は Command Line Tools が選択されているため、上の切り替えコマンドを再実行する。

- [x] Git(Xcode 付属で可)
- [x] Python 3.10+(iPlug2 のスクリプトと、後の NAM トレーナーで使用)
- [x] GitHub アカウントと SSH キー設定

**副開発機(Windows、P0-6 で使用)**:
- [x] Visual Studio 2022(「C++ によるデスクトップ開発」ワークロード)、Git for Windows、Python 3.10+

### P0-2. リポジトリの取得と依存関係のセットアップ(Mac)

> **fork-first を推奨**: 最初に `sdatkinson/NeuralAmpModelerPlugin` を自分のアカウントへ **Fork** し、その fork を clone すれば、P0 のクローンがそのまま P1 の製品リポジトリの種になり二度手間がない(詳細は P1「製品リポジトリ化の方針」)。純粋な使い捨てスパイクで良い場合のみ公式リポジトリを直接 clone する(その場合は P1 で fork 後に `git remote set-url origin <自分のfork>` で張り替える)。

- [x] GitHub 上で `sdatkinson/NeuralAmpModelerPlugin` を自分のアカウントへ **Fork** する
- [x] ターミナルを開き、デスクトップ上の `developer` フォルダへ移動する(無ければ作成する)

```bash
mkdir -p ~/Desktop/developer
cd ~/Desktop/developer
```

- [x] 自分の fork を、サブモジュールを含めてクローンする

```bash
git clone --recursive https://github.com/kai860306/NeuralAmpModelerPlugin.git
cd NeuralAmpModelerPlugin
```

使い捨てスパイクとして公式版をそのまま検証する場合は、クローン URL を公式リポジトリに置き換える。

```bash
git clone --recursive https://github.com/sdatkinson/NeuralAmpModelerPlugin.git
cd NeuralAmpModelerPlugin
```

- [x] サブモジュールの URL 設定を同期し、リポジトリ直下のセットアップスクリプトを実行する

```bash
git submodule sync --recursive
./setup_container.sh
```

実行権限に関するエラーが表示された場合は、以下で実行する。

```bash
bash ./setup_container.sh
```

`setup_container.sh` は、主に以下の処理をまとめて実行する。

1. Git サブモジュールの初期化と取得(ネスト含む)
2. iPlug2 が使用する SDK の取得
3. iPlug2 が使用するプリビルド依存ライブラリの取得

このスクリプトは名前に `container` を含むが、ローカルの macOS 環境でも使用できる。ただし、Xcode の設定やアプリのビルドまでは実行しない。

- [x] サブモジュールの取得状態を確認する

```bash
git submodule status --recursive
```

各行の先頭に `-` が付いていなければ取得済み。`-` が表示される場合は、再度以下を実行する。

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

- [x] iPlug2の依存ライブラリが取得されたことを確認する

```bash
ls iPlug2/Dependencies/Build/mac
```

環境やiPlug2のバージョンによって内容は異なるが、`bin`、`include`、`lib` などのディレクトリが表示されることを確認する。

- [x] 現在位置と主要なディレクトリを確認する

```bash
pwd
ls
```

`pwd` が以下のパスを返すことを確認する。

```text
/Users/<ユーザー名>/Desktop/developer/NeuralAmpModelerPlugin
```

`ls` の一覧に少なくとも以下が存在することを確認する(サブモジュールは iPlug2 / eigen / AudioDSPTools / NeuralAmpModelerCore の 4 つ)。

```text
NeuralAmpModeler
NeuralAmpModelerCore
AudioDSPTools
iPlug2
eigen
setup_container.sh
```

> `setup_container.sh` の実行中にエラーが発生した場合の再実行手順は、P0-3「ビルドに失敗した場合の再確認」に一本化した。そちらを参照。


### P0-3. Standalone Appのビルドと起動(Mac)

> Xcode のインストール・`xcode-select --switch`・ライセンス同意は **P0-1 で完了済み**。参照先が `/Applications/Xcode.app/Contents/Developer` でない場合は P0-1 の切り替えコマンドを再実行する。

- [x] Xcodeとビルドツールのバージョンを確認する

```bash
xcodebuild -version
git --version
python3 --version
```

- [x] NeuralAmpModelerのプロジェクトディレクトリへ移動し、現在位置を確認する

```bash
cd ~/Desktop/developer/NeuralAmpModelerPlugin/NeuralAmpModeler
pwd
```

以下のパスになっていることを確認する。

```text
/Users/<ユーザー名>/Desktop/developer/NeuralAmpModelerPlugin/NeuralAmpModeler
```

- [x] Xcodeプロジェクトとビルド設定ファイルが存在することを確認する

```bash
ls -ld ./projects/NeuralAmpModeler-macOS.xcodeproj
ls -l ./config/NeuralAmpModeler-mac.xcconfig
```

- [x] Release構成でmacOS Standalone Appをビルドする

```bash
xcodebuild \
  -project ./projects/NeuralAmpModeler-macOS.xcodeproj \
  -xcconfig ./config/NeuralAmpModeler-mac.xcconfig \
  -target APP \
  -configuration Release \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGNING_REQUIRED=NO \
  DEVELOPMENT_TEAM=
```

このコマンドでは、ローカルでの動作確認を目的としてコード署名を無効化する。配布用の署名と公証はP8で実施する。

- [x] ビルドログの最後に以下が表示されることを確認する

```text
** BUILD SUCCEEDED **
```

- [x] Standalone Appが生成されたことを確認する

```bash
ls -ld "$HOME/Applications/NeuralAmpModeler.app"
```

通常の出力先は以下。

```text
~/Applications/NeuralAmpModeler.app
```

指定場所に見つからない場合は、以下で検索する。

```bash
find \
  "$HOME/Applications" \
  ~/Desktop/developer/NeuralAmpModelerPlugin/NeuralAmpModeler/build-mac \
  -name "NeuralAmpModeler.app" \
  -type d \
  -print 2>/dev/null
```

- [x] Standalone Appを起動する

```bash
open "$HOME/Applications/NeuralAmpModeler.app"
```

初回起動時にmacOSからマイク入力の許可を求められた場合は許可する。

拒否した場合は、以下から設定を変更してアプリを完全に終了し、再起動する。

```text
システム設定
→ プライバシーとセキュリティ
→ マイク
→ NeuralAmpModelerをオン
```

> App 内のオーディオ設定(オーディオインターフェース / 入出力 ch / サンプルレート / バッファサイズ)は、音出しテストとまとめて **P0-5** で行う。

#### ビルドスクリプトの使い分け

P0の目的は、macOSのStandalone Appをビルドして音出しを確認することなので、基本的には上記の `xcodebuild` コマンドで `APP` ターゲットだけをビルドする。

| コマンドまたはスクリプト                                    | 用途                                   |
| ----------------------------------------------- | ------------------------------------ |
| `./setup_container.sh`                          | サブモジュールとiPlug2依存関係の初回セットアップ          |
| `xcodebuild ... -target APP`                    | Standalone Appだけをビルドする。P0と日常開発で使用    |
| `NeuralAmpModeler/scripts/makedist-mac.sh`      | APP、AU、VST3などの複数形式をまとめてビルドし、成果物を整理する |
| `NeuralAmpModeler/scripts/makeinstaller-mac.sh` | ビルド済み成果物からmacOSインストーラを作成する           |

`setup_container.sh` は依存関係のセットアップ用であり、アプリのビルドは行わない。

`NeuralAmpModeler/scripts/makedist-mac.sh` は、Xcodeプロジェクトの `All` ターゲットを使用するCI・配布向けのスクリプトである。Standalone Appの音出しだけを確認するP0では使用しなくてよい。

APP、AU、VST3などをまとめてビルドしてZIP成果物を作る場合は、依存関係とXcodeの準備後に以下を使用する。

```bash
cd ~/Desktop/developer/NeuralAmpModelerPlugin/NeuralAmpModeler/scripts
./makedist-mac.sh full zip
```

インストーラを含む配布用成果物を作る処理はP8で実施する。

```bash
cd ~/Desktop/developer/NeuralAmpModelerPlugin/NeuralAmpModeler/scripts
./makedist-mac.sh full installer
```

#### ビルドに失敗した場合の再確認

サブモジュールや依存関係に関するエラーが発生した場合は、リポジトリ直下へ戻ってセットアップを再実行する。

```bash
cd ~/Desktop/developer/NeuralAmpModelerPlugin

git submodule sync --recursive
git submodule update --init --recursive

./setup_container.sh
```

実行権限エラーが発生する場合は、以下を使用する。

```bash
bash ./setup_container.sh
```

依存関係だけを個別に再取得する場合は、以下を実行する。

```bash
cd ~/Desktop/developer/NeuralAmpModelerPlugin

(
  cd iPlug2/Dependencies/IPlug
  ./download-iplug-sdks.sh
)

(
  cd iPlug2/Dependencies
  ./download-prebuilt-libs.sh
)
```

古いビルド結果が原因と思われる場合は、ビルドディレクトリを削除して再ビルドする。

```bash
cd ~/Desktop/developer/NeuralAmpModelerPlugin

rm -rf NeuralAmpModeler/build-mac

cd NeuralAmpModeler

xcodebuild \
  -project ./projects/NeuralAmpModeler-macOS.xcodeproj \
  -xcconfig ./config/NeuralAmpModeler-mac.xcconfig \
  -target APP \
  -configuration Release \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGNING_REQUIRED=NO \
  DEVELOPMENT_TEAM=
```

生成されたAppが起動できない場合は、アドホック署名を行ってから再度起動する。

```bash
codesign \
  --force \
  --deep \
  --sign - \
  "$HOME/Applications/NeuralAmpModeler.app"

open "$HOME/Applications/NeuralAmpModeler.app"
```

### P0-4. NAM モデルの入手(検証用・個人利用)

- [x] TONE3000(tone3000.com)でアカウント作成 → 好みのアンプを検索 → `.nam` をダウンロード
- [x] **A1 と A2 の両方**を最低 3 個ずつ入手(モデルページの表記で世代を確認)。クリーン / クランチ / ハイゲインを揃えると後の 3ch 検証にそのまま使える
- [x] 置き場所: リポジトリ外(例: `~/NAM-models/`)。**リポジトリ内に置かない**
- [x] あわせて無料キャビ IR も数個入手(同じく個人利用・リポジトリ外)

### P0-5. 音出しテスト(オーディオ設定を含む)

- [x] ギター → オーディオインターフェース → PC を接続
- [x] App 内のオーディオ設定で、オーディオインターフェース / 入力 ch / 出力 ch / サンプルレート / バッファサイズを選択する。目安は以下

```text
Sample Rate: 44.1 kHz または 48 kHz
Buffer Size: 128 samples または 256 samples(まずは 128 目安)
Input: ギターを接続したオーディオインターフェースの入力
Output: モニタースピーカーまたはヘッドフォンを接続した出力
```

- [x] `.nam` をロードして演奏。A1 / A2、複数モデルで音質と CPU 使用率(アクティビティモニタ / タスクマネージャ)を記録
- [x] IR もロードして「アンプ + キャビ」の完成音を確認

### P0-6. Windows副開発機の環境確立

**ゴール: Windows上で公式NeuralAmpModelerのStandalone AppをRelease/x64構成でビルドし、オーディオインターフェースを使用して音が出ることを確認する。Visual StudioのGUIは使用せず、Developer PowerShell for VS 2022からコマンドラインで実行する。**

#### 事前準備

- [x] Visual Studio 2022をインストールする
- [x] Visual Studio Installerで以下を追加する

  - `C++によるデスクトップ開発`ワークロード
  - MSVC v143 C++ x64/x86ビルドツール
  - Windows 10 SDKまたはWindows 11 SDK
- [x] Git for Windowsをインストールする

  - Bashスクリプトを実行するため、Git Bashを含めてインストールする
- [x] Python 3.10以上をインストールする
- [x] 使用するオーディオインターフェースのWindows用ドライバをインストールする

  - 可能であれば、メーカー公式ASIOドライバを使用する

以降のコマンドは、Windowsのスタートメニューから以下を起動して実行する。

```text
Developer PowerShell for VS 2022
```

通常のPowerShellではなくDeveloper PowerShellを使用する。これにより、`msbuild`、MSVCコンパイラ、Windows SDKなどの環境変数が自動的に設定される。

#### 1. 開発ツールの確認

- [x] Git、Python、Bash、MSBuildが使用できることを確認する

```powershell
git --version
python --version
Get-Command bash
Get-Command msbuild
msbuild -version
```

`python`コマンドでバージョンが表示されない場合は、Python Launcherを確認する。

```powershell
py -3 --version
```

`Get-Command msbuild`でコマンドが見つからない場合は、通常のPowerShellを開いていないか確認する。必ず`Developer PowerShell for VS 2022`を使用する。

#### 2. developerフォルダの作成と移動

- [x] デスクトップ上に`developer`フォルダを作成し、移動する

```powershell
New-Item -ItemType Directory -Force "$HOME\Desktop\developer"
Set-Location "$HOME\Desktop\developer"
```

- [x] 現在位置を確認する

```powershell
Get-Location
```

以下のようなパスになっていることを確認する。

```text
C:\Users\<ユーザー名>\Desktop\developer
```

デスクトップがOneDriveで管理されている場合は、実際のパスが以下になることがある。

```text
C:\Users\<ユーザー名>\OneDrive\Desktop\developer
```

その場合は、以降のパスを実際の保存場所に読み替える。

#### 3. リポジトリのクローン

Windows版GitによるBashスクリプトのCRLF変換を避けるため、クローン時に`core.autocrlf=false`を指定する。

- [x] 自分の fork をサブモジュール込みでクローンする(fork-first。理由は P0-2 と同じ。P0 のクローンがそのまま P1 の製品リポジトリの種になる)

```powershell
git -c core.autocrlf=false clone --recursive git@github.com:<you>/NeuralAmpModelerPlugin.git
Set-Location .\NeuralAmpModelerPlugin
```

使い捨てスパイクとして公式版を直接検証する場合は、クローン URL を公式リポジトリに置き換える(P1 で fork 後に `git remote set-url origin <自分のfork>` で張り替える)。

```powershell
git -c core.autocrlf=false clone --recursive https://github.com/sdatkinson/NeuralAmpModelerPlugin.git
Set-Location .\NeuralAmpModelerPlugin
```

- [x] このリポジトリ内でGitの自動CRLF変換を無効にする

```powershell
git config --local core.autocrlf false
```

- [x] 設定を確認する

```powershell
git config --local --get core.autocrlf
```

以下が表示されることを確認する。

```text
false
```

- [x] 現在位置を確認する

```powershell
Get-Location
```

以下のようなパスになっていることを確認する。

```text
C:\Users\<ユーザー名>\Desktop\developer\NeuralAmpModelerPlugin
```

#### 4. サブモジュールの取得

- [x] サブモジュールのURLを同期し、ネストされたものを含めて取得する

```powershell
git submodule sync --recursive
git submodule update --init --recursive
```

- [x] サブモジュールの状態を確認する

```powershell
git submodule status --recursive
```

各行の先頭に`-`が付いていなければ取得済み。

`-`が表示される場合は、再度以下を実行する。

```powershell
git submodule sync --recursive
git submodule update --init --recursive
```

#### 5. Bashスクリプトの改行コード修正

Windows環境では、`.sh`ファイルがCRLF改行になると、Bash実行時に以下のエラーが発生することがある。

```text
$'\r': command not found
```

また、パスの末尾に`\r`が付加され、以下のようなエラーになることがある。

```text
cd: $'iPlug2/Dependencies/IPlug/\r': No such file or directory
```

- [x] リポジトリ直下の`setup_container.sh`をLF改行へ変換する

```powershell
bash -lc "sed -i 's/\r$//' setup_container.sh"
```

- [x] iPlug2配下の依存取得用BashスクリプトもLF改行へ変換する

```powershell
bash -lc "find iPlug2/Dependencies -type f -name '*.sh' -exec sed -i 's/\r$//' {} +"
```

- [x] 改行コードを確認する

```powershell
bash -lc "file setup_container.sh"
bash -lc "file iPlug2/Dependencies/IPlug/download-iplug-sdks.sh"
bash -lc "file iPlug2/Dependencies/download-prebuilt-libs.sh"
```

出力に以下が含まれていないことを確認する。

```text
with CRLF line terminators
```

#### 6. iPlug2依存関係のセットアップ

- [x] リポジトリ直下のセットアップスクリプトを実行する

```powershell
bash ./setup_container.sh
```

`setup_container.sh`は主に以下の処理を実行する。

1. Gitサブモジュールの初期化
2. iPlug2 SDKのダウンロード
3. iPlug2プリビルドライブラリのダウンロード

このスクリプトは依存関係の準備用であり、NeuralAmpModeler本体のビルドは行わない。

- [x] サブモジュールを再確認する

```powershell
git submodule status --recursive
```

- [x] Windows用プリビルドライブラリのディレクトリを確認する

```powershell
Test-Path .\iPlug2\Dependencies\Build\win
Get-ChildItem .\iPlug2\Dependencies\Build\win
```

`Test-Path`の結果が以下になれば、ディレクトリは存在する。

```text
True
```

- [x] 主要なディレクトリを確認する

```powershell
Get-ChildItem
```

少なくとも以下が存在することを確認する(サブモジュールは iPlug2 / eigen / AudioDSPTools / NeuralAmpModelerCore の 4 つ)。

```text
NeuralAmpModeler
NeuralAmpModelerCore
AudioDSPTools
iPlug2
eigen
setup_container.sh
```

#### 7. NeuralAmpModelerプロジェクトへ移動

- [x] NeuralAmpModelerディレクトリへ移動する

```powershell
Set-Location .\NeuralAmpModeler
```

- [x] 現在位置を確認する

```powershell
Get-Location
```

以下のようなパスになっていることを確認する。

```text
C:\Users\<ユーザー名>\Desktop\developer\NeuralAmpModelerPlugin\NeuralAmpModeler
```

- [x] Visual Studioソリューションが存在することを確認する

```powershell
Test-Path .\NeuralAmpModeler.sln
```

以下が表示されれば存在する。

```text
True
```

#### 8. Standalone Appのビルド

- [x] Developer PowerShellでMSBuildが使用できることを再確認する

```powershell
Get-Command msbuild
msbuild -version
```

- [x] Release/x64構成でStandalone Appだけをビルドする

```powershell
msbuild .\NeuralAmpModeler.sln `
  /t:NeuralAmpModeler-app `
  /p:Configuration=Release `
  /p:Platform=x64 `
  /m
```

1行で実行する場合は以下を使用する。

```powershell
msbuild .\NeuralAmpModeler.sln /t:NeuralAmpModeler-app /p:Configuration=Release /p:Platform=x64 /m
```

各オプションの意味は以下。

| オプション                      | 内容                     |
| -------------------------- | ---------------------- |
| `/t:NeuralAmpModeler-app`  | Standalone Appだけをビルドする |
| `/p:Configuration=Release` | Release構成でビルドする        |
| `/p:Platform=x64`          | 64bit版をビルドする           |
| `/m`                       | 複数CPUコアを使用して並列ビルドする    |

- [x] ビルド結果に成功が表示されることを確認する

英語環境では以下のように表示される。

```text
Build succeeded.
```

日本語環境では以下のように表示される場合がある。

```text
ビルドに成功しました。
```

失敗数が`0`であることも確認する。

#### 9. 生成されたexeの確認

通常、Standalone Appは以下に生成される。

```text
build-win\app\x64\Release\NeuralAmpModeler.exe
```

- [x] 実行ファイルが存在することを確認する

```powershell
Test-Path .\build-win\app\x64\Release\NeuralAmpModeler.exe
```

以下が表示されれば生成済み。

```text
True
```

- [x] ファイル情報を確認する

```powershell
Get-Item .\build-win\app\x64\Release\NeuralAmpModeler.exe
```

想定した場所に見つからない場合は、プロジェクト内を検索する。

```powershell
Get-ChildItem . -Recurse -Filter NeuralAmpModeler.exe |
  Select-Object FullName
```

#### 10. Standalone Appの起動

- [x] 生成されたStandalone Appを起動する

```powershell
& ".\build-win\app\x64\Release\NeuralAmpModeler.exe"
```

別プロセスとして起動する場合は以下を使用する。

```powershell
Start-Process ".\build-win\app\x64\Release\NeuralAmpModeler.exe"
```

#### 11. オーディオ設定

- [x] Standalone App内で以下を設定する

```text
Audio API: ASIOを優先
Device: 使用するオーディオインターフェース
Input: ギターを接続した入力チャンネル
Output: ヘッドフォンまたはスピーカーを接続した出力
Sample Rate: 44.1 kHzまたは48 kHz
Buffer Size: 128または256 samples
```

ASIOデバイスが表示されない場合は、オーディオインターフェースメーカーのWindows用ASIOドライバをインストールし、アプリを再起動する。

ASIOで問題が発生する場合は、動作確認用としてWASAPIも試す。

#### 12. NAMモデルのロードと音出し確認

- [x] ギターをオーディオインターフェースへ接続する
- [x] Standalone Appの入力と出力を設定する
- [x] 検証用の`.nam`ファイルをロードする
- [x] 入力メーターが反応することを確認する
- [x] ギターを演奏し、出力から音が出ることを確認する
- [x] A1モデルとA2モデルの両方を確認する
- [x] CPU使用率、サンプルレート、バッファサイズ、体感レイテンシーを記録する
- [x] IRをロードし、「NAMアンプ + キャビIR」の完成音を確認する

NAMモデルとIRは製品リポジトリ内へ入れず、リポジトリ外で管理する。

例:

```text
C:\Users\<ユーザー名>\NAM-models
```

#### 初回セットアップからビルドまでのコマンドまとめ

以下は、Developer PowerShell for VS 2022を起動した直後から実行する一連のコマンド。

```powershell
New-Item -ItemType Directory -Force "$HOME\Desktop\developer"
Set-Location "$HOME\Desktop\developer"

git -c core.autocrlf=false clone --recursive https://github.com/sdatkinson/NeuralAmpModelerPlugin.git
Set-Location .\NeuralAmpModelerPlugin

git config --local core.autocrlf false
git submodule sync --recursive
git submodule update --init --recursive

bash -lc "sed -i 's/\r$//' setup_container.sh"
bash -lc "find iPlug2/Dependencies -type f -name '*.sh' -exec sed -i 's/\r$//' {} +"

bash ./setup_container.sh

Set-Location .\NeuralAmpModeler

msbuild .\NeuralAmpModeler.sln `
  /t:NeuralAmpModeler-app `
  /p:Configuration=Release `
  /p:Platform=x64 `
  /m

Test-Path .\build-win\app\x64\Release\NeuralAmpModeler.exe

& ".\build-win\app\x64\Release\NeuralAmpModeler.exe"
```

#### クリーンビルドする場合

古いビルド結果を削除する。

```powershell
Set-Location "$HOME\Desktop\developer\NeuralAmpModelerPlugin\NeuralAmpModeler"

Remove-Item .\build-win -Recurse -Force -ErrorAction SilentlyContinue
```

再度ビルドする。

```powershell
msbuild .\NeuralAmpModeler.sln `
  /t:NeuralAmpModeler-app `
  /p:Configuration=Release `
  /p:Platform=x64 `
  /m
```

#### 依存関係のセットアップをやり直す場合

リポジトリ直下へ戻る。

```powershell
Set-Location "$HOME\Desktop\developer\NeuralAmpModelerPlugin"
```

サブモジュールを再取得する。

```powershell
git submodule sync --recursive
git submodule update --init --recursive
```

BashスクリプトをLF改行へ変換する。

```powershell
bash -lc "sed -i 's/\r$//' setup_container.sh"
bash -lc "find iPlug2/Dependencies -type f -name '*.sh' -exec sed -i 's/\r$//' {} +"
```

セットアップを再実行する。

```powershell
bash ./setup_container.sh
```

#### 完了条件

- [x] Developer PowerShell for VS 2022からクローン、依存取得、ビルドを再現できる
- [x] `NeuralAmpModeler.exe`がRelease/x64構成で生成される
- [x] Standalone Appが正常に起動する
- [x] ASIOまたはWASAPIで入力・出力デバイスを選択できる
- [x] `.nam`モデルをロードし、ギターの音が出る
- [x] WindowsでのCPU使用率、バッファサイズ、サンプルレートを記録している
- [x] 以後のWindows手動確認は原則として各フェーズ末に行う

### 完了条件(Go/No-Go)
- [x] ビルド〜音出しまで自力で再現できる(Mac / Windows 両方。手順をメモ化 → 後で CLAUDE.md に転記)
- [x] **NAM の音に製品化の価値を感じる**(ここが企画全体の Go/No-Go)

> ✅ **GO 判定済み(2026-07-14)**。Mac / Windows とも P0 を手作業で全完了。NAM の音に製品化の価値ありと判断し、P1 へ進む。

---

## P1 — 製品化の土台(1 週)

### P0→P1: 製品リポジトリ化の方針

**方針: `NeuralAmpModelerPlugin` を fork してそのまま製品リポジトリ化する(依存を全部組み直さない)。依存ライブラリはサブモジュールを P0 検証 SHA に固定し、実際に手を入れるものだけ後からフォークする。**

- **理想手順(今後 P0 からやる場合・二度手間ゼロ)**: P0 で最初に Fork → 自分の fork を clone。P0 のクローンがそのまま製品リポジトリになる。
- **今回の最短手順(P0 は直接 clone で完了済みの場合)**: 再クローン不要。
  1. GitHub で `sdatkinson/NeuralAmpModelerPlugin` を **Fork**
  2. 既存 P0 クローンで origin を自分の fork に張り替えて push(Mac / Windows 両方)

  ```bash
  git remote set-url origin git@github.com:<you>/NeuralAmpModelerPlugin.git
  git push -u origin main
  ```

  3. docs(spec.md / design.md / implementation.md / CLAUDE.md)をリポジトリへコピーして commit。**CLAUDE.md はリポジトリルート**に置く
  4. 現行 `my-guitar-plugin`(docs のみの repo)は docs 反映後にアーカイブ / 破棄
- fork なので GitHub 上で親リポジトリとの関係が自動で付く。`upstream` remote の追加は任意(張るなら `upstream` = `sdatkinson/NeuralAmpModelerPlugin`)。

**依存サブモジュールは P0 で検証した SHA に固定する**(常に最新へ追従させない。実クローンの実測値):

| サブモジュール | version / SHA |
|---|---|
| (base) NeuralAmpModelerPlugin | v0.7.15 / 96337e9 |
| NeuralAmpModelerCore | v0.5.3 / 9c7b185 |
| AudioDSPTools | v0.1.1 / 0827c6c |
| eigen | before-3.4-568 / ed8cda3 |
| iPlug2(`sdatkinson/iPlug2` フォーク) | v0.0.1-alpha-4019 / 66f9060 |

**「手を入れるものだけ後からフォーク」方針**(最初から全部フォークしない):

| ライブラリ | 方針 |
|---|---|
| eigen | フォークしない(上流の固定コミットのまま) |
| iPlug2 | iPlug2 本体へのパッチが必要になった場合のみフォーク |
| NeuralAmpModelerCore | Core 内部の推論処理 / API を変更する場合のみフォーク |
| AudioDSPTools | 汎用ライブラリ自体を改造する場合のみフォーク |

製品固有の DSP / UI(2 マイク IR、Position/Distance 補間、ChainBlock、トーンスタック、3 スロット、独自 IControl / スキン等)は**製品側の `src/dsp/` `src/ui/` に置く**。

**上流(upstream)追従は cherry-pick 限定**: Core 更新 / A1・A2 互換修正 / 音声スレッド・モデルロードのバグ修正 / iPlug2・OS・DAW 互換 / ビルド・署名関連のみ取り込む。UI 変更や公式 NAM 固有機能は取り込まない。差分が大きくなるため `upstream/main` 丸ごとのマージは避ける。

**`docs/upstream.md` を新設**: base と各サブモジュールの固定 SHA、ローカルパッチの有無を記録しておく。

**⚠️ Core バージョンの判断(要決定)**: spec / CLAUDE.md は Core **v0.5.4 以降**を想定しているが、P0 検証状態は **v0.5.3**(v0.7.15 同梱。A1/A2 とも動作確認済み)。v0.5.4 + Eigen 5.0.1 は上流 PR #650(2026-07-14 時点 open)。

- 選択肢 (a): **v0.5.3(P0 検証 SHA)で開始**し、Core v0.5.4 化は動作確認済みの独立 PR として実施 — **推奨**(P0 の動作実績を捨てない)
- 選択肢 (b): PR #650 相当(Core 0.5.4 + Eigen 5.0.1)を取り込み、Mac / Windows で再度 P0 相当のビルド・音出しテストを行う

### あなたのタスク
- [ ] 製品の仮名称を決める(後で変更可。ID 類に使用)
- [ ] `CLAUDE.md` の初版を作成(§0.2 / §0.3 の内容 + P0 で確立したビルド手順。「ファイル追加時は Xcode / VS 両プロジェクトを更新」ルールを含める)
- [ ] 各 PR のビルド / DAW ロード確認は mac で実施(Reaper 推奨。AU は Logic または `auval -v` でも確認)。フェーズ末に Windows で VST3 ロード確認

### Claude Code タスク
1. **製品識別子の変更**: `config.h` の `PLUG_*` 定数群(製品名 / メーカー名 / VST3 クラス ID / AU manufacturer・subtype / Bundle ID)。受け入れ条件: 旧 NAM プラグインと**同居して両方ロードできる**
2. **リポジトリ整理**: `src/dsp/`, `src/ui/` の骨組み、`.gitignore`(`*.nam`, `*.wav`, IR フォルダ)、`.gitattributes`(改行 LF 統一)、`THIRD_PARTY_NOTICES.md` 初版(iPlug2 / WDL / NanoVG / Skia / RtAudio / RtMidi / NAMCore / NeuralAmpModelerPlugin)
3. **CI 整備**: GitHub Actions で mac(Universal Binary)/ Windows の両ビルドを毎 PR 実行。ソースファイルの両プロジェクト登録漏れがリンクエラーとして検出されることを確認
4. **コードベース解説ドキュメント生成**: `docs/codebase.md` に ProcessBlock のシグナルフロー、モデルロードのステージング交換構造、ResamplingNAM、A2 Slim パラメータ、パラメータ追加の作法(`kNumParams` → `InitParam` → `OnParamChange`)をまとめさせる(あなたの学習用 + 以後の共通理解)
5. **練習タスク**: ダミーパラメータを 1 つ追加して UI ノブと接続 → 動作確認後に削除する PR

### 完了条件
- 自分の製品 ID で DAW にロードされ、セッション保存 / 復元が動く

---

## P2 — シグナルチェーン拡張(2 週)

### あなたのタスク
- [ ] 各 PR ごとに音質確認(特に位相反転・Pan の正しさは耳でチェック)
- [ ] 複数ポジション収録の無料 IR を用意(Position ブレンド検証用)

### Claude Code タスク
1. **ChainBlock 基底クラス**: バイパスをクロスフェード(10〜20 ms)で切り替える共通基底 + 単体テスト
2. **パラメータスムージング部品**: 1-pole の `SmoothedValue` を共通化し、既存パラメータへ適用
3. **IR 2 マイク化**(spec §9): `ImpulseResponse` を 2 系統化、各系統に Level / Pan / Phase、ミックス実装
4. **Position / Distance ブレンド**: IR マトリクス(ファイル命名規約 `mic_posX_distY.wav`)の読み込みと双線形補間ブレンド
5. **外部 IR の D&D**: IGraphics `OnDrop` 対応 + 再指定 UI
6. **ゲート仕様合わせ**: Threshold -90〜0 dB、検出位置をチェーン先頭に(spec §6.3)

### 完了条件
- `[Gate] → [NAM] → [IR×2ミックス]` で完成音。Phase / Pan / Position ブレンドが正しく効く

---

## P3 — アンプ 3ch + トーンスタック + プリセット(2 週)

### あなたのタスク
- [ ] P0-4 のモデルから Clean / Crunch / Lead 用を割り当て、切替の音を評価
- [ ] DAW セッション保存 → 再起動 → 復元の手動テスト(VST3 / AU 両方)

### Claude Code タスク
1. **モデルスロット × 3**: 公式のステージング交換を 3 スロット化、150 ms 等パワークロスフェード(切替中の CPU 増を計測ログ出力)
2. **トーンスタック**: NAM 後段に Bass / Middle / Treble / Presence(RBJ biquad)。アンプごとのマッピングテーブルを外部 JSON 化(後から手調整できるように)
3. **Gain ノブ**: NAM 前段入力トリム ±18 dB(spec §8.1)
4. **A2 品質切替**: Core API からサイズ候補を動的取得 → UI 反映、A1 時は無効化(spec §8)
5. **プリセット管理**: 自作 JSON(全パラメータ + モデル / IR パス + 品質設定)、`SerializeState` / `UnserializeState` 統合、`*` 表示、パス喪失時の再指定 UI(spec §13)
6. **キャビ 3 種 + リンクロジック**: Clean / Crunch / Lead 対応のキャビ 3 セットを保持し、リンク ON でアンプ切替に連動 / OFF で自由組み合わせ。OFF → ON 時は現在アンプ対応キャビへスナップ(spec §8.2 / design §6)。プリセットにリンク状態 + 両選択を保存

### 完了条件
- 3 アンプ切替 + 全ノブ + プリセット保存 / 呼出 / セッション復元

---

## P4 — ペダル・EQ・ECHO・REVERB(3〜4 週)

### あなたのタスク
- [ ] 各エフェクトの音質評価(実機 / 他プラグインとの比較メモ)
- [ ] フェーズ末に CPU プロファイル計測(48kHz/128、全 ON、A2 small / large)

### Claude Code タスク(1 効果 = 1 PR)
1. **共通 DSP 部品**: RBJ biquad、オーバーサンプリング(2〜4 倍ポリフェーズ)、LFO、フラクショナルディレイライン(+ 単体テスト)
2. **COMP**: エンベロープフォロワー + パラレル Mix
3. **OD1 / OD2**: オーバーサンプリング内クリッピング + プリ / ポストフィルタ
4. **MOD**: Tremolo / Vibrato / Chorus
5. **GEQ**: 9 バンド + HPF / LPF(spec §10.1)
6. **ECHO**: マルチタップ(ヘッド構成テーブル 4 種)、FB ループ内 BASS / TREBLE + サチュレーション、TAPE-MOD、SYNC = FREE / APP / TAP(spec §10.2。テンポはグローバルテンポ §6.2 を参照)
7. **REVERB**: Freeverb 系 + プリディレイ + 帯域フィルタ(FDN に置換可能な構造)

### 完了条件
- 全ブロック ON で spec §14 の CPU 目標(1 コア 15% 以下)

---

## P5 — ユーティリティ(2〜3 週)

### あなたのタスク
- [ ] チューナー精度の実測(既知ピッチの音源 / 実ギターで ±1 cent 確認)
- [ ] MIDI コントローラ実機での Learn 動作テスト(**Mac / Windows 両方**)
- [ ] **Windows でのオーディオ設定確認**(ASIO / WASAPI のデバイス列挙・切替・バッファ変更)

### Claude Code タスク
1. **グローバルテンポ**: 内部テンポの単一ソース化(下部バー BPM / TAP / ECHO APP / メトロノーム共有。spec §6.2 で確定済。プラグイン時はホスト追従を既定、手動オーバーライド可)
2. **チューナー**: 自己相関 / MPM、ロックフリーキューで UI へ、ミュート / Live Tuner / 基準ピッチ(spec §10.4)
3. **メトロノーム**: クリック音源埋め込み、拍子 / RHYTHM / SOUND / PAN、ホストテンポ連携(spec §10.5)
4. **Transpose**: Signalsmith Stretch 組み込み、`SetLatency` 報告、0 以外で警告表示
5. **Doubler**: 短ディレイ + 微小ピッチ変調(L/R 逆相)
6. **MIDI Learn**: 右クリックメニュー → CC 割当、一覧編集、JSON 永続化、プログラムチェンジ(spec §12)

### 完了条件
- spec §5〜§12 の機能が仮 UI で全て操作できる

---

## P6 — 仮 UI 完成(design.md 準拠、2〜3 週)

### あなたのタスク
- [ ] design.md との突き合わせレビュー(配置 / 操作感 / 状態表示)
- [ ] **Windows での表示確認**(HiDPI スケーリング 100 / 150 / 200%、フォント描画、マウス操作感)
- [ ] **日常使用開始**: 自分の練習 / 録音で常用し、不満点を GitHub Issue 化

### Claude Code タスク
1. タブナビ + グローバルバー + 下部バー(design §2〜4、ページの Hide/Show)
2. 独自 IControl 群を「挙動 / スキン」分離で実装、仮 UI は `FlatSkin`(design §10)
3. モーダル 4 種: チューナー / メトロノーム / SETTINGS / MIDI(design §8)
4. 状態表示ルール(design §9)、ショートカット(design §12)
5. リンクウィジェット UI(design §6。確定仕様: アンプ⇔キャビのセット連動 / 解除で自由組合せ。ロジックは P3-6 で実装済みのものに接続)

### 完了条件
- **個人利用アプリとして完成(Phase A 終了)**

---

## P7 — 素材制作(あなたの手作業。spec §16 を実施)

**ゴール: 商用同梱できる自前のアンプモデル(A2)と IR マトリクスを制作する。**

### P7-1. 準備

- [ ] 機材調達(spec §16.2): リアクティブロードボックス / リアンプボックス / インターフェース。⚠️ 真空管アンプを無負荷で通電しない
- [ ] キャプチャ対象実機の確保(§18-1): レンタル / スタジオ予約。Clean / Crunch / Lead 用
- [ ] テスト信号 `v3_0_0.wav` を公式(GitHub: sdatkinson/neural-amp-modeler のドキュメント)から入手
- [ ] トレーナー導入: `pip install neural-amp-modeler` → `nam` で GUI 起動を確認(GPU 無しなら公式 Colab をブックマーク)

### P7-2. テスト信号の再生と録音(1 セッティングあたり約 15 分)

1. 結線(spec §16.3): OUT1 → リアンプボックス → アンプ IN / アンプ SP OUT → ロードボックス(キャビシム OFF)→ IN1
2. DAW で `v3_0_0.wav` を配置、アンプを目的セッティングに(トーン系 12 時、Gain は狙いの位置)
3. **セッティング全ノブを写真記録**
4. レベル調整: ピーク -6〜-3 dBFS、処理は一切かけない
5. 録音: 48 kHz / 24 bit / モノ WAV。信号と同じ長さでトリム
6. ファイル命名: `<amp>_<ch>_gain<位置>_in.wav / _out.wav`。**入出力ペアは恒久保管**(spec §3.7: 将来の再学習用)

### P7-3. NAM モデルの学習

1. `nam` GUI で入力 WAV(v3_0_0)+ 出力 WAV を指定 → アーキテクチャ **A2**(既定)→ Train
2. **ESR を確認**: 0.01 未満 = 優秀 / 0.05 未満 = 実用 / 0.1 超 = 再収録(レベル・クリップ・トリムずれを疑う)
3. 実ギターで A/B 試奏(実機 vs モデル)。合格したら `.nam` にメタデータ(録音レベル / セッティング写真参照)を添えて素材リポジトリ(製品リポジトリとは別)へ
4. 3 アンプぶん + (任意)Gain 3〜5 段階(spec §16.6)

### P7-4. キャビ IR 収録

- [ ] ロードボックス THRU → 実キャビ → マイク → IN でサインスイープ収録 → デコンボリューション(spec §16.7)
- [ ] マイク 3 種 × 位置 5 点 × 距離 3 点のグリッド。P2 の命名規約 `mic_posX_distY.wav` に合わせる

### 完了条件
- Clean / Crunch / Lead の自前 A2 モデル(ESR < 0.05)+ IR マトリクス一式 + 保管された生 WAV

---

## P8 — 商用化ゲート

### あなたのタスク(手作業)
- [ ] Apple Developer Program 登録、証明書取得(macOS 署名 / 公証、Windows は Authenticode 証明書)
- [ ] Steinberg VST3 ライセンス契約書の提出、および Windows スタンドアロンで ASIO を使う場合は **Steinberg ASIO SDK ライセンス同意**(無償)
- [ ] 主要 DAW での QA(Logic / Cubase / Studio One / Reaper / Live)、実機 2〜3 台での動作確認
- [ ] EULA / プライバシーポリシー、販売チャネル整備、価格決定(§18-4〜6)

### Claude Code タスク
1. **素材総入れ替え**: TONE3000 / GPL 由来素材の完全排除を確認するチェックスクリプト(リポジトリ履歴含む)+ 自前素材の組み込み
2. **フォーク元 due diligence**: インストーラ製品情報 / macOS パッケージ ID / アイコン / プリセット名 / About 表示の総点検、`THIRD_PARTY_NOTICES` 最終化(MIT 表記含む)
3. 本番 UI 差し替え(`SkeuoSkin` + フィルムストリップ、design §10)
4. コピープロテクト / ライセンス認証(§18-4 の確定後)
5. インストーラ(Inno Setup / pkg)、CI での署名・公証・pluginval / auval 自動化

---

## 進め方のガイドライン

- 各フェーズは完了条件を満たしてから次へ。**P0 と P7 はあなたの手作業が主役**、P1〜P6 / P8 はClaude Code への発注とあなたのビルド / テストの反復
- iPlug2 のドキュメントは薄いため、**公式 NeuralAmpModelerPlugin のソースを一次資料**として読む(Claude Code にもそう指示する)
- CPU プロファイルは P0 から毎フェーズ記録
- モデル回帰テスト(A1×4 / A2×2 / ブロックサイズ / サンプルレート / 不正ファイル)は P3 以降 CI 化(spec §3.7)
- 素材(モデル / IR / 録音 WAV)は製品リポジトリに入れない。素材専用の非公開リポジトリ or ストレージで管理
