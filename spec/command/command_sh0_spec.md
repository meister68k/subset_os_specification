# 極小シェル sh0 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SH-L0-SPEC-001 |
| 版 | 1.1（スクリプト実行を追加） |
| 対象 | UNIXサブセットOSのシェルコマンド `sh0` |
| 関連仕様 | INIT-SPEC-001 v0.1、ILIBC-REF-001 v0.1、`command_sh0_pid_spec.md`、`../userland/btx_readline_spec.md`、`../kernel/ioctl_implementation_spec.md`、`chdir_implementation_spec.md`、`getcwd_implementation_spec.md`、`../kernel/getpid_implementation_spec.md`、`dup_implementation_spec.md`、`../kernel/pipe_implementation_spec.md`、`open_close_implementation_spec.md`、`process_management_spec.md`、`../kernel/syscall_entry_spec.md`、`../kernel/cooperative_scheduler_spec.md` |
| 状態 | これまでの合意事項に基づく提案。未合意の境界条件は本書に「暫定」と明記 |

## 1. 目的と位置づけ

極小シェル`sh0`は、対話入力またはスクリプトfileから1行ずつコマンドを受け取り、組み込みコマンドを処理するか、外部プログラムを起動して終了を待つ最小のコマンドランチャーである。OSの`spawn` / `wait` / `chdir` / `dup` / `open`、引数受渡し、標準入出力を検証することを主目的とする。

- シェルはユーザー空間プログラムであり、カーネルにコマンド構文解析を追加しない。
- 基本動作は **入力 → 行全体を解析 → `;`で区切られたコマンド単位を順に実行・待機 → 次の入力** とする。
- 動的メモリ確保なし。入力行と引数配列は固定長とする。
- ilibcの公開APIは増やさず、OS側のシステムコールラッパと既存ilibcを利用する。
- 初版で `exit` すればシェルは終了する。initから `wait` で起動されている場合、制御はinitへ戻る。

## 2. 起動形式と動作モード

実行ファイル名はファイルシステム上では `/bin/sh0` 、内部バイナリでは `sh0` とする。起動時引数がなければ対話モード、スクリプトpathnameが1つあればスクリプトモードとする。

```text
/bin/sh0 [script_path]
```

- **対話モード**では標準入力fd=0から行入力し、標準出力fd=1にプロンプトを表示する。
- **スクリプトモード**では`script_path`を読み取り専用で開き、ファイルからコマンド行を逐次読み込む。プロンプトと入力echoは出さない。`sh0 /etc/rc`のように起動する。
- 引数なしは対話モード、pathname 1個はスクリプトモードとする。pathname以外のoption、`-c`、2個以上の引数はusage errorとして終了コード1で終了する。単独の`-`は通常のpathnameとし、標準入力をスクリプトsourceにする特別扱いはしない。`-foo`形式のpathnameは`./-foo`のように指定する。
- シェル自身の標準入力・標準出力・標準エラーは継承されたfd 0/1/2を使う。スクリプトfdは別に開き、標準入力fd 0を置き換えない。対話モードのTTY判定と入力編集は`btx::readline()`に委譲する。

`sh0`のshebang対応は本書の対象外である。将来`spawn()`がshebangを解釈する段階では、インタープリターに`sh0`とスクリプトpathnameを渡す形を想定する。初期段階ではshebangの任意引数やscript positional parametersを扱わない。

### 起動引数

- `argc == 1`なら対話モード、`argc == 2`なら`argv[1]`をscript pathnameとしてスクリプトモードにする。
- `argc > 2`またはoption形式の先頭引数はusage errorとする。`-c`によるコマンド文字列実行、script positional parameters、環境変数展開は行わない。
- pathnameは一つだけ指定できる。`-`は標準入力の別名ではなく通常のpathnameとする。
- shebang起動の初期形は、interpreterとして起動されたsh0にscript pathnameが`argv[1]`として一つ渡る形を想定する。shebangの任意引数や元の呼出し側引数の伝搬は別途仕様化する。

## 3. 入力構文

### 3.1 行入力

