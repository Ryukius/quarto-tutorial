# nbstata で `.qmd` の Run Cell を使う

Quarto（`.qmd`）で Stata セルを対話実行するための手順です。

## Render と Run Cell の違い

| 操作 | エンジン | YAML `jupyter: nbstata` |
|------|----------|-------------------------|
| `quarto render` / Preview | Quarto → nbstata | 効く |
| Run Cell | Quarto 拡張 → Jupyter Interactive Window | 効かない（別途カーネル選択） |

**Interactive Window** は、セル実行結果を表示する Jupyter のパネルです。  
Run Cell はここにコードを送ります。カーネル（実行エンジン）は Interactive Window 側で選びます。

仕様上、Interactive Window の既定カーネルは Python です。  
`settings.json` や YAML で最初から nbstata に固定する公式な方法はありません。初回だけ **Stata (nbstata)** へ切り替えます。

## Interpreter と Kernel

| 種類 | 選ぶ場所 | 内容 |
|------|----------|------|
| Python Interpreter | ステータスバー / Python: Select Interpreter | どの Python（`.venv` など）を使うか |
| Jupyter Kernel | Interactive Window 右上 | 実行エンジン（Python / **Stata (nbstata)** など） |

nbstata は Jupyter Kernel として選びます。Interpreter 一覧には出ません。

## `.vscode/settings.json`（推奨・必須ではない）

カーネルを nbstata にする操作自体には不要です。  
ただし、使う Python や言語モードを揃えておくと安定しやすいので、プロジェクトに置くことを推奨します。

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "quarto.pythonPath": "${workspaceFolder}/.venv/bin/python",
  "jupyter.interactiveWindow.creationMode": "perFile",
  "files.associations": {
    "*.qmd": "quarto"
  }
}
```

| 設定 | 役割 |
|------|------|
| `python.defaultInterpreterPath` | Jupyter / 拡張が参照する Python を `.venv` に固定 |
| `quarto.pythonPath` | エディタからの Preview / Render でも同じ Python を使う |
| `jupyter.interactiveWindow.creationMode` | Interactive Window をファイル単位にし、カーネル記憶を効きやすくする |
| `files.associations` | `.qmd` を Quarto として開き、Run Cell を有効にしやすくする |

## 事前準備（初回・環境作り直し時）

```bash
source .venv/bin/activate
python -m nbstata.install --user
jupyter kernelspec list   # nbstata があること
quarto check jupyter      # Kernels に nbstata があること
```

`.qmd` の YAML:

```yaml
---
jupyter: nbstata
---
```

コードブロック:

````markdown
```{stata}
sysuse auto
regress price mpg weight
```
````

## Run Cell の手順

1. `.qmd` を開き、右下の言語モードが **Quarto** であることを確認する
2. **Run Cell**、または `Cmd+Shift+P` → **Quarto: Run Current Cell**
3. Interactive Window が開いたら、パネル右上のカーネルをクリックする
4. **Select Another Kernel…** → **Jupyter Kernel…** → **Stata (nbstata)**
5. もう一度 **Run Cell**

一度選べば、同じ Interactive Window（`perFile` なら同じファイル）では以降も nbstata が使われやすいです。

## Interactive Window が開かないとき

1. `Cmd+Shift+P` → **Developer: Reload Window**
2. 拡張機能を確認: **Quarto** / **Jupyter** / **Python**
3. `Cmd+Shift+P` → **Jupyter: Create Interactive Window** を実行し、  
   **Stata (nbstata)** を選んでから `.qmd` で Run Cell
4. 言語モードが Quarto 以外なら **Quarto** に変更

回避策として、ターミナルから Preview / Render も使えます。

```bash
uv run quarto preview stata-demo.qmd
# または
uv run quarto render stata-demo.qmd
```

## トラブルシュート

| 症状 | 対処 |
|------|------|
| `invalid syntax` | カーネルが Python → **Stata (nbstata)** に切替 |
| カーネル一覧に Stata (nbstata) がない | `python -m nbstata.install --user` 後に Reload Window |
| Quarto が nbstata を見つけない | Interpreter / `quarto.pythonPath` が `.venv` か |
| Stata 接続エラー | `.venv/etc/nbstata.conf` の `stata_dir`（`.app` の親ディレクトリ） |
