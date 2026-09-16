---
title: macOS＋VS Code／CursorでQuartoを使う：Stata・Python・Rの導入手順
category: Research Tips & Materials/Research-environments
tags:
created_at: '2026-09-16T17:31:29+09:00'
updated_at: '2026-09-16T19:28:10+09:00'
published: true
number: 842
---

作成者：ChatGPT

# README：macOS＋VS Code／CursorでQuartoを使う：Stata・Python・Rの導入手順

## この記事の目的と読み方

生成AIの支援を受けながら分析コードを書き、分析結果を文章・表・図とともに確認するために、Quartoを導入する。

主な原稿には`.qmd`を使い、HTMLとして出力して分析全体を確認する。再利用する処理や、独立して実行・配布する分析コードは、必要に応じて`.do`・`.py`・`.R`に分ける。

`.qmd`は通常、実行結果を原稿内に保存しない。そのため、コードを変えていないのに出力の更新だけで原稿のGit差分が生じる問題を避けやすい。

**Stata・Python・Rをすべて導入したり、すべて同じプロジェクトで使ったりする必要はない。** プロジェクトに応じて、必要な構成を選ぶ。

| 想定する使い方 | 基本となる構成 | 読む箇所 |
|---|---|---|
| Stata中心 | Quarto＋Jupyter＋nbstata | 共通設定、Python環境、Stata |
| Python中心 | Quarto＋Jupyter＋Python | 共通設定、Python環境、Python |
| Rのみ | Quarto＋knitr＋R | 共通設定、R |
| Stata中心でPythonも使う | StataとPythonの処理を分ける、またはPythonカーネル＋PyStata | Stata、Python、複数言語の併用 |
| Stata中心でRも使う | `.do`と`.R`を分けて連携 | Stata、R、複数言語の併用 |
| R中心でStataも使う | Quarto＋knitr＋Statamarkdown、またはスクリプトを分ける | R、複数言語の併用 |

python のバージョン管理は uv を前提としているが、venv でも同様のことが可能。

## 共通：エディタとQuartoを導入する

### エディタ