- 1物理行を1コマンド列とする。`;`がなければ従来どおり1コマンド単位である。
- 改行はLF (`\n`) を受け付け、引数に含めない。
- 対話モードの入力は `btx::readline()` で読み取る。TTYではBS/DELによる末尾byte削除と入力echoを行い、非TTYでは入力をechoせずBS/DELも通常byteとして扱う。詳細は `spec/userland/btx_readline_spec.md` に従う。
- スクリプトモードは独自の固定長行読取処理でscript fdから読み取る。TTY判定、行編集、echo、標準入力fd 0へのアクセスは行わない。
- `std::isspace()`が真となる文字だけの行は何も実行せず次の入力へ進む。
- 先頭の空白類を除いた先頭文字が `#` の行はコメントとして読み飛ばす（暫定）。行末コメントは非対応。
- 入力バッファは128 byteとし、最大入力行長は **126 byte（LFを除く）** とする。LFと終端NULを含めて128 byteに収める。127 byte以上の本文は超過として扱い、行末LFまたはEOFまで読み捨て、1回だけエラーを報告して次の入力へ進む。
- CRLF専用の行末処理は行わない。`readline()` はLFだけを行末として除くためCRは入力データに残るが、単語分割では`std::isspace()`に従って空白区切りとして扱う。
- 対話モードでEOFが行頭で発生したときはシェルを終了する。改行なしの最終行が残っている場合は、その行を1回処理してから終了する。詳細は `btx::readline()` のEOF契約に従う。
- スクリプトモードではLFを行終端とし、CRは通常byteとして保持する。EOF前に本文が残っていれば最終行として1回処理して終了する。CRを除去せず、単語分割では`std::isspace()`に従う。
- 対話モードで`readline()`が`too_long`を返した場合、およびスクリプトモードで行長上限を超えた場合は、行を実行せず診断して次行へ進む。行読取中のNULまたはI/O errorは入力errorとしてシェルを終了コード1で終了する。
- スクリプトモードの診断には、可能な場合、script pathnameと1始まりの物理行番号を含める。行番号は入力行ごとに進め、構文error・実行errorを該当行へ結び付ける。
- スクリプトモードは`sys_open(path, O_RDONLY, 0)`でscript fdを取得する。読取には`sys_read(script_fd, &byte, 1)`を使い、LFまたはEOFまで1行を組み立てる。バッファ長、LF/EOF、NULの扱いは本節の対話入力規則と一致させる。
- 1 byte readの戻り値1はbyte取得、0はEOF、負値はerrorとする。要求byte数を超える正の戻り値は契約外としてinput errorにする。
- script fdの取得失敗は診断してstatus 1で終了する。読取中のI/O errorまたはNULはscript実行を中止してstatus 1で終了する。行長超過は残りをLF/EOFまで読み捨て、該当行を診断して次行へ進む。
- 正常なEOF、`exit`、fatal errorのいずれでもscript fdをcloseする。close失敗はshell status 1とする。ただし先行するfatal errorがある場合はそのerrorを主原因として保つ。
- スクリプトモードで外部コマンドをspawnするときはFD継承リスト`{0, 1, 2}`を指定する。これによりscript fdをchildへ漏らさず、redirect後の標準FDはそのまま継承する。対話モードの単独外部コマンドは従来どおり全FD継承を使う。pipeline stageはいずれのモードでも既定のpipeline契約に従い`{0, 1, 2}`だけを継承する。

### 3.2 単語の分割

