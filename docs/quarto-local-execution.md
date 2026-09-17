# Codex / Work ローカル環境でのQuarto実行手順

作成日・動作確認日：2026年9月17日

## この手順で行うこと

既存のPython・Stata環境を使い、必要な入力ファイルを許可済みの一時フォルダに配置してQuartoを実行します。完成したHTMLと実行ログをプロジェクトへ戻します。

この方法は、Codex / Workのローカル実行で、uvのキャッシュ、StataのPNG生成、Quartoによる一時ディレクトリの削除が原因となってレンダリングに失敗した場合に使います。元の原稿や恒久的な環境設定を変更せず、その実行に必要な設定を指定します。

**動作確認済みの範囲は、macOS上のローカルタスク、Quarto＋nbstata＋Stata、単一のHTML出力です。** 他のプロジェクトでも同じ流れを使えますが、使用環境・入力ファイル・回収する成果物はプロジェクトごとに指定します。R、通常のPython、PDF、Webサイト・Book全体、Workのクラウド環境での成功を確認した手順ではありません。

通常のターミナルなどですでに問題なく実行できている場合は、既存の実行方法を継続できます。

## 1. 使用環境と入出力を決める

次の項目を確認します。

| 項目 | このプロジェクトでの例 | 他のプロジェクトで確認すること |
|---|---|---|
| 作業フォルダ | プロジェクトのルート | `_quarto.yml` のある場所 |
| 実行する原稿 | `stata-demo.qmd` | 原稿の相対パス |
| Python | `.venv/bin/python` | 必要なパッケージとカーネルがある環境 |
| Jupyterカーネル | `nbstata` | 原稿の `jupyter:` とカーネル登録が一致すること |
| Stata | `/Applications/StataNow`、MP | nbstata設定のパス・エディション、有効なライセンス |
| 入力ファイル | `_quarto.yml`、原稿 | データ、参考文献、画像、追加設定、フィルター、呼び出すスクリプトなど |
| 成果物 | `_output/stata-demo.html` | HTML以外に保存する表・図・加工データがあるか |

Python環境の作成・依存パッケージの導入・カーネル登録は、[READMEの導入手順](../README.md)で済ませておきます。この実行手順自体では、パッケージを追加・更新しません。

入力ファイルは必要なものを明示し、プロジェクトからの相対パスを保ってコピーします。`.venv` や `.git`、秘密設定ファイルを含めてプロジェクト全体をコピーする必要はありません。

原稿や設定が絶対パスを参照している場合、その参照先は一時フォルダに移りません。また、分析コードが元データを更新したり、指定した成果物以外の場所へ書き込んだりする場合は、その動作も事前に確認します。データに保存場所の指定がある場合は、その条件を満たす実行領域を使います。

## 2. 既存の実行環境を確認する

このプロジェクトでは、通常のターミナルまたはローカルタスクで次を実行します。他のプロジェクトでは最初のパスを置き換えます。

```bash
cd ~/coding-tutorial/quarto-tutorial
quarto --version
.venv/bin/python --version
.venv/bin/python -m jupyter kernelspec list
```

`nbstata` が一覧にあり、その登録先のPythonとStata設定が利用できることを確認します。`QUARTO_PYTHON` でPythonを指定しても、古いカーネル登録の起動先まで自動的に修正されるわけではありません。

Quartoから使用環境を確認する場合は、次を使います。

```bash
QUARTO_PYTHON="$PWD/.venv/bin/python" quarto check jupyter
```

## 3. 一時フォルダで実行し、成果物を戻す

以下は、このプロジェクトでそのまま実行できる例です。上の `cd` を実行した状態で使います。丸括弧の内側で処理するため、変数やシェル設定は通常のターミナルに残りません。

他のプロジェクトでは、冒頭の `document`、`python_bin`、`inputs`、`artifacts` を変更します。この例は `project.type: default`、`project.execute-dir: project` を前提とし、出力先を `_output`、形式を画像埋め込みのHTMLに指定します。

