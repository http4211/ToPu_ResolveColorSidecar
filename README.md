# ToPu Blender → DaVinci Resolve Color Bridge

Blender でレンダリングした EXR などの画像を、DaVinci Resolve（Fusion）で
**Blender の画面と同じ見た目** にするためのツールセットです。

| パッケージ | 対象 | 役割 |
|---|---|---|
| **[ToPu_ResolveColorSidecar](ToPu_ResolveColorSidecar/)** | Blender アドオン | レンダリング後、出力フォルダに色の設定（JSON）と LUT（.cube）を書き出す |
| **[ToPu_BlenderColorLoader](ToPu_BlenderColorLoader/)** | DaVinci Resolve スクリプト | 書き出された設定を読み、Fusion に色変換ノードを自動で組む |

- 作者: [http4211](https://github.com/http4211)
- 対応: Blender 4.2 以降 / DaVinci Resolve 18 以降（無料版・Studio 版）
- 言語: 日本語 / English

> **English summary** — A Blender add-on bakes the scene's full display transform (view transform,
> look, exposure, gamma, curves, white balance) into a LUT next to your renders, and a DaVinci Resolve
> script reads it and sets up Fusion so linear EXRs look exactly like Blender's viewer.

---

## なぜ必要？

Blender が書き出す EXR は **リニア（未変換）** のデータです。Blender の画面で見えている
Khronos PBR Neutral / AgX / Filmic などの **ビュー変換は EXR には焼き込まれません**。
そのまま Resolve に読み込むと、暗く見えたり、明るい部分が白飛びしたり、色味が変わったりします。

さらに Blender のカラーマネジメントには Look・露出・ガンマ・カーブ・**ホワイトバランス** などがあり、
手作業で Resolve 側を合わせるのは大変です。このツールはそれを全部自動で再現します。

---

## しくみ

```
┌──────────── Blender ────────────┐        ┌──────────── DaVinci Resolve ─────────────┐
│ レンダリング                      │        │ Workspace > Scripts >                     │
│   ↓ (ToPu_ResolveColorSidecar)   │        │   ToPu_BlenderColorLoader                 │
│ 出力フォルダ/                     │ ─────▶ │                                           │
│   ├ shot_0001.exr ...            │        │ MediaIn → (合成) → BlenderView_Shaper     │
│   ├ BlenderColor.json  設定      │        │                  → BlenderView_LUT        │
│   └ BlenderView.cube   LUT       │        │                  → MediaOut               │
└──────────────────────────────────┘        └───────────────────────────────────────────┘
```

- `BlenderView.cube` は、Blender 自身の色処理に格子状のテスト色を通して作る **3D LUT** です。
  ビュー変換・Look・露出・ガンマ・カーブ・ホワイトバランスがすべて含まれます。
- LUT の入力は **log2 シェイパー**（既定 −12.5〜+12.5 段）なので、HDR の明るい値まで正しく扱えます。
- Resolve 側は、各クリップの **素材フォルダにある `BlenderColor.json`** を見つけて自動で設定します。

**精度:** Blender 5.3（Khronos PBR Neutral + ホワイトバランス）→ Resolve Studio 21.2 で、
8×8 ピクセル平均の差が約 **1.5 / 255**、平均色は一致することを確認しています。

---

## インストール