- 単語区切りはilibcの`std::isspace()`が真となる文字とする。これは`"C"`ロケールのASCII空白類である。連続する区切りは1つとみなし、先頭・末尾の区切り文字は無視する。
- コマンドの引数は、リダイレクト演算子とpathnameを除いて**最大8個** (`argv[0]`を含む)。超過した行は実行しない（暫定）。
- `argv[argc] = nullptr` を必ず付ける（9要素のポインタ配列）。
- `'` と `"` は通常文字。文字列の結合、引用符の除去を行わない。
- `<`と`>`はリダイレクト演算子として解釈し、前後の空白の有無にかかわらず単語を分割する。クォート・エスケープ機能がないため、引用符内や`\`で囲まれた演算子も演算子として扱う。
- `;`はコマンド単位の区切り演算子として扱う。引用符・エスケープ機能がないため、単語やpathname内の`;`を通常文字として渡す方法はない。`|`は4.3節の2段パイプ演算子として解釈する。`\`、`$`、`*`、`?`、`&`、`(`、`)`は演算子として解釈せず、単語の通常文字とする。ワイルドカード・変数展開は行わない。
- 1行には最大8個のコマンド単位を書ける。各単位は単独コマンドまたは既定の2段パイプラインとし、`;`の前後に空白は不要とする。
- 先頭・末尾の`;`、連続した`;;`、空白だけの単位は構文エラーとする。構文エラーが行内に1つでもあれば、その行のコマンドを一つも実行しない。
- コマンド単位は左から順に実行する。各単位の組み込み処理または外部childの終了待ち（pipelineなら両stageの回収）が完了してから次へ進む。単位の実行時エラーは診断して次の単位へ進むが、`exit`が有効に実行された場合は残りを実行せずsh0を終了する。
- `cd`等の組み込みによるシェル状態の変更は後続単位に反映する。例：`cd /tmp; pwd`は`/tmp`を表示する。
- 空白を含む1個の引数を指定する方法はない。

例：

```text
入力: echo "hello world"
argv[0] = "echo"
argv[1] = "\"hello"
argv[2] = "world\""
argv[3] = nullptr
```

## 4. コマンドの解決と実行

### 4.1 組み込みコマンド

`sh0`で提供する組み込みコマンドは`exit`、`cd`、`pwd`、`PID`とする。リダイレクトは組み込みコマンドにも適用する。`PID`は大文字表記を含めて完全一致で判定し、シェル固有の特殊な組み込みであることを示す。

| 入力 | 動作 |
|---|---|
| `exit` | シェルを終了コード0で終了 |
| `exit` に引数がある | 引数不正を表示し、シェルは継続（暫定） |
| `cd path` | `chdir(path)` syscallを呼び、成功すればシェルProcessのcwdを変更 |
| `cd`（引数なし） | 使用方法を診断し、シェルは継続 |
| `cd` にpathが複数ある | 引数不正を診断し、`chdir()`を呼ばずシェルは継続 |
| `pwd` | 現在のcwdの絶対pathnameを標準出力へ1行出力 |
| `pwd` に引数がある | 引数不正を診断し、シェルは継続 |
| `PID` | `sh0`自身のProcess IDを標準出力へ1行出力 |
| `PID` に引数がある | 引数不正を診断し、シェルは継続 |

`cd`、`pwd`、`PID`は組み込みであり、子Processとしてspawnしてはならない。`export`、`set`、`echo`、`true`、`false`、`test`は組み込みとして実装しない。用意されたものは外部プログラムとして扱う。

#### `cd`の動作

- `cd`はpathnameをちょうど1つ受け取る。`HOME`環境変数や前回のdirectoryを保持する機能がないため、引数省略時の既定directoryや`cd -`による移動は提供しない。
- `cd path`は`btx::sys_chdir(path)`相当を1回呼び出す。directoryの存在確認、pathname解決、Process cwd参照の保持・交換は`chdir_implementation_spec.md`の責務とし、シェルが`stat()`やcwd inodeを直接操作しない。
- 絶対pathはrootから、相対pathは現在のシェルProcessのcwdから解決する。成功後のcwdは以降の相対path syscallで使われる。
- 成功時は出力しない。失敗時はfd=2へ`sh:`を付けて診断し、次のコマンド単位へ進む。失敗時のcwd状態は`chdir()` syscall仕様に従う（通常の検証失敗では不変だが、旧cwd参照release後の失敗では新cwdへ遷移済みの場合がある）。
- `-`を含むpathも通常のpathnameとして渡し、optionや「前回directory」として解釈しない。空白区切りのみの構文のため、空白を含むpathnameは指定できない。
- `spawn()`で起動される子Processは、Process管理仕様に従い`cd`後のcwdを継承する。

#### `pwd`の動作

- `pwd`は引数を取らない。引数が指定された場合はfd=2へ`sh:`を付けて使用方法を診断し、`getcwd()`を呼ばず次のコマンド単位へ進む。
- `pwd`は`btx::sys_getcwd(buf, capacity)`相当を呼び、成功時に返るNUL終端pathnameとLFを標準出力へ出力する。root cwdは`/`と表示する。
- 出力用固定長bufferは`PATH_MAX` byteとする。syscallの成功戻り値は終端NULを含む長さであり、出力時はNULを除いてpathnameを出力した後、LFを1 byte出力する。
- `pwd`はProcess cwdを変更しない。cwdが削除済み等でrootからpathnameを復元できない場合、またはpathname取得・出力に失敗した場合はfd=2へ診断し、シェルを継続する。
- 標準出力への書き込みに失敗した場合はfd=2へ診断し、シェルを継続する。

#### `PID`の動作

詳細は[`command_sh0_pid_spec.md`](command_sh0_pid_spec.md)に従う。`PID`は子Processを起動せず、`btx::sys_getpid()`で現在の`sh0` ProcessのPIDを取得して標準出力へ表示する。既存の組み込みコマンドと同様にリダイレクトを適用する。

### 4.2 リダイレクト

sh0は各コマンド単位につき入力リダイレクトを最大1つ、出力リダイレクトを最大1つ受け付ける。パイプラインでは入力リダイレクトは左側コマンド、出力リダイレクトは右側コマンドに限る。

| 構文 | 動作 |
|---|---|
| `command < path` | `path`を`O_RDONLY`で開き、コマンド実行中のfd 0へ接続 |
| `command > path` | `path`を`O_WRONLY | O_CREAT | O_TRUNC`、mode `0666`で開き、コマンド実行中のfd 1へ接続 |
| `command < in > out` | fd 0とfd 1の両方をそれぞれ接続 |

- `<` / `>`の直後には空でないpathnameをちょうど1つ指定する。空白を含むpathnameは指定できない。演算子の前後に空白がなくても認識する（例：`cat<in`、`echo hi>out`）。
- pathnameは通常のpathnameとして扱い、相対pathnameはProcess cwd、絶対pathnameはrootを基準に解決する。`-`で始まるpathnameもoption扱いしない。
- 各コマンド単位で入力・出力のいずれかに同種演算子が複数ある場合、pathname不足、余分なoperand、コマンド名なしの場合は構文エラーとする。行全体の構文と全単位の引数数を検査してからファイルを開くため、構文エラーではどの単位も実行せず、ファイルを作成・truncateしない。
- `>>`、`2>`等のfd番号指定、here-document、3段以上のパイプラインは非対応。`>>`および数字列が直結した`n>`形式は構文エラーとし、`n`をコマンド引数として扱わない。`2 >out`のように空白で分離された`2`は通常の引数とし、`>`は標準出力fd 1へ適用する。
- 演算子文字を通常の引数やpathnameとして渡す方法はない。クォート・エスケープは実装しない。

#### FDの接続と復元

リダイレクトはシェルProcessのfd tableを一時変更し、組み込み処理または`spawn()`時の子Processに継承させる。`dup2()`は使わず`dup()`で標準fdを保存・接続する。

1. 構文検査後、対象となる元のfd 0/1をそれぞれ`dup()`し、保存用fd番号を記録する。保存に失敗したらファイルを開かず終了する。
2. リダイレクトpathnameを入力行に現れた順で`open()`する。入力は`O_RDONLY, mode 0`、出力は`O_WRONLY | O_CREAT | O_TRUNC, mode 0666`とする。いずれかのopen失敗時は標準fdをまだ変更せず、開いたfdと保存用fdをcloseする。
3. 各リダイレクト対象について、対象の標準fdを`close()`し、openしたfdを`dup()`する。`dup()`は最小空きfdを返すため、0または1が他の低いfdで塞がれていることを確認し、返却値が対象標準fdと一致することを検査する。続けてopen元fdをcloseする。
4. 付け替えがすべて成功した後にコマンドを実行する。外部コマンドでは`spawn()`が変更後のfd 0/1を継承した時点で、親sh0のfdを直ちに復元する。その後、従来どおり同じ子を`wait()`する。初期同期spawnが子をwait時まで実行しないため、復元をwaitより先に行ってよい。
5. 組み込みコマンドでは処理完了後に復元する。`exit`が成功終了を選んだ場合も、シェル終了前に復元・一時fd解放を試みる。
6. 復元は対象fdの変更順にかかわらずfd番号の昇順（0、次に1）で行う。対象fdをcloseし、保存用fdを`dup()`して元の番号へ戻し、保存用fdをcloseする。

元のfd 0/1はsh0起動時に有効であることを前提とする。マッピング時には対象fdを先に空け、他のfd 0/1が必要な低位番号を占有するため、`dup(open_fd)`は目標番号を返す。期待したfd番号と異なる場合は内部状態異常として扱い、実行前に復元する。

接続途中の失敗ではコマンドを実行せず、すでに変更した標準fdを保存用fdから復元して一時fdをcloseする。復元に失敗して標準fd状態を保証できない場合は、sh0を継続せず異常終了する。pathnameを順にopenするため、先行する出力openで起きた作成・truncateは、後続openやsetupの失敗時もrollbackしない。これはopen後の永続filesystem変更を戻さない`open()`契約に従う。

### 4.3 パイプライン

```text
command1 [args ...] | command2 [args ...]
```

- 初版は1個の`|`で区切る2段パイプラインのみを受け付ける。先頭・末尾の`|`、連続する`||`、空のstage、3段以上は構文エラーとし、pipeや子Processを作成しない。
- `|`の前後に空白は不要とする（例：`cat|wc`）。`|`は単語分割とstage分割の両方を行う。
- 各stageは通常コマンドと同じ最大8引数（`argv[0]`を含む）とする。入力行全体の解析と全stageの引数数検査を終えてから、ファイルopenやpipe作成を開始する。
- 各stageは外部内蔵バイナリとしてspawnする。`exit`、`cd`、`pwd`、`PID`などのsh0組み込みはpipeline内で実行せず、診断して行全体を拒否する。シェル状態を変える組み込みやシェル終了をstageとして実行しない。
- 左stageのfd 1をpipe書込側へ、右stageのfd 0をpipe読出側へ接続する。fd 2は両stageとも継承元のままとする。
- pipeline stageは`spawn()`にfdリスト`{0, 1, 2}`を指定する。spawn前に親側fd 0/1を対応するpipe端点またはstageリダイレクト先へ付け替えるため、childは標準fdのみを継承し、pipeの不要端点や他の一時fdを保持しない。
- pipeline内のリダイレクトは左stageの`<`と右stageの`>`に各1つまで許可する。例：`sort <in | head >out`。右stageの`<`、左stageの`>`、同一stage内の重複リダイレクトは構文エラーとする。
- pipelineの2 stageを別々のProcessとしてspawnし、両方をspawnしてからwaitする。最初のstageをwaitしてから次をspawnしてはならない。すべてのchild終了statusを回収してから次の入力へ進む。stageの非0 statusはshell全体を終了させず、pipefailや`$?`も導入しない。
- 親Processは必要なchildをspawnした後に自分のpipe端点と一時fdを閉じる。childはfd 0/1/2だけを継承するため、不要なwriter参照が残ってEOFが妨げられることはない。
- 固定上限は2 stage、pipe 1本とし、動的メモリを使わない。

#### カーネル側の依存仕様

実装済みpipelineは、以下のカーネル契約を使用する。ABI、FD所有権、pipe I/O、Schedulerとの連携の詳細は各仕様書に従う。

1. [`pipe_implementation_spec.md`](../kernel/pipe_implementation_spec.md)がsyscall 42のpipe pair作成ABIと失敗時保証を定める。pipe read/write自体は[`pipe_io_implementation_spec.md`](../kernel/pipe_io_implementation_spec.md)に従う。
2. `spawn()`は継承fd番号を明示できる。pipeline stageはfd 0/1/2のみを指定し、不要なpipe端点をchildへ継承させない。通常の単独外部コマンドでは既存動作を保つため`spawn(name, argv, nullptr, 0)`を使い、全open fdを継承する。
3. [`pipe_io_implementation_spec.md`](../kernel/pipe_io_implementation_spec.md)が有限buffer、read/writeの部分転送、EOF・reader消滅時の失敗、pipe waiterのwait/wakeup条件を定め、[`cooperative_scheduler_spec.md`](../kernel/cooperative_scheduler_spec.md)が`WAITING(IO)`をSchedulerへ統合する。busy-waitで代用しない。
4. pipe endpoint、FileDescription、ProcessごとのFD参照の所有権と解放順、および各段階の失敗時処理は各kernel仕様に従う。構文不正時はpipeや子Processを作らず、資源準備に失敗した場合は取得済み資源を解放する。

### 4.4 実行ファイルの探索

環境変数 `PATH` は使用しない。実行対象の解決は実装段階で次のように扱う。

1. **現段階**：`spawn`は内蔵バイナリ名だけを起動できる。内蔵バイナリ名に`/`は含まれない。コマンド名をそのまま内蔵バイナリ名として渡し、該当しなければ起動失敗として扱う。`/bin/cat`等のパス起動はまだ提供しない。
2. **将来段階**：`spawn`はフルパスで実行ファイル名を受け取る仕様とする。内蔵バイナリ名とフルパスの対応・解決方法は未確定であり、ここでは定めない。`PATH`検索は行わない。
3. いずれの段階でも、受け渡しに必要な固定長バッファに収まらない場合はエラーとし、途中で切り詰めない。

| 入力 | 実行パス |
|---|---|
| `echo hello` | `echo` |
| `/bin/cat /etc/inittab` | 将来段階で`/bin/cat` |
| `./test` | 将来段階で`./test` |
| `tools/test` | 将来段階で`tools/test` |

`argv[0]`は入力されたコマンド名を維持する。現段階の内蔵バイナリ名解決ではこの値を`spawn`の名前引数にも用いる。将来段階でフルパス引数を別に渡す場合も、プログラムへ渡す`argv[0]`は入力されたコマンド名とする。

外部プログラムから行う相対pathname操作は、シェルのProcess cwdを子Processが継承することで`cd`後のdirectoryを基準にする。現段階の内蔵バイナリ名検索自体はcwdによる`PATH`検索を行わない。

### 4.5 実行・待機

1. 入力行を`;`でコマンド単位に分割し、各単位を単独コマンドまたは2段pipelineとして解析する。単位ごとに`argv[]`とredirect情報を構成する。
2. 行全体の構文、コマンド単位数、各stageの引数数を検証する。どれかが不正ならファイル・pipe・childに関する副作用を起こさず行全体を拒否する。
3. 検証後、コマンド単位を左から順に実行する。単独コマンドは既存のredirect手順を適用してchildをspawnし、終了をwaitする。組み込みはshell内で実行する。pipelineはpipe資源を準備し、左stage、右stageの順で両方をspawnする。
4. pipelineではspawn後に親がpipe端点と一時fdを閉じ、両childの終了をwaitして回収する。
5. 各単位の完了後、`exit`でシェル終了にならない限り次の単位へ進む。全単位の処理後、対話モードでは次のプロンプトを表示し、スクリプトモードでは次の物理行を読む。

単独コマンドの同時起動数は1、pipelineは2とする。バックグラウンド実行やジョブ制御はない。childは標準fd 0/1/2を継承し、pipelineではfd 0/1を接続先へ置き換える。

## 5. プロンプト・表示

- プロンプトは **`$ `** とし、標準出力fd=1に出力する（暫定）。
- 改行は付けず、入力を受け付ける前に出力を完了する。
- コマンドの通常出力は子プロセスが直接標準出力へ行う。シェルは出力を捕捉・加工しない。
- シェルからの診断は標準エラーfd=2に表示し、`sh: ...` を付ける。
- 行編集は `btx::readline()` が提供するBS/DELによる末尾byte削除に限る。履歴、補完、カーソル制御、UTF-8文字単位編集は行わない。入力echoも `readline()` に委譲し、console device側では行編集しない。

使用例：

```console
$ echo Hello, World
Hello, World
$ cat /etc/inittab
boot::wait:/bin/echo Starting
shell::wait:/bin/sh
$ cd /tmp
$ cat notes.txt
$ cat < input.txt > output.txt
$ exit
```

## 6. エラー処理・終了ステータス

Level 0では、**コマンド単位の失敗でシェルを終了しない**ことを基本原則とする。次の表は初版の推奨規則（暫定）。

| 事象 | 動作 |
|---|---|
| 空行・コメント | 無視して次の入力へ |
| 入力行長超過 | 残りを読み捨てて診断し、次の入力へ |
| 先頭・末尾の`;`、`;;`、空のcommand unit | 行全体を実行せず診断して継続 |
| コマンド単位数が8個を超える | 行全体を実行せず診断して継続 |
| 引数8個超過 | 診断して実行せず、次の入力へ |
| 実行パスが長すぎる | 診断して実行せず、次の入力へ |
| `spawn`失敗（存在しないコマンドなど） | 診断して次のコマンド単位へ。後続がなければ次の入力へ |
| pipeline構文不正 | pipe/ファイル/childを作成せず診断して継続 |
| pipe作成・fd接続・redirect準備失敗 | 一時fdとpipe端点を解放して継続。復元不能なら起動済みchildを回収後に異常終了 |
| pipelineの2つ目の`spawn()`失敗 | 親fdとpipe端点を復元・解放し、起動済みchildをwaitして回収してから継続 |
| pipeline stageの非0終了 | 全stageをwaitで回収し、次のコマンド単位へ |
| 子が非0終了 | 子の終了コードをシェル全体のエラーとはみなさず、次のコマンド単位へ |
| `wait`失敗 | 診断し、同じ子を再度`wait`する。回収に成功するまで次の入力へ進まない |
| リダイレクト構文不正 | 診断し、ファイルを開かず次の入力へ |
| リダイレクトpathnameの`open()`失敗 | 一時fdを解放して診断し、この単位を実行せず次の単位へ進む。先行openの作成/truncateは維持 |
| リダイレクトの`dup()` / `close()`失敗 | 標準fdを復元して診断。復元不能なら子の回収後にシェル異常終了 |
| リダイレクト付き`spawn()`失敗 | 親の標準fdを復元し、一時fdを解放して診断・継続 |
| `cd`の引数なし・複数path | 診断し、`chdir()`を呼ばず次のコマンド単位へ |
| `chdir()`失敗 | 診断して次のコマンド単位へ。cwd状態はsyscall契約に従う |
| `exit` | 正常終了（0） |
| 対話入力の先頭でEOF | 正常終了（0） |
| 対話入力中のI/Oエラー | 診断してシェル異常終了（1） |
| script pathnameのopen失敗 | 診断してシェル異常終了（1） |
| script入力のEOF | script fdをcloseして正常終了（0）。改行なしの最終行は1回実行する |
| script入力中のread errorまたはNUL | script fdをcloseしてシェル異常終了（1） |
| script行長超過 | 残りをLF/EOFまで読み捨てて診断し、次行へ進む |
| script fdのclose失敗 | 正常終了予定の場合は異常終了（1）。先行errorがある場合は先行errorを保持 |
| script内の構文error・コマンド失敗 | 対話モードと同じ規則で診断し、次の行またはコマンド単位へ進む |
| プロンプトの書き込み失敗 | シェル異常終了（1） |
| 起動引数が不正 | 診断してシェル異常終了（1） |

全体の終了コードは `0`（`exit`またはEOFによる正常終了）と `1`（シェル自身の致命的エラー）とする（暫定）。直前の子の終了コードを `exit` の戻り値として採用する機能や `$?` は次ステップ以降の検討対象。

## 7. 入力バッファと資源上限

| 資源 | 初版案 |
|---|---|
| 入力行本文 | 最大126 byte（LFを除く） |
| line storage | 128 byte（本文126 byte、LFなし、終端NUL）。対話モードは`btx::readline()`、script modeはscript fd readerが使用 |
| script fd | script modeにつき1個。shellが所有し、EOF・`exit`・fatal error時にclose |
| 引数数 | 各command stageにつき最大8個 |
| コマンド単位数 | 1入力行あたり最大8個 |
| `argv` 配列 | 各単独コマンドまたはpipeline stageにつき9要素（終端nullptrを含む） |
| redirect operand | 各コマンド単位につき入力1個、出力1個まで。pathnameは入力buffer中を参照 |
| redirect用一時fd | redirect対象ごとに保存用fdとopen元fdを最大各1個 |
| pipeline stage | 各コマンド単位につき最大2個（`|` 1個）、stageごとに最大8引数 |
| pipeline用pipe | pipelineごとに1本 |
| 実行パス用バッファ | 少なくとも 最大入力単語 + 終端NUL を収容できる容量。具体値は実装時固定 |
| 動的メモリ確保 | なし |
| 同時起動する外部コマンド | 単独実行時1個、pipeline時2個 |

`btx::readline()` は本文長を返し、超過行をLF/EOFまで読み捨ててから `too_long` を返す。sh0は `line` のみを解析し、`too_long` の部分内容を実行しない。126 byteちょうど、127 byte以上、LF、CRLF（CRはbufferに残るが`isspace()`で区切られる）、改行なしEOF、超過行の次の呼び出し位置をテストする。埋め込みNULはreadline仕様に従ってerrorとなり、sh0は入力errorとして終了する。

## 8. ilibcおよびOSとの境界

### 8.1 使用する既存ilibc機能

| API | 用途 |
|---|---|
| `std::strcmp()` | `exit` / `cd` / `pwd` / `PID` 組み込み名の判定 |
| `std::isspace()` | 入力行の空白類判定・単語分割 |
| `btx::readline(line, capacity, &length)` | 対話モードでの標準入力取得、TTY判定、TTYでのBS/DEL編集とecho |
| `ilibc::putstr()` またはOSの `write()` | プロンプトの出力 |
| `ilibc::fprint_f<fmt>()` | 診断。`%s`・`%d`で足りる |
| `btx::sys_read(script_fd, &byte, 1)` | スクリプトモードでの1 byte読取 |
| `btx::sys_open()` / `btx::sys_close()` | スクリプトfileのopen・close |

単語分割、`;`によるコマンド単位分割、pipe演算子、リダイレクトを入力buffer上で解析し、最大8単位分の構文情報を固定長で保持する専用parserとして実装する。各単位は最大2 stage、stageごとに最大8引数、入力・出力各1個のpathname参照を持つ。`strtok`や`malloc`を新設しない。行全体の構文検査と全stageの引数数検査を完了するまでファイル・pipe操作を開始しない。

### 8.2 OS側への依存

- プロンプトと診断：OSの標準fd 1/2への書き込み。
- 対話入力：`btx::readline()`。内部で `btx::sys_read(0,...)`、`btx::sys_write(1,...)`、`btx::isatty(0)` を使用する。
- スクリプト入力：`sys_open()`で開いた専用fdから`sys_read()`し、標準入力fd 0は変更しない。EOF/error後に`sys_close()`する。
- 外部コマンド起動・終了待ち：`spawn`、`wait`相当。
- リダイレクト：`btx::sys_open()`、`btx::sys_close()`、`btx::sys_dup()`。`dup2()`は使わない。
- パイプライン：[`pipe_implementation_spec.md`](../kernel/pipe_implementation_spec.md)の`btx::sys_pipe(int32_t* fd_r, int32_t* fd_w)`、[`pipe_io_implementation_spec.md`](../kernel/pipe_io_implementation_spec.md)のread/write契約、`sys_close()`、`sys_dup()`。pipeline stageはspawn時にfd 0/1/2だけを継承し、不要なpipe端点を保持しない。
- `cd`：`btx::sys_chdir(path)`相当の`chdir` syscall。
- `pwd`：`btx::sys_getcwd(buf, capacity)`相当の`getcwd` syscall。
- `PID`：`btx::sys_getpid()` syscall。
- シェルの終了：`main()`のreturnからランタイムが終了システムコールを呼ぶ。
- 対話モードの行入力は `btx::readline()` が `btx::sys_read()` / `btx::sys_write()` / `btx::isatty()` を使う。スクリプトモードはTTY APIを使わない。`<cstdio>` は対話プロンプトと診断出力に用い、BaTTaX用ilibc backendを選択する。

## 9. 非対応機能（次ステップ以降）

- `-c`によるコマンド文字列実行、script positional parameters、環境変数展開。
- リダイレクト`>>`、`2>`、fd番号指定、3段以上のpipeline、バックグラウンド `&`。
- 論理演算 `&&` / `||`。
- クォート、エスケープ、環境変数、PATH探索、ワイルドカード展開。
- `if`、`for`、`while`、関数、コマンド置換。
- `export`、ジョブ制御、シグナル処理、履歴、補完、行編集。

## 10. 最小受入テスト

| 入力・状況 | 期待結果 |
|---|---|
| `sh0`（引数なし） | 対話モードで起動し、従来どおりfd 0から読み、プロンプトを表示する |
| `sh0 /scripts/test.sh` | scriptを逐次実行し、プロンプトと入力echoを出さず、EOF後にstatus 0で終了 |
| scriptの先頭行が`#!/bin/sh0` | `#`コメント行として読み飛ばし、次行からコマンドを実行 |
| scriptの最終行にLFがない | 最終行を1回実行してから終了 |
| script中に構文errorまたはコマンド起動失敗 | pathnameと行番号を含む診断を可能な範囲で出し、次の行・command unitへ進む |
| script行が126 byte | 通常の入力行として解析する |
| script行が127 byte以上 | 残りをLF/EOFまで読み捨て、その行を実行せず次行へ進む |
| scriptにNULまたは読取error | scriptを中止し、script fdをcloseしてstatus 1で終了 |
| script pathnameが存在しない | 診断してstatus 1で終了し、対話loopへ入らない |
| script中に`exit` | 残りのscriptを実行せず、script fdをcloseしてstatus 0で終了 |
| script modeで外部コマンドを起動 | fd 0/1/2（redirect後の状態を含む）のみ継承し、script fdをchildに継承しない |
| script中のコマンドが標準入力を読む | fd 0はscript readerと別なので、継承元の標準入力から読む |
| script fdのclose failure | 正常終了予定をstatus 1とし、先行fatal errorがあれば先行errorを維持 |
| `sh0 /scripts/test.sh extra`、`sh0 -c echo` | usage errorとしてstatus 1で終了 |
| 空行・空白行 | 起動なし。対話モードは次のプロンプト、スクリプトモードは次の行へ |
| `echo hello` | `echo`起動、`hello`を渡して終了待ち |
| `/bin/echo hello` | 現段階では内蔵バイナリ名として解決できず起動失敗。将来は指定フルパスで起動 |
| `echo "a b"` | 3単語に分割。引用符を保持 |
| `echo a;echo b` | `echo a`の終了後に`echo b`を実行する |
| `cd /tmp;pwd` | `cd`成功後のcwdで`pwd`を実行し、`/tmp`を表示する |
| `false;echo next` | 最初の外部コマンドの失敗後も`echo next`を実行する |
| `exit;echo unreachable` | sh0を終了し、後続コマンドを実行しない |
| `;echo a`、`echo a;`、`echo a;;echo b` | 行全体を構文エラーとして拒否し、何も実行しない |
| 1行に9個以上のコマンド単位 | 行全体を上限超過として拒否し、何も実行しない |
| `echo a;b >out`の後半が構文不正 | 行全体を拒否し、先行コマンドも出力ファイルも処理しない |
| `echo a | cat` | 2 stageをspawnしてpipe経由でデータを渡し、両childの終了後に次へ進む |
| `sort <in | head >out` | 左stageのstdinと右stageのstdoutをredirectし、両stage完了後に継続 |
| `echo hello | wc` | 2 stageをspawnし、標準出力から標準入力へpipeで渡して`wc`がEOFまで集計 |
| `cat|wc` | 空白なしのpipe演算子として解析・実行 |
| `| cat`、`cat |`、`cat || wc`、3段pipeline | 構文エラー。pipeやchildを生成しない |
| `exit | cat`、`pwd | cat` | pipeline builtinとして拒否し、組み込み処理を行わない |
| `cat<in` | `<`を認識し、`in`をfd 0へ接続して起動 |
| `echo hi >out` | `out`をcreate/truncateし、子のfd 1へ接続 |
| `cat <in >out` | 両fdを付け替え、spawn後に親fd 0/1を復元 |
| 組み込み`pwd >out` | リダイレクト中に`pwd`を実行し、完了後に親fd 1を復元 |
| `echo hi >>out`、重複`<`または`>`、operandなし | 構文エラー。ファイルopen・truncateなし |
| 出力open後に後続setupまたはspawn失敗 | コマンドを実行せずfdを復元。作成/truncateは維持 |
| redirect用fd枯渇またはdup失敗 | コマンドを実行せず、標準fdを復元して診断 |
| `unknowncmd` | 起動失敗を表示して継続 |
| `cd /tmp` | `chdir()`を呼び、成功後の相対path操作にcwdが反映される |
| `cd`、`cd a b` | 診断し、`chdir()`を呼ばず継続 |
| `PID` | `sh0`自身のPIDを`PID: <10進PID>`形式で表示し、spawnしない |
| `PID extra` | usageを表示し、PIDを取得せず継続 |
| `PID >pid.txt` | PID行をファイルへ出力し、親fd 1を復元 |
| `pid` | `PID`組み込みとは認識せず、外部コマンドとして解決 |
| 通常fileや不存在pathへの`cd` | 失敗を診断し、shellを継続 |
| `cd`成功後に外部コマンドを起動 | 子Processが更新後のcwdを継承 |
| `pwd`をroot cwdで実行 | `/`とLFを出力し、cwd不変 |
| `cd /tmp`成功後に`pwd` | `/tmp`とLFを出力 |
| 相対`cd`のpathに`.`、`..`、連続slashを含む | syscall成功後、実cwdから再構成した絶対pathnameを`pwd`で出力 |
| cwd pathnameが出力buffer容量を超える | `pwd`が診断して継続 |
| `pwd extra` | 診断し、`getcwd()`を呼ばず継続 |
| 削除済みcwdで`pwd` | pathname取得失敗を診断して継続 |
| `pwd`出力失敗 | 診断して継続し、Process cwd不変 |
| 9引数のコマンド | 起動せずエラー、継続 |
| 126文字ちょうどの本文とLF | 有効な行として処理可能 |
| 127文字以上の本文 | 行全体を破棄して継続 |
| CRLF | CRを行バッファに残し、`isspace()`による単語分割で空白として扱う |
| `exit` | シェルが0で終了 |
| 対話入力先頭でEOF | シェルが0で終了 |
| 対話入力エラー、またはscript入力error | シェルが1で終了し、script modeならscript fdをclose |

## 11. 正式実装前に確定する事項

1. 入力最大126文字、最大8コマンド単位、各stage最大8引数、プロンプト`$ `の採否。
2. `exit`の引数と終了コード、EOF直前の不完全行の処理。
3. エラー時の継続・終了規則、入力のコメント行取り扱い。
4. 内蔵バイナリ名と将来のフルパスの対応・解決方法。OS側の段階別spawn仕様と整合させる。