```bash
(
  set -euo pipefail

  project_dir="$PWD"
  document="stata-demo.qmd"
  python_bin="$project_dir/.venv/bin/python"

  # すべてプロジェクトからの相対パスで、ファイルを個別に指定します。
  inputs=("_quarto.yml" "$document")
  artifacts=("_output/stata-demo.html")

  test -x "$python_bin"
  command -v quarto >/dev/null

  for relative_file in "${inputs[@]}" "${artifacts[@]}"; do
    case "$relative_file" in
      ""|/*|..|../*|*/../*|*/..)
        printf 'プロジェクト内の相対パスを指定してください: %s\n' "$relative_file" >&2
        exit 1
        ;;
    esac
  done
  for relative_file in "${inputs[@]}"; do
    test -f "$project_dir/$relative_file"
  done

  build_dir="$(mktemp -d /private/tmp/quarto-local.XXXXXX)"
  mkdir -p "$project_dir/_output/local-render"
  run_dir="$(mktemp -d "$project_dir/_output/local-render/run.XXXXXX")"
  printf '一時フォルダ: %s\n結果の保存先: %s\n' "$build_dir" "$run_dir"

  for relative_file in "${inputs[@]}"; do
    mkdir -p "$(dirname "$build_dir/$relative_file")"
    cp "$project_dir/$relative_file" "$build_dir/$relative_file"
  done

  render_status=0
  (
    cd "$build_dir"
    QUARTO_PYTHON="$python_bin" \
    JAVA_TOOL_OPTIONS="${JAVA_TOOL_OPTIONS:+$JAVA_TOOL_OPTIONS }-Djava.awt.headless=true" \
      quarto render "$document" \
        --to html --output-dir _output -M embed-resources:true \
        --execute --no-cache --no-execute-daemon
  ) > "$run_dir/render.log" 2>&1 || render_status=$?

  if [ "$render_status" -ne 0 ]; then
    cat "$run_dir/render.log"
    printf '実行失敗。調査用に一時フォルダを残しました: %s\n' "$build_dir" >&2
    exit "$render_status"
  fi

  for relative_file in "${artifacts[@]}"; do
    if [ ! -f "$build_dir/$relative_file" ]; then
      printf '指定した成果物がありません: %s\n一時フォルダ: %s\n' \
        "$relative_file" "$build_dir" >&2
      exit 1
    fi
    mkdir -p "$(dirname "$run_dir/$relative_file")"
    cp "$build_dir/$relative_file" "$run_dir/$relative_file"
  done

  # この実行で作った一時フォルダだけを、回収完了後に削除します。
  rm -r -- "$build_dir"
  printf 'レンダリングと成果物の回収が完了しました: %s\n' "$run_dir"
)
```

実行ごとに異なる保存先を作るため、以前のHTMLを今回の結果と取り違えずに確認できます。元の `_output/stata-demo.html` は上書きしません。この例では、表示された保存先に `render.log` と `_output/stata-demo.html` ができます。

失敗した場合は一時フォルダを残します。ログを確認し、調査が終わってから、表示された該当フォルダだけを削除します。

### この例で使っている指定

| 指定 | 目的・適用範囲 |
|---|---|
| `QUARTO_PYTHON` | 既存のPython環境を明示します。実行のためにuvのキャッシュを開く必要をなくします |
| `JAVA_TOOL_OPTIONS` のheadless設定 | 今回のStataのPNG生成で失敗した、画面デバイスの初期化を避けます。その実行にだけ適用します |
| `/private/tmp` | 今回、ファイルとディレクトリの作成・削除が可能だった一時領域です。別環境では許可された場所を確認します |
| `--execute --no-cache` | 今回の原稿を実行し、実行結果のキャッシュを再利用しません |
| `--no-execute-daemon` | レンダリング後のカーネル常駐を無効にします。子プロセスの終了確認エラーを解決する指定ではありません |
| `embed-resources:true` | HTMLが参照する画像などを埋め込みます。コードが別途保存した表・データまですべて回収する指定ではありません |

### 外部ファイルがある場合の変更例

例えば原稿が `data/sample.dta` と `references.bib` を参照し、分析コードが `tables/results.csv` も保存する場合は、上の2行を次のように変更します。ファイル名は実際の構成に合わせます。

```bash
inputs=("_quarto.yml" "$document" "data/sample.dta" "references.bib")
artifacts=("_output/stata-demo.html" "tables/results.csv")
```

コードが出力するフォルダは、コード自身で作成するか、実行前に一時フォルダ内へ作成します。追加のYAML、拡張機能、CSS、Luaフィルター、読み込む `.do` などがある場合も、必要なファイルを含めます。大きなプロジェクトでは、この入力一覧と成果物一覧をプロジェクト内の実行スクリプトに保存すると管理しやすくなります。

## 4. 成功を確認する

終了コード0だけでなく、次を確認します。

1. 今回の `render.log` にセルの実行と `Output created` が記録されていること。
2. 今回の保存先にあるHTMLを開き、期待する分析結果、図、キャプションが含まれていること。
3. HTMLの本文に、設定エラーや実行失敗のメッセージが出力されていないこと。
4. 別途保存する表・図・加工データも回収できていること。

今回の `stata-demo.qmd` では、74観測の回帰表、燃費と価格の散布図、キャプション「燃費と価格」を確認しました。nbstataは設定エラーを通常の出力として表示し、Quarto自体は終了コード0となる場合もあるため、成果物の確認を省略しません。

手順3に掲載したコードそのものを実行し、新しいHTMLの生成、成果物・ログの回収、一時フォルダの削除まで確認しています。元の `stata-demo.qmd` は変更していません。

## 5. 終了時のJupyterエラーとプロセス確認

今回、次のエラーはHTML生成が成功した実行でも残りました。

```text
Exception during subprocesses termination
Operation not permitted (originated from sysctl())
```

