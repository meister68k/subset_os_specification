# `sh0`組み込み `PID` コマンド仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SH-L0-PID-001 |
| 版 | 0.1（実装前） |
| 対象 | シェル `sh0` 固有の組み込みコマンド `PID` |
| 関連仕様 | [`command_sh0_spec.md`](command_sh0_spec.md)、[`getpid_implementation_spec.md`](../kernel/getpid_implementation_spec.md) |

## 1. 目的と位置づけ

`PID`は現在実行中の`sh0`自身のProcess IDを表示する、シェル固有の特殊な組み込みコマンドである。大文字のコマンド名により、一般的な外部コマンド名と区別する。独立した実行ファイルやカーネル機能は追加しない。

`sh0`はコマンド解決時に組み込み名を外部プログラムより先に判定する。`PID`は大文字小文字を区別し、トークンが正確に`PID`と一致した場合だけ本コマンドとして扱う。たとえば`pid`や`Pid`はこの組み込みではなく、通常の外部コマンド名として扱う。

## 2. 起動形式

```text
PID
```

- 引数を取らない。余分な引数がある場合はusageを標準エラーfd=2へ出し、コマンドを実行せず次のプロンプトへ進む。
- `PID`は子Processとしてspawnしない。
- `sh0`は組み込み処理中も通常のProcessとして実行されていることを前提とする。

## 3. 動作

1. 引数がないことを確認する。
2. `btx::sys_getpid()`を呼び、呼び出し元である`sh0`自身のPIDを取得する。
3. PIDを10進表記し、`PID: `、PID、LFの順で標準出力fd=1へ出力する。
4. 出力成功後はシェルのコマンド処理へ戻り、次のプロンプトを表示する。

出力例：

```text
PID: 1
```

PID値は実行環境やProcess割り当てに従い、固定しない。`getpid`の仕様により通常のProcess実行中に取得失敗は発生しない。コマンドはProcess状態、子Process、cwd、FD構成を変更しない。

## 4. 入出力・エラー処理

- 標準出力fd=1へ出力し、`sh0`共通のファイルリダイレクトを適用する。たとえば`PID >pid.txt`はPID行をファイルへ出力する。
- `write()`が部分書込みを返した場合は残りを続けて書く。0、負値、または要求量を超える戻り値は出力失敗とする。
- 引数不正時は`sh: PID: usage: PID\n`相当をfd=2へ出し、`getpid` syscallを呼ばずに継続する。
- 出力失敗時は可能ならfd=2へ診断し、シェルを継続する。診断出力の失敗を再帰的に処理しない。
- 動的メモリ確保や`printf()`は使用しない。PIDの10進変換には固定長のローカルbufferを使う。

## 5. 使用API

| API | 用途 |
|---|---|
| `btx::sys_getpid()` | `sh0`自身のPID取得 |
| `btx::sys_write()`相当 | 標準出力・標準エラーへの出力 |

`getpid` syscallの番号、戻り値、ProcessManagerとの境界は[`getpid_implementation_spec.md`](../kernel/getpid_implementation_spec.md)に従う。`PID`コマンド専用のsyscallやilibc APIは追加しない。

## 6. 受入確認項目

1. `PID`で`sys_getpid()`が一度呼ばれ、`PID: <10進PID>\n`がfd=1へ出力される。
2. `PID`実行前後で`sh0`のPIDが一致し、子Processをspawnしない。
3. `PID extra`ではusageをfd=2へ出し、`sys_getpid()`を呼ばずシェルを継続する。
4. `PID >pid.txt`ではPID行だけがファイルへ出力され、処理後に親sh0のfd 1が復元される。
5. 部分書込みを処理し、write失敗時はシェルを継続する。
6. `pid`および`Pid`は`PID`組み込みとして扱われない。
