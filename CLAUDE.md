# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリの現状

**現時点ではコードは存在せず、計画文書のみのリポジトリ**である。ギター用アンプシミュレーター / マルチエフェクトプラグイン（NAM 互換、iPlug2 製）を商用販売する企画で、以下 3 文書が唯一の正である。作業前に必ず該当箇所を参照すること。

- **spec.md** — 機能仕様書（何を作るか）。シグナルチェーン、各セクションの仕様、NAM 技術の基礎、ライセンス方針
- **design.md** — 画面 UI 設計書（どう見せるか）。ゾーニング、共通コンポーネント、仮 UI → 本番 3D UI 差し替え方針
- **implementation.md** — 開発手順 / 実装計画書（どう作るか）。フェーズ分割、役割分担、タスクごとの受け入れ条件

実装コードは **`sdatkinson/NeuralAmpModelerPlugin` をフォーク**して持ち込む（まだ未実施 = implementation.md の P0）。コードが入るまで、下記の「ビルド」節のコマンドは実在しない点に注意。

## 開発の進め方

- **1 タスク = 1 ブランチ = 1 PR** の粒度。implementation.md 各フェーズの「Claude Code タスク」がそのままタスク分割になっている
- 各タスクでは「変更対象ファイル」と「受け入れ条件」を明確にする（implementation.md 記載）
- フェーズ構成: **Phase A（P0〜P6）= 個人利用アプリとして完成**（公開 NAM モデルで検証、配布はしない）→ **Phase B（P7〜P8）= 素材を自前キャプチャに総入れ替えして商用化**
- **公式 NeuralAmpModelerPlugin のソースを一次資料とする**。iPlug2 の公式ドキュメントは薄いため、実装判断は公式プラグインのコード（`mLayoutFunc` / 各カスタムコントロール / モデルロードのステージング交換）を読んで行う

## ビルド（コードをフォーク後に有効）

**CMake 一元管理ではない。Xcode と Visual Studio の 2 つのプロジェクトファイルを個別に持つ。**

- 主開発機は **macOS**（`NeuralAmpModeler.xcworkspace` / Standalone・VST3・AUv2）。Apple Silicon + Intel の Universal Binary
- 副開発機は **Windows**（`.sln` / Standalone・VST3）。ASIO または WASAPI
- 取得は `git clone --recursive`（サブモジュールは iPlug2 / eigen / AudioDSPTools / NeuralAmpModelerCore の 4 つ。iPlug2 は `sdatkinson/iPlug2` フォーク）。**NAMCore は A1/A2 両対応版**を使う（P0 検証時点は v0.5.3 で両対応済み。v0.5.4 化は P1 で動作確認済みの独立 PR として実施。詳細は implementation.md P1「製品リポジトリ化の方針」）
- 検証: pluginval 最高レベル（VST3）+ auval（AU, mac のみ）。CPU 計測は Core 付属の `benchmodel`
- CI（P1 で整備）: GitHub Actions で mac / Windows 両ビルドを毎 PR 実行

### ⚠️ ファイル追加時の必須ルール

ソースファイルを追加したら **`.xcodeproj` と `.vcxproj` の両方に登録する**。片方だけだと他方の OS でリンクエラーになる（CMake で自動解決されない）。CI がこの漏れを検出する。

## リアルタイム DSP の鉄則

音声スレッド（`ProcessBlock` 系）では以下を**禁止**する。違反はドロップアウト/クラッシュの原因になる。

- `new` / `delete`（動的メモリ確保・解放）
- ファイル I/O
- ロック取得（mutex 等）

モデルロード等の重い処理は公式のステージング交換パターンに従い、音声スレッド外で準備してからロックフリーに差し替える。UI へのレベル値受け渡し（メーター / VU 針 / チューナー）もロックフリーキューで行う。パラメータは全数スムージング。

## アーキテクチャの要点

### シグナルチェーン（spec §5）

```
[Input Gain] → [Noise Gate] → [Transpose] → [ペダルボード: COMP→OD1→OD2→MOD]
 → [アンプ (NAM + トーンスタック)] ×3切替 → [キャビ/IR (2マイク+外部IR)]
 → [9バンド GEQ (+HPF/LPF)] → [ECHO(テープエコー)] → [REVERB]
 → [Doubler] → [Output Gain]
```

各ブロックは個別バイパス可（クリックノイズ防止のクロスフェード付き `ChainBlock` 基底）。**初期リリースは Mono 入力のみ**（Stereo は将来）。

### NAM ハイブリッド構成（spec §8.1）

NAM キャプチャは特定ノブ設定の「スナップショット」なので、ノブを効かせるためにデジタル処理で挟む:

```
[入力トリム(=Gainノブ)] → [NAMモデル推論] → [デジタルトーンスタック(Bass/Mid/Treble/Presence, IIR)] → [Master]
```

NAM モデルは入力レベルに対する歪み方まで学習しているため、入力レベルの扱い（メーター/推奨レベル/キャリブレーション）が重要。

### A1 / A2 アーキテクチャ対応（spec §3.7）

- 最新 Core を採用し **A1 / A2 両対応**。A2 は 1 ファイルに複数サイズ（small/large 等）を持ち、CPU 余力で品質切替
- **モデル判定を `"A1"`/`"A2"` の文字列やサイズ数 2 個で決め打ちしない**。公式 Core のローダーと能力照会 API（サイズ・ブレークポイント取得）に委ねる
- 回帰テスト対象: A1-standard/lite/feather/nano、A2-small/large、ブロックサイズ・サンプルレート変更、モデル切替、不正ファイル（P3 以降 CI 化）

### UI（design.md）

- **仮 UI（フェーズ 1）→ 本番 3D UI（フェーズ 2）は「見た目の差し替え」だけで済む設計**にする。独自 IControl を「挙動（当たり判定・パラメータ接続）」と「描画（スキン）」に分離し、描画側を `FlatSkin`（仮）/ `SkeuoSkin`（本番）で差し替える
- レイアウトはデータ駆動（JSON: コンポーネント名/座標/サイズ/アセット ID）
- iPlug2 の **IGraphics + Skia** バックエンド。基準解像度 1200×950、比率固定スケーリング
- リンクウィジェット: アンプ⇔キャビは Clean/Crunch/Lead の 3 リグ。リンク ON で連動、OFF で自由組合せ

## `.gitignore` / 素材の扱い

- **モデル（`*.nam`）・IR（`*.wav`）・キャプチャ録音を製品リポジトリに入れない**。最初から `.gitignore` 対象。素材は別の非公開リポジトリ / ストレージで管理
- Phase A では TONE3000 等の公開モデル・無料 IR を**個人利用の範囲**で使用（ビルド同梱・配布・無償公開もしない）
- 各 PR のレビュー時、diff に素材が混入していないか確認する
- `.gitattributes` で改行コードを LF に統一。`#include` のファイル名は大文字小文字を正確に書く（OS でファイルシステムの扱いが異なる）

## ライセンス（商用販売の前提）

- フレームワーク iPlug2 は zlib 系で商用・クローズドソース可。フォーク時は**製品名 / メーカー名 / VST3 クラス ID / AU manufacturer・subtype / Bundle ID / インストーラ情報を自社向けに変更**する
- 依存ライブラリ（MIT の NAMCore/NeuralAmpModelerPlugin 含む）は `THIRD_PARTY_NOTICES.md` に表記を同梱
- 実在アンプ名・他社ブランドの名称/ロゴ/アートを使わない（「〜風」の独自名称）

## コミュニケーション

- **ユーザーへの応答は日本語。git コミットメッセージ / PR は英語。**