これは、Jupyterカーネルの終了時に、子プロセスの一覧取得・終了処理が完了しなかったことを示します。今回の実測では、回帰表と散布図を含むHTMLは生成できています。一方、別途起動する並列計算のワーカーや外部プログラムがある処理では、それらの終了も確認する必要があります。

nbstataカーネル本体の残留は、**Macの通常の「ターミナル」アプリ**で次のように確認できます。Codex側のプロセス一覧取得に制限がある場合に使います。

```bash
ps -axo pid,ppid,lstart,etime,pcpu,pmem,command |
  awk 'NR == 1 || /-m[[:space:]]+[n]bstata([[:space:]]|$)/'
```

見出しだけなら、その時点でこの起動形式のnbstataカーネルは検出されていません。行が表示された場合は、別のノートブックの可能性もあるため、起動日時とPythonのパスを照合します。CPUが0%でも、終了したことを意味しません。

2026年9月17日の確認では、ユーザーがこのコマンドを実行し、見出しだけが表示されました。ただし成功回のPIDは記録していないため、名前の異なる補助プロセスを含むすべての関連プロセスの不在を確認したものではありません。厳密な確認が必要な処理では、実行中からプロセスを追跡します。

## エラーが出た場合

| 表示・症状 | 対応 |
|---|---|
| `uv` のキャッシュで `Operation not permitted` | この手順のように既存Pythonを直接指定します。`uv run --no-sync` だけでは今回の問題は解消しませんでした |
| PNG出力時に `Not a png file` | Javaのheadless設定がカーネル起動前から渡されているか確認します。今回のログでは `Picked up JAVA_TOOL_OPTIONS` が記録されました |
| `execute-results` などの削除で `PermissionDenied` | 原稿のあるプロジェクト内で実行していないか確認します。一時領域での実行・削除が可能かを確認します |
| データ・画像・追加設定が見つからない | 入力一覧と相対パスを確認します |
| 指定した成果物が見つからない | 原稿・設定の出力名や配置と、`artifacts` の指定を確認します |
| 終了時の `sysctl` エラー | 成果物の確認と、必要に応じたプロセス確認を行います。他の実行エラーと分けて扱います |

代替としてSVGを使う場合は、描画前の独立したStataセルに `*%set graph_format = svg` を置きます。この設定と `scatter` などのコマンドを同じセルにまとめないでください。SVGによるHTML生成も確認済みですが、作業フォルダでのディレクトリ削除制限は別途対応が必要です。

## 他の環境への適用

「使用環境を固定する」「必要な入力を配置する」「許可された場所で実行する」「成果物とログを回収する」という流れは共通化できます。ただし、次は個別に検証します。

- **通常のPython**：Python環境とカーネルを指定します。Stata向けのJava設定が必要とは限りません。
- **R／knitr**：R環境・パッケージ・描画方法を確認します。Jupyterを使わない構成には、今回のカーネル設定は適用しません。
- **PDF・Webサイト・Book**：追加の実行環境、複数の出力、相互参照、必要なリソースを含めて確認します。上の単一HTML用コードをそのまま使う範囲ではありません。
- **Workのクラウド環境**：ローカルMacのStata本体・ライセンス・Python環境を自動的には引き継ぎません。クラウド側の実行環境を別途確認します。

## 検証記録と参考資料

2026年9月17日の検証環境は、Quarto 1.10.18、Python 3.13.5、nbstata 0.8.6、ipykernel 7.3.0、psutil 7.2.2、`/Applications/StataNow` のMP版です。

今回のmacOS描画エラーを引き起こした個別のOS拒否ルールは未特定です。また、作業フォルダでの空ディレクトリ削除拒否は秘密ファイル保護の実装と整合しますが、その個別ルールとの対応は推定です。この手順は、権限制限の原因すべてを修正したものではなく、確認できた実行方法を整理したものです。

- [導入手順：README](../README.md)
- [Quarto：Python環境とカーネルの実行](https://quarto.org/docs/computations/python.html)
- [Quarto：プロジェクトの実行と作業ディレクトリ](https://quarto.org/docs/projects/code-execution.html)
- [nbstata：User Guide](https://hugetim.github.io/nbstata/user_guide.html)
- [Oracle：Javaのheadlessモード](https://www.oracle.com/technical-resources/articles/javase/headless.html)
- [Oracle：JAVA_TOOL_OPTIONS](https://docs.oracle.com/en/java/javase/22/troubleshoot/environment-variables-and-system-properties.html)
- [uv：キャッシュの動作](https://docs.astral.sh/uv/concepts/cache/)
- [OpenAI：Sandbox](https://learn.chatgpt.com/docs/sandboxing)
- [OpenAI：Workのローカルセキュリティ](https://learn.chatgpt.com/ja-JP/docs/enterprise/chatgpt-work-local-security)

詳細な比較実験のログは、検証した端末の `_output/stata-diagnostics/` に保存しています。このフォルダはGit管理対象外のため、共有先には存在しない場合があります。