[Releases](https://github.com/http4211/ToPu_ResolveColorSidecar/releases) から 2 つの zip をダウンロードします。

| ファイル | 対象 |
|---|---|
| `ToPu_ResolveColorSidecar-x.y.z.zip` | **Blender 用** |
| `ToPu_BlenderColorLoader-x.y.z.zip` | **DaVinci Resolve 用** |

> 覚え方: **Sidecar = Blender が書き出す側**、**Loader = Resolve が読み込む側**

### 1. Blender（ToPu_ResolveColorSidecar）

1. `ToPu_ResolveColorSidecar-x.y.z.zip` は **展開せず** そのまま使います。
2. **編集（Edit）→ プリファレンス（Preferences）→ エクステンション（Get Extensions）/ アドオン（Add-ons）**
3. 右上の **∨** → **ディスクからインストール（Install from Disk...）** → zip を選択
4. **ToPu_ResolveColorSidecar** にチェックが入っていれば完了

確認: プロパティエディタの **出力（Output）** タブに **ToPu_ResolveColorSidecar** パネルが出ます。

### 2. DaVinci Resolve（ToPu_BlenderColorLoader）

**Windows（かんたん）**

1. `ToPu_BlenderColorLoader-x.y.z.zip` を展開します。
2. 中の **`install.bat`** をダブルクリックします。
3. Resolve を再起動します。

**手動コピー（Windows / macOS）**

展開したフォルダの `Scripts/Utility/*.lua` を次の場所にコピーし、Resolve を再起動します。

| OS | コピー先 |
|---|---|
| Windows | `%APPDATA%\Blackmagic Design\DaVinci Resolve\Support\Fusion\Scripts\Utility` |
| macOS | `~/Library/Application Support/Blackmagic Design/DaVinci Resolve/Fusion/Scripts/Utility` |

確認: **Workspace（ワークスペース）→ Scripts（スクリプト）** に次の 2 つが出ます。

- `ToPu_BlenderColorLoader - Apply to Timeline`
- `ToPu_BlenderColorLoader - Import Renders`

---

## 使い方

### Blender 側

1. いつも通りレンダリングします（F12 / アニメーションレンダリング）。
2. 完了すると、出力フォルダに `BlenderColor.json` と `BlenderView.cube` が自動で作られます。

**Properties → Output → ToPu_ResolveColorSidecar**

| 項目 | 内容 | 既定値 |
|---|---|---|
| レンダー後に書き出し（Write After Render） | レンダリングのたびに自動で書き出す | ON |
| LUT サイズ（LUT Size） | 3D LUT の格子数 | 65 |
| シェイパー最小 / 最大（Shaper Min / Max） | LUT がカバーする明るさの範囲（log2 の段数） | −12.5 / +12.5 |
| 今すぐ書き出し（Write Sidecar Now） | 今の設定で手動で書き出す | — |

- 色の設定が前回と同じなら、書き出しは省略されます（書き出しは 1〜2 秒程度）。
- F12 の静止画を手動で別フォルダへ保存したときは、そのフォルダに 2 ファイルをコピーしてください。

### Resolve 側

**プロジェクト設定:** カラーサイエンスを **DaVinci YRGB（カラーマネージメントなし）** にしてください。
RCM / ACES だと Resolve 側の変換が追加でかかり、見た目がずれます。

**新しく読み込む**

1. **Workspace → Scripts → ToPu_BlenderColorLoader - Import Renders**
2. Blender の出力フォルダを選択
3. メディアプールへの読み込み → タイムラインへの追加 → Fusion の設定まで自動で行います

**すでにタイムラインにある**

1. 対象のタイムラインを開く
2. **Workspace → Scripts → ToPu_BlenderColorLoader - Apply to Timeline**
3. 素材フォルダに `BlenderColor.json` があるクリップすべてに設定が入ります

- もう一度実行すると、既存ノードの設定だけを更新します（ノードは増えません）。
  Blender 側の色設定を変えて再レンダリングしたときは、もう一度実行してください。
- 結果は **Workspace → Console** に表示されます（例: `完了: 追加 3 / 更新 0 / スキップ 1`）。
- Fusion での合成作業は、`BlenderView_Shaper` より **前（リニアのまま）** で行ってください。

---

## よくある質問 / トラブルシューティング

**Q. 見た目が Blender と少し違う**
- プロジェクトのカラーサイエンスが **DaVinci YRGB** か確認してください。
- カラーページのグレード（ノード）は Fusion の後にかかります。比べるときはカラーノードを無効にしてください。
- タイムライン解像度とレンダー解像度が違うと、拡大縮小による小さな差が出ます。
- Blender の色設定を変えた後に **Apply to Timeline** を再実行したか確認してください。

**Q. 「スキップ」と出て設定されない**
- 画像と同じフォルダに `BlenderColor.json` と `BlenderView.cube` があるか確認してください。
- 画像だけを別フォルダに移した場合は、2 ファイルも一緒に移してください。

**Q. Workspace → Scripts にメニューが出ない**
- `.lua` のコピー先が `...\Fusion\Scripts\Utility` か確認し、Resolve を再起動してください。

**Q. Blender のコンポジターの処理はどうなる？**
- EXR に焼き込まれているので、そのまま反映されます。LUT はビュー変換（表示の変換）部分だけです。

**Q. 日本語のフォルダ名でも動く？**
- 動きます。

---

## 表示言語

| | 切り替え方 |
|---|---|
| Blender | Preferences → Interface → Translation で日本語を選び、「インターフェイス」「ツールチップ」「レポート」を有効化 |
| Resolve | Resolve の UI 言語に合わせます。「自動」の場合は OS の表示言語 |

---

## リポジトリ構成

```
.
├─ README.md                         ← このファイル
├─ build.py                          配布用 zip を dist/ に作るスクリプト
├─ ToPu_ResolveColorSidecar/         Blender アドオン
│  ├─ __init__.py
│  ├─ blender_manifest.toml
│  └─ README.md
├─ ToPu_BlenderColorLoader/          DaVinci Resolve スクリプト
│  ├─ src/
│  │  ├─ core.lua                    共通処理（両スクリプトの先頭に結合される）
│  │  ├─ apply.lua                   Apply to Timeline
│  │  └─ import.lua                  Import Renders
│  ├─ install.bat / uninstall.bat
│  └─ README.md
└─ dist/                             ビルド結果（zip + 配布用 README）
```

### ビルド

Python 3 で実行します（Blender 同梱の Python でも可）。

```bash
python build.py
```

`dist/` に次の 2 つが作られます。

- `ToPu_ResolveColorSidecar-<ver>.zip`（Blender の Install from Disk 用）
- `ToPu_BlenderColorLoader-<ver>.zip`（Resolve 用。`core.lua` + 各スクリプトを結合した `.lua`、README、インストーラー入り）

---

## 動作確認環境

- Windows 11 / Blender 5.3 / DaVinci Resolve Studio 21.2
- macOS は未検証です

## License

GPL-3.0-or-later

## Links

- Blender アドオン: https://github.com/http4211/ToPu_ResolveColorSidecar
- Resolve プラグイン: https://github.com/http4211/ToPu_BlenderColorLoader