[VS Code](https://code.visualstudio.com/)または[Cursor](https://cursor.com/download)を対象としている。

Quartoの編集・実行に関する基本構成は共通。AIによる支援は、各エディタで利用する機能・拡張に応じて設定する。

### Quarto本体

[Quarto公式ダウンロード](https://quarto.org/docs/download/)から、macOS用インストーラを導入する。

インストール後にエディタを再起動し、ターミナルで確認する。

```bash
quarto --version
```

**Quarto本体と、エディタのQuarto拡張は別のもの。** 拡張だけでなく、Quarto本体も必要。

### 拡張機能

Cursor/VS Codeの拡張機能画面から、必要なものをインストールする。

| 拡張機能 | 拡張ID | 導入する場合 |
|---|---|---|
| Quarto | `quarto.quarto` | 共通 |
| Python | `ms-python.python` | Python、またはnbstataを使う |
| Jupyter | `ms-toolsai.jupyter` | Python、またはnbstataで部分実行する |
| R | `REditorSupport.r` | Rを使う |

Rだけで分析する場合、Python・Jupyter拡張は不要。

参考：[QuartoのVS Code連携](https://quarto.org/docs/tools/vscode/index.html)

## 共通：プロジェクトを作る

### 作業フォルダ

ターミナルで実行する。

```bash
mkdir -p ~/Projects/my-analysis
cd ~/Projects/my-analysis
```

エディタの「フォルダーを開く」から、このフォルダを開く。

以後、特に断りがなければ、ターミナルの作業場所はこのプロジェクトのルートとする。

### Quartoの設定

プロジェクト直下に`_quarto.yml`を作る。

```yaml
project:
  type: default
  output-dir: _output
  execute-dir: project
  render:
    - "*.qmd"

format:
  html:
    toc: true
    code-fold: true
    embed-resources: true
```

| 設定 | 意味 |
|---|---|
| `output-dir: _output` | HTMLなどの出力先をまとめる |
| `execute-dir: project` | レンダリング時の作業ディレクトリをプロジェクトルートにする |
| `render` | この例ではルート直下の`.qmd`を処理する |
| `toc: true` | 目次を付ける |
| `code-fold: true` | HTML上でコードを折りたためるようにする |
| `embed-resources: true` | 図などをHTMLに埋め込み、共有しやすくする |

HTMLへの埋め込みは共有に便利だが、図が多いとHTML自体は大きくなる。

`execute-dir: project`はレンダリング時の設定であり、エディタの部分実行時の作業ディレクトリまで一律に変更する設定ではない。

参考：[Quartoの作業ディレクトリ](https://quarto.org/docs/projects/code-execution.html#working-dir)


## Python環境をuvで用意する

この節は、次のいずれかに当てはまる場合に必要となる。

- Pythonで分析する
- nbstataを使ってStataの`.qmd`を実行する
- Rからreticulateを使ってPythonを利用する

**Rのみの構成や、R＋Statamarkdownの構成では、この節を省略できる。**

- uv自体のインストールについて、詳しくは[uv公式のインストール手順](https://docs.astral.sh/uv/getting-started/installation/)を参照する。

### Pythonのバージョンを指定する

ここではPython 3.13を使用する。Stataと連携する場合、Pythonの最新版を無条件に選ぶのではなく、使用するStataとの互換性を確認する。
（※現時点での最新 python は 3.14 だが、Stata 17 は python 3.13 までにしか対応していないため、下記では python 3.13 を使用して進める）

```bash
uv python install 3.13
```

Python本体もuvで導入するため、この手順のために別途Pythonのインストーラーを使う必要はない。[uvによるPythonのインストール](https://docs.astral.sh/uv/guides/install-python/)を参照する。

### プロジェクトのPython環境を作る

まだ`pyproject.toml`がない新規プロジェクトでは、ルートフォルダで次を実行する。

```bash
uv init --bare --python 3.13
uv python pin 3.13
uv add jupyter
```

各オプションの役割と、指定する理由は次のとおり。

- `--bare`：必要なPythonのバージョン条件や依存パッケージを記述する`pyproject.toml`だけを作成するようにして、README・サンプルコード・`.python-version`の作成やGitの初期化を行わないようにする。QuartoやStataを中心とする分析フォルダに、不要な雛形を増やさずにPython環境の管理を追加するために指定する。
- `--python 3.13`：初期化時にPython 3.13を指定し、`pyproject.toml`のPythonバージョン条件を設定する。

すでにuvで管理しているプロジェクトでは、初期化を繰り返さず、既存の設定に必要なパッケージを追加する。

作成される主なファイル・フォルダの役割は次のとおり。

| ファイル・フォルダ | 役割 |
|---|---|
| `.python-version` | このプロジェクトで使うPythonのバージョン指定 |
| `pyproject.toml` | プロジェクトの設定と、必要なPythonパッケージの宣言 |
| `uv.lock` | 解決された依存パッケージのバージョンなどを記録する |
| `.venv/` | 実際にPythonとパッケージを使う仮想環境 |

`uv add`は、依存関係の宣言・ロックファイル・仮想環境を更新する。通常の作業では、別途`python -m venv`や`pip install`を実行する必要はない。[uvのプロジェクト管理](https://docs.astral.sh/uv/guides/projects/)を参照する。


### Quartoとエディタが同じPython環境を使うようにする

VS Code／Cursorで「Python: Select Interpreter」を実行し、プロジェクト内の次のPythonを選択する。

```text
.venv/bin/python
```

ターミナルでQuartoを実行するときは、`uv run`を付ける。

```bash
uv run quarto check jupyter
```

別のPython環境が選ばれることを避けるため、この手順では、プロジェクトのルートで次も設定する。

```bash
export QUARTO_PYTHON="$PWD/.venv/bin/python"
uv run quarto check jupyter
```

この指定は、そのターミナルで有効になる。新しいターミナルを開いたときは設定し直す。

プロジェクトごとに参照先が変わるため、この絶対パスを共通のシェル設定に固定して書くことは避ける。

なお、ターミナルで設定した環境変数は、すでに起動しているエディタの拡張機能に自動で反映されるとは限らない。エディタの実行機能については、インタープリターやカーネルの選択も確認する。[Quartoの仮想環境の説明](https://quarto.org/docs/projects/virtual-environments.html)を参照する。

## Stata中心の`.qmd`を使う

### nbstataをインストールする

この構成では、QuartoからJupyterの仕組みを経由してStataを実行する。

```bash
uv add nbstata
uv run python -m nbstata.install --sys-prefix
uv run jupyter kernelspec list
```

一覧に`nbstata`が表示されることを確認する。

`--sys-prefix`を付けることで、カーネルをプロジェクトの仮想環境内に登録する。

nbstataの利用には、別途インストールされ、ライセンスが有効なStataが必要となる。Stata・Pythonの対応条件は、[nbstataのユーザーガイド](https://hugetim.github.io/nbstata/user_guide.html)を確認する。

### Stataの場所とエディションを設定する

設定ファイルも作成する場合は、次を実行する。

```bash
uv run python -m nbstata.install --sys-prefix --conf-file
```
- `--sys-prefix：登録先を、使用中のPython環境にする。今回ならプロジェクト内の`.venv`
- `--conf-file`：Stataの場所やエディションなどを指定する設定ファイルも作成する

この方法では、通常、次の場所に設定ファイルが作成される。

```text
.venv/etc/nbstata.conf
```

インストーラーが表示した保存先を確認し、Stataの場所とエディションを設定する。

たとえば、Stata/MPを`/Applications/Stata`にインストールしている場合は、次のようにする。

```ini
[nbstata]
stata_dir = /Applications/Stata
edition = mp
```

エディションには、使用している製品に合わせて`mp`・`se`・`be`を指定する。

ホームディレクトリなどに既存のnbstata設定がある場合は、そちらが優先される可能性がある。設定が反映されない場合は、[設定ファイルの説明](https://hugetim.github.io/nbstata/user_guide.html)に沿って読み込み先を確認する。

`.venv`は再作成できる環境なので、この中に書いた設定も再作成時には失われる。必要な設定内容は、プロジェクトのREADMEなどにも残しておく。

### 動作確認用の文書を作る

`stata-demo.qmd`を作成する。

````markdown
---
title: "Stataの動作確認"
jupyter: nbstata
---

## 回帰分析

Stata付属のデータを使って、回帰分析を実行する。

```{stata}
sysuse auto, clear
regress price mpg weight
```

## 散布図

```{stata}
*| label: fig-stata
*| fig-cap: "燃費と価格"

scatter price mpg
```
````

プロジェクトのルートで実行する。

```bash
uv run quarto render stata-demo.qmd
```

生成された`_output/stata-demo.html`をブラウザで開き、推定結果と図を確認する。

## Pythonの`.qmd`を使う

Stataの実行にPythonを使うだけなら、この節の分析用パッケージは不要。Pythonでも分析する場合に追加する。

### 分析用パッケージとカーネルを用意する

```bash
uv add ipykernel pandas matplotlib statsmodels
uv run python -m ipykernel install --sys-prefix \
  --name analysis-python \
  --display-name "Python (my-analysis)"
```

`analysis-python`は、Quartoから指定するカーネル名である。

### 動作確認用の文書を作る

`python-demo.qmd`を作成する。

````markdown
---
title: "Pythonの動作確認"
jupyter: analysis-python
---

## 回帰分析

動作確認用の小さなデータを使う。

```{python}
import pandas as pd
import statsmodels.api as sm

df = pd.DataFrame({
    "x": [1, 2, 3, 4, 5, 6],
    "y": [2, 4, 4, 7, 8, 9],
})

X = sm.add_constant(df["x"])
model = sm.OLS(df["y"], X).fit()

print(model.summary())
```

## 散布図

```{python}
#| label: fig-python
#| fig-cap: "動作確認用データの散布図"

import matplotlib.pyplot as plt

fig, ax = plt.subplots()
ax.scatter(df["x"], df["y"])
ax.set_xlabel("x")
ax.set_ylabel("y")
plt.show()
```
````

実行する。

```bash
uv run quarto render python-demo.qmd
```

このデータは環境の動作確認用であり、推定結果の実質的な解釈を目的とするものではない。

詳しくは[QuartoのPythonドキュメント](https://quarto.org/docs/computations/python.html)と[uvのJupyter連携ガイド](https://docs.astral.sh/uv/guides/integration/jupyter/)を参照する。

## Rの`.qmd`を使う

### Rをインストールする

[CRANのmacOS向けページ](https://cran.r-project.org/bin/macosx/)から、Macに合ったRをインストールする。

Apple Silicon搭載のMacでは、対応するarm64版を選ぶ。すでにRを導入済みであれば、既存の環境を確認して使う。

```bash
R --version
```

### 必要なRパッケージを入れる

Rのコンソールで実行する。

```r
install.packages(
  c("rmarkdown", "knitr", "languageserver"),
  repos = "https://cloud.r-project.org"
)
```

その後、ターミナルで確認する。

```bash
quarto check knitr
```

### 動作確認用の文書を作る

`r-demo.qmd`を作成する。

````markdown
---
title: "Rの動作確認"
engine: knitr
---

## 回帰分析

```{r}
model <- lm(mpg ~ wt + hp, data = mtcars)
summary(model)
```

## 散布図

```{r}
#| label: fig-r
#| fig-cap: "車重と燃費"

plot(
  mtcars$wt,
  mtcars$mpg,
  xlab = "Weight",
  ylab = "Miles per gallon"
)
```
````

実行する。

```bash
quarto render r-demo.qmd
```

Rだけで分析する場合は、uv・Python・Jupyter・nbstataは必要ない。[QuartoのRドキュメント](https://quarto.org/docs/computations/r.html)を参照する。

## 必要に応じて複数言語を併用する

複数の言語を同じプロジェクトで使うことと、同じ`.qmd`内で使うことは別である。

まずは、処理を言語ごとのスクリプトに分け、ファイルを介してつなぐ方法を検討する。変数やデータを同じ文書内で受け渡したい場合に、言語間の連携機能を使う。

### Stata中心でPythonやRの処理を追加する

たとえば、Pythonで前処理、Stataで推定、Rで作図する場合は、次のように分けられる。

```text
my-analysis/
├── _quarto.yml
├── report.qmd
├── .python-version
├── pyproject.toml
├── uv.lock
├── scripts/
│   ├── prepare.py
│   ├── analysis.do
│   └── figures.R
├── intermediate/
└── _output/
```

使わない言語のスクリプトを作る必要はない。

処理の例は次のとおり。

```bash
uv run python scripts/prepare.py
```

```bash
"/Applications/Stata/StataMP.app/Contents/MacOS/stata-mp" \
  -b do scripts/analysis.do
```

```bash
Rscript scripts/figures.R
```

Stataの実行ファイルの場所は、インストール先とエディションに合わせて変更する。

各処理の結果を確認した後、`report.qmd`で生成済みの表や図を読み込んでまとめる。レポートがPythonまたはnbstataを使う場合は、次を実行する。

```bash
uv run quarto render report.qmd
```

Rのみのレポートなら、次でよい。

```bash
quarto render report.qmd
```

ファイルを介した連携では、CSVなどの扱いやすい形式を使える。ただし、CSVにはStataの変数ラベル・値ラベルなどがそのまま保持されるわけではない。

また、スクリプトを配置しただけで、Quartoが自動的に順番どおり実行するわけではない。実行順序と受け渡すファイルを明確にしておく。

### StataとPythonを1つの文書内で使う

Pythonカーネルを使う文書から、PyStataを経由してStataを実行できる。

この場合、Stataが分析の中心であっても、**文書の実行エンジンはPython**になる。

必要なパッケージを追加する。

```bash
uv add stata-setup
```

以下はStata/MPの例。

````markdown
---
title: "PythonとStataの連携"
jupyter: analysis-python
---

## Stataとの接続

```{python}
import stata_setup

stata_setup.config("/Applications/Stata", "mp")
```

## Stataで分析する

```{python}
%%stata
sysuse auto, clear
regress price mpg weight
```

## StataのデータをPythonで受け取る

```{python}
from pystata import stata

df = stata.pdataframe_from_data()
df.describe()
```
````

`%%stata`は、そのPythonコードチャンクの先頭に置く。

`stata-setup`を追加するだけで、Stata本体がインストールされるわけではない。使用するStataの場所とエディションを指定する。[Stata公式のPyStataドキュメント](https://www.stata.com/python/pystata/)を参照する。

### R中心でStataを使う

Rを中心に使い、文書内でStataのコードも実行したい場合は、`Statamarkdown`を利用する方法がある。

Rでインストールする。

```r
install.packages(
  "Statamarkdown",
  repos = "https://cloud.r-project.org"
)
```

文書の例：

````markdown
---
title: "RとStataの連携"
engine: knitr
---

```{r}
#| include: false

library(Statamarkdown)
```

## Rで分析する

```{r}
summary(lm(mpg ~ wt + hp, data = mtcars))
```

## Stataで分析する

```{stata}
sysuse auto, clear
regress price mpg weight
```
````

Stataが自動検出されない場合は、セットアップ用のRコードで実行ファイルを指定する。

```r
knitr::opts_chunk$set(
  engine.path = list(
    stata = "/Applications/Stata/StataMP.app/Contents/MacOS/StataMP"
  )
)
```

ここで示したパスは、Statamarkdownからバッチ実行するための設定例である。使用するエディションやインストール先に合わせて変更する。

Statamarkdownでは、通常、Stataの各コードチャンクは別々のバッチ実行になる。前のチャンクで読み込んだデータや作成した変数が、そのまま次のチャンクに残るとは考えない。

処理をつなぐ場合は、必要なコードをまとめるか、ファイルに保存して受け渡す。`collectcode`によるコードの再実行も利用できるが、単一のStataセッションを維持する仕組みとは異なる。[Statamarkdownの説明](https://github.com/Hemken/Statamarkdown)と[実行ファイルの設定方法](https://users.ssc.wisc.edu/~dehemken/Stataworkshops/Statamarkdown/stata-engine-path.html)を参照する。

この構成では、Pythonやuvは必須ではない。

### R中心でPythonを使う

Rの文書からPythonを使う場合は、`reticulate`を利用できる。

Rでインストールする。

```r
install.packages(
  "reticulate",
  repos = "https://cloud.r-project.org"
)
```

Python環境は、前述の手順でuvを使って作成する。必要なPythonパッケージもuvで追加する。

```bash
uv add pandas
uv sync
```

文書の最初のRコードチャンクで、Pythonを初めて使う前に環境を指定する。

````markdown
```{r}
#| include: false

library(reticulate)
use_virtualenv(".venv", required = TRUE)
```
````

この指定は、プロジェクトのルートにある`.venv`を参照する。部分実行するときも、作業フォルダを確認する。

reticulateで環境を選ぶことと、uvで依存パッケージをインストールすることは別である。共有したプロジェクトを開いた直後などは、先に`uv sync --locked`で環境を復元しておく。[reticulateのPython環境の選択方法](https://rstudio.github.io/reticulate/articles/versions.html)を参照する。

## 日常の実行と結果確認

### 文書全体を実行する

Pythonまたはnbstataを使うプロジェクトでは、ルートフォルダから実行する。

```bash
export QUARTO_PYTHON="$PWD/.venv/bin/python"
uv run quarto render report.qmd
```

プロジェクト内の対象文書をまとめて実行する場合：

```bash
uv run quarto render
```

`uv run`は、プロジェクトの仮想環境を使ってコマンドを実行する。通常は、事前に仮想環境を手動で有効化する必要はない。[uvのコマンド実行の説明](https://docs.astral.sh/uv/concepts/projects/run/)を参照する。

Rだけを使うプロジェクトでは、通常どおり実行する。

```bash
quarto render report.qmd
```

### プレビューする

Pythonまたはnbstataを使う場合：

```bash
uv run quarto preview report.qmd
```

Rのみの場合：

```bash
quarto preview report.qmd
```

プレビューを終了するときは、ターミナルで`Control + C`を押す。

エディタの「Quarto: Preview」も利用できるが、この操作が自動的に`uv run`を経由するわけではない。ターミナルからは動くのにエディタから動かない場合は、エディタが選択しているPython・カーネル・環境変数を確認する。

コードの一部だけ実行する

`.qmd`でも、エディタの拡張機能を使って、コードチャンクや選択部分を実行できる。ただし、言語と実行エンジンによって仕組みが異なる。

- **Python**：Python・Jupyter拡張機能を使い、プロジェクトのカーネルで実行する。
- **R**：R拡張機能を使い、Rセッションにコードを送って実行する。
- **Stata＋nbstata**：JupyterのInteractive Windowでnbstataカーネルを選ぶ。文書の`jupyter: nbstata`だけでは、Run Cellの接続先が切り替わらない場合がある。具体的な手順は次節を参照する。
- **R＋Statamarkdown**：Stataチャンクのバッチ実行と、Rセッションでの部分実行は別の仕組みとして扱う。

部分実行は試行錯誤に便利だが、実行順序によっては、過去に作った変数や読み込んだデータに依存することがある。

仕上げには文書全体をレンダリングし、上から順番に実行できることを確認する。また、部分実行の結果が、そのまま生成済みHTMLに反映されるわけではない。

### StataのRun Cellで使うカーネルを選ぶ

Cursorで、文書全体のレンダリングは成功する一方、StataチャンクのRun CellではPythonの`SyntaxError: invalid syntax`が出る場合がある。これは、文書の実行設定とInteractive Windowの接続先を分けて確認する必要があるためである。

| 操作 | 実行経路 | カーネルの指定 |
|---|---|---|
| `quarto render`／QuartoのPreview | Quarto → nbstata | 文書のYAMLにある`jupyter: nbstata`を使う |
| Run Cell | Quarto拡張 → Jupyter Interactive Window | Interactive Window側でStata (nbstata)を選ぶ |

Interactive Windowは、コードの実行結果を表示する画面である。下部の「Jupyter」タブに表示される変数一覧（Name・Type・Size・Value）とは別の画面なので、取り違えないようにする。

また、Python InterpreterとJupyter Kernelの選択も異なる。

| 選択するもの | 役割 | 今回選ぶ対象 |
|---|---|---|
| Python Interpreter | Python拡張などが使うPython環境 | `.venv/bin/python` |
| Jupyter Kernel | セルを実際に処理する実行エンジン | Stata (nbstata) |

nbstataはJupyter Kernelとして選ぶ。Python Interpreterで`.venv`を選んだだけでは、Stataコードを実行できる状態にはならない。[VS Codeのカーネル選択の説明](https://code.visualstudio.com/docs/datascience/jupyter-kernel-management)も参照する。

部分実行の手順は次のとおり。

1. `.qmd`を開き、エディタの言語モードがQuartoになっていることを確認する。
2. Stataチャンクの「Run Cell」、またはコマンドパレット（⌘⇧P）の「Quarto: Run Current Cell」を実行する。
3. 開いたInteractive Windowのカーネル選択で、「Select Another Kernel…」→「Jupyter Kernel…」→「Stata (nbstata)」を選ぶ。選択ボタンは通常、この実行結果画面の上部にあるが、位置や表示名はエディタ・拡張機能のバージョンによって異なる。
4. 元の`.qmd`で、もう一度Run Cellを実行する。

接続先が`.venv (Python 3.13.x)`と表示されている場合は、Pythonカーネルが選ばれている。StataのコードをPythonとして解釈させると構文エラーになるため、Stata (nbstata)へ切り替える。

同じInteractive Windowを使い続ける間は、選択したカーネルで作業する。ウィンドウを作り直したりエディタを再起動したりした場合は、接続先を再確認する。

### Interactive Windowが開かない・nbstataが見つからない場合

Interactive Windowが表示されない場合は、次を確認する。

1. Quarto・Jupyter・Pythonの各拡張機能が有効になっているか確認する。
2. コマンドパレットで「Developer: Reload Window」を実行する。
3. 「Jupyter: Create Interactive Window」で実行結果画面を開き、Stata (nbstata)を選ぶ。
4. `.qmd`のRun Cellを実行し、そのコードが送られるInteractive Windowの接続先も確認する。別のウィンドウが開いた場合は、そちらでもカーネルを選ぶ。

nbstataが候補に出ない場合は、プロジェクトのルートで登録状況を確認する。

```bash
uv run jupyter kernelspec list
uv run quarto check jupyter
```

未登録なら、前述のプロジェクト内への登録を行い、エディタを再読み込みする。

```bash
uv run python -m nbstata.install --sys-prefix --conf-file
```

プロジェクト内に登録されていてもエディタが検出できない場合は、ユーザー共通の場所にカーネルを登録する方法もある。

```bash
uv run python -m nbstata.install --user
```

`--sys-prefix`はプロジェクトの仮想環境内、`--user`はユーザー共通の場所への登録である。`--user`でも、登録されるカーネルの起動先は、このコマンドを実行したプロジェクトのPythonになる。別プロジェクトからも見えるため、同名のnbstata登録を別の環境で上書きしたり、参照先の`.venv`を移動・削除したりした場合は再確認する。

また、ユーザー側にnbstataの設定ファイルが存在する場合は、`.venv/etc/nbstata.conf`より優先される。Stataの場所やエディションが反映されない場合は、起動できたnbstataのセルで`%status`を実行して設定の読み込み先を確認する。[nbstataの設定に関する説明](https://hugetim.github.io/nbstata/user_guide.html)を参照する。

部分実行の設定が済むまでは、ターミナルから文書全体を確認できる。

```bash
uv run quarto preview stata-demo.qmd
```

### エディタのプロジェクト設定をそろえる（任意）

プロジェクトの`.vscode/settings.json`に、次の設定を追加できる。Cursorでもこのファイルを使用する。すでにファイルがある場合は、既存の設定を残して必要な項目を追加する。

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "jupyter.interactiveWindow.creationMode": "perFile",
  "files.associations": {
    "*.qmd": "quarto"
  }
}
```

| 設定 | 役割 |
|---|---|
| `python.defaultInterpreterPath` | ワークスペースで最初に使うPythonの候補を`.venv`にする |
| `jupyter.interactiveWindow.creationMode` | ソースファイルごとにInteractive Windowを分ける |
| `files.associations` | `.qmd`をQuartoの言語モードで開く |

これらはnbstataカーネルを自動選択する設定ではない。すでにPython Interpreterを選択済みの場合、`python.defaultInterpreterPath`を書き換えるだけでは選択が変わらないため、「Python: Select Interpreter」で変更する。[Python拡張の設定リファレンス](https://code.visualstudio.com/docs/python/settings-reference)を参照する。

エディタからのPreview／Renderについても、プロジェクトのPythonが使われているかを別途確認する。ターミナルでの`QUARTO_PYTHON`の設定方法は、「Quartoとエディタが同じPython環境を使うようにする」を参照する。

### 生成AIと一緒に作業する

次の流れを基本にする。

1. `.qmd`に分析の目的・処理内容・コードを書く。
2. AIにコードの作成や修正を依頼する。
3. 文書全体をレンダリングする。
4. HTMLで表・図・説明のつながりを確認する。
5. 必要な結果やエラーをAIに渡し、修正する。

AIに原稿だけを渡した場合、実行結果まで確認できているとは限らない。推定結果の検討や作図の修正を依頼するときは、該当する出力も渡す。

再利用する処理が増えたら、`.do`・`.py`・`.R`に分け、`.qmd`を結果と説明をまとめる文書として使うこともできる。

## 原稿・環境・生成物を分けて管理する

### Gitで管理するファイルを分ける

Pythonを使う場合、次のファイルをGitで管理する。

- `.qmd`と`_quarto.yml`
- `.do`・`.py`・`.R`などの分析コード
- `.python-version`
- `pyproject.toml`
- `uv.lock`
- 環境構築や実行順序を記載したREADME

`.gitignore`の例：

```gitignore
.venv/
.quarto/
_output/
**/*_cache/
**/*_files/
.DS_Store
```

`.venv`は環境を再作成できるため、Gitには含めない。一方、`uv.lock`は依存関係を再現するために共有する。

`_output`を除外する例を示したが、完成したレポートをGitでも管理するかどうかは、プロジェクトの運用に合わせて決める。

Cursorで生成物などをAIの参照対象から除外したい場合は、`.cursorignore`を別途設定できる。これはCursor固有の機能であり、`.gitignore`と同じ目的の設定ではない。[Cursorのignoreファイルの説明](https://prod.cursor.com/docs/reference/ignore-file)を参照する。

### uvで環境を復元する

共有されたプロジェクトを取得した場合は、プロジェクトのルートで実行する。

```bash
uv sync --locked
```

`--locked`を付けると、既存のロックファイルに従って環境を構築し、`pyproject.toml`との不整合があればエラーになる。[uvの同期とロックの説明](https://docs.astral.sh/uv/concepts/projects/sync/)を参照する。

ただし、**カーネルの登録やnbstataの設定ファイルは、`uv.lock`だけでは復元されない。**

nbstataを使う場合は、環境を作り直した後に再登録する。

```bash
uv run python -m nbstata.install --sys-prefix --conf-file
```

Pythonの名前付きカーネルを使う場合：

```bash
uv run python -m ipykernel install --sys-prefix \
  --name analysis-python \
  --display-name "Python (my-analysis)"
```

プロジェクトの場所を移動した場合も、カーネルが古いPythonのパスを参照していないか確認する。

### パッケージを追加・削除する

Pythonパッケージの変更はuvに統一する。

追加する場合：

```bash
uv add パッケージ名
```

削除する場合：

```bash
uv remove パッケージ名
```

これにより、インストールした内容と`pyproject.toml`・`uv.lock`の記録をそろえやすくなる。

### 言語ごとに環境を記録する

| 対象 | 主に記録する内容 |
|---|---|
| Python | `.python-version`・`pyproject.toml`・`uv.lock` |
| R | Rのバージョン、必要に応じて`renv.lock` |
| Stata | Stataのバージョン・エディション、追加コマンドと導入方法 |
| Quarto | Quartoのバージョン |
| 言語間連携 | カーネル登録手順、Stataの場所、nbstataなどの設定 |

`.python-version`に`3.13`と書いた場合、3.13系の利用を指定するが、パッチバージョンまで固定するものではない。

より厳密にそろえる必要がある場合は、動作確認済みの完全なバージョン番号を`uv python pin`で指定する。`uv.lock`はPythonパッケージの依存関係を記録するもので、Python本体のバージョン固定の代わりにはならない。

また、uvがRパッケージやStataの追加コマンドまで管理するわけではない。Rパッケージの環境管理には、必要に応じて[renv](https://rstudio.github.io/renv/)を使う。

## パス管理とトラブルシューティング

### データのパスはプロジェクトを基準にする

個人のホームディレクトリをコードに直接書くことを避け、プロジェクト内の相対パスを基本にする。

Stata：

```stata
use "intermediate/analysis.dta", clear
```

Python：

```python
import pandas as pd

df = pd.read_csv("intermediate/analysis.csv")
```

R：

```r
df <- read.csv("intermediate/analysis.csv")
```

Quartoによるレンダリングでは、`_quarto.yml`の`execute-dir: project`で実行場所をそろえる。

一方、外部スクリプトを直接実行したり、エディタから部分実行したりするときは、その実行方法での作業フォルダを確認する。

Stataのスクリプトからプロジェクトのルートを扱う補助として、`repkit`の`reproot`を使う方法もある。必要な場合に、[reprootのドキュメント](https://worldbank.github.io/repkit/reference/reproot.html)に沿って設定する。

### よくある問題

| 症状 | 確認する点 |
|---|---|
| `quarto`が見つからない | Quarto本体をインストールしたか。エディタとターミナルを開き直したか |
| `uv`が見つからない | uvのインストールが完了し、実行ファイルの場所がPATHに反映されているか |
| Pythonパッケージが見つからない | `uv add`で追加したか。`uv run`やエディタがプロジェクトの`.venv`を使っているか |
| `uv sync --locked`が失敗する | `pyproject.toml`と`uv.lock`が整合しているか。両方が共有されているか |
| カーネルが見つからない | `uv run jupyter kernelspec list`で確認し、必要に応じて再登録する |
| Stata単体では動くがQuartoから動かない | nbstataの設定、Stataの場所・エディション、選択されたPython環境 |
| nbstataが起動しない | StataとPythonの対応バージョン、ライセンス、CPUアーキテクチャの組み合わせ |
| Rの文書を実行できない | `quarto check knitr`でRと必要なパッケージを確認する |
| Statamarkdownで前のチャンクのデータが使えない | チャンクごとに別のバッチ実行になる点を確認する |
| ターミナルでは動くがエディタでは動かない | エディタが選択した環境・カーネル・環境変数。仮想環境が作成済みか |
| 部分実行では動くが全文実行では失敗する | 過去のセッションに残った変数や、実行順序に依存していないか |
| データファイルが見つからない | 実行時の作業フォルダと相対パス |

最初は、使用する言語の小さな動作確認用`.qmd`をレンダリングし、環境が動いてから実際の分析コードを移す。