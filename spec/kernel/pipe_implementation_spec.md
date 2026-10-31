# `pipe()` システムコール実装仕様

## 1. 目的と適用範囲

本書は、現在Processに読み出し用・書き込み用のfdを一対作成する`pipe()` syscallのABI、生成処理、FD登録、失敗時rollbackを定める。syscall番号42を有効化する。

本書が定めるのはpipe endpoint pairの生成と通常のFD機構への登録である。有限バッファの読書き、EOF、reader/writer消滅時の動作、Schedulerのblock/wakeupは[`pipe_io_implementation_spec.md`](pipe_io_implementation_spec.md)で定める。したがって本書だけではsh0のpipelineが実行可能になったことを意味しない。

## 2. syscall ABI

```cpp
int32_t sys_pipe(int32_t* fd_r, int32_t* fd_w);
```

- syscall番号はV7由来の**42**とする。
- `fd_r`と`fd_w`は、それぞれ読み出しfd・書き込みfdを格納する出力先を指す。成功時、カーネルはFdTableの空きfdを2つ割り当て、それぞれの番号を出力先へ書き込む。呼出し側は両pointerが有効で、書込み可能かつ互いに重ならない領域を指す責任を持つ。
- 成功時は`0`、失敗時は`-1`を返す。errnoや失敗理由別の戻り値は追加しない。
- syscall境界で出力pointerの有効性、alignment、書込み可能性、重複を検査しない。ユーザープログラムが有効な出力先を渡す。無効pointerへのアクセスで発生するfaultやクラッシュは許容する。
- `*fd_r`と`*fd_w`の値は戻り値が`0`の場合だけ有効とする。失敗時に出力領域が部分的に更新されても、そこに残る数値をfdとして使ってはならない。
- `pipe2()`、flags引数、指定fdへの割当、close-on-spawn属性は設けない。

ユーザー側wrapper:

```cpp
int32_t btx::sys_pipe(int32_t* fd_r, int32_t* fd_w); // system_call(42, fd_r, fd_w)
```

カーネル側handler:

```cpp
int32_t kernel::sys_pipe(int32_t* user_fd_r, int32_t* user_fd_w);
```

## 3. 責務と内部API

```text
btx::sys_pipe(fd_r, fd_w)
  -> syscall dispatcher (42)
  -> kernel::sys_pipe(user_fd_r, user_fd_w) // 内部API呼出し・出力値書込み・結果変換
  -> kernel::kernel_pipe(Process&, PipeFdPair&)
       -> Pipe管理領域からpipe state/endpointsを確保
       -> FileDescriptionをread/write用に各1個確保
       -> 現ProcessのFdTableの最小空きfd 2つへ登録
```

概念上の内部型/API:

```cpp
bool kernel_pipe(Process& process, PipeFdPair& out);
```

- 内部APIはProcess/FdTableとPipe資源を直接管理し、2つの最小空きfdへendpointを登録する。
- `kernel::sys_pipe()`は内部API呼出し、出力値書込み、結果変換に限る。pipe state、FileDescription、FdTableをhandlerから直接操作しない。
- 内部失敗の詳細型は既存のFD/IPC実装規約に合わせる。新しいrepository-wide error enumやerrnoを導入しない。

## 4. 成功時の意味

成功時、呼出し元ProcessのFdTableに読み出し端と書き込み端が1つずつ登録される。

| 出力 | 意味 | FileDescriptionのaccess mode |
|---|---|---|
| `*fd_r` | pipeから読み出すendpointのfd番号 | `O_RDONLY` |
| `*fd_w` | pipeへ書き込むendpointのfd番号 | `O_WRONLY` |

- 空きfd番号は通常のFdTable採番規則に従い、生成開始時点で空いている最小の2番号を昇順で割り当てる。
- 2個の空きfdがない場合は失敗し、片側だけ登録して成功させない。
- 各endpointは別個の`FileDescription`を持ち、対応するread-only/write-only access modeを設定する。両endpointは同じpipe stateを共有する。
- 新規FileDescriptionは通常の`alloc_file_description()`で各1参照として作り、対応するfdを登録する。FileDescriptionのrefcountやFdTable内部配列を直接操作しない。
- pipeはseek不能である。`lseek()`は既存のseek不能target規則により失敗する。
- `dup()`および`spawn()`で複製・継承されたfdは、通常のFileDescription共有規則に従う。`pipe()`はspawn継承対象を特別扱いしない。
- `pipe()`の成功はfd pairが作成されたことを表す。読書きのblock/EOF等の詳細動作は[`pipe_io_implementation_spec.md`](pipe_io_implementation_spec.md)に従う。

## 5. 生成とcommitの順序

syscall handlerと内部APIは、観測可能な成功状態が両fd同時に成立するよう処理する。

1. `kernel_pipe()`内で現Processを取得できること、およびFdTableに空きfdが2つあることを確認する。検証前にPipe資源を確保しない。
2. 固定長Pipe管理領域からpipe stateとread/write endpointを確保する。動的heap allocationを使わない。
3. read endpointとwrite endpointをそれぞれ`alloc_file_description(endpoint, O_RDONLY/O_WRONLY)`相当でFileDescription化する。
4. 選んだ2 fd番号へread description、write descriptionを登録する。2つ目の登録に失敗した場合は1つ目を`detach()`し、両FileDescriptionをreleaseしてpipe stateを再利用可能にする。
5. 両fdの登録成功後、内部APIはfd pairを`out`へ返す。
6. syscall handlerはread fd番号を`*user_fd_r`へ、write fd番号を`*user_fd_w`へ直接書き込む。成功時に0を返す。これらのstoreでfaultが起きる場合はクラッシュを許容し、copyout失敗の検出やrollbackは行わない。

FdTableとPipe管理領域の状態遷移は、2つのfdが両方commitされるか、呼出し前の状態へrollbackされるかのどちらかとする。片側endpointだけをユーザーから利用可能な状態で残してはならない。

## 6. 失敗時とrollback

| 失敗条件 | 結果と状態保証 |
|---|---|
| current ProcessまたはFdTableを取得できない | `-1`。状態不変 |
| 空きfdが2個未満 | `-1`。Pipe/FileDescriptionを確保せず、FdTable不変 |
| Pipe管理領域枯渇 | `-1`。FdTable不変 |
| 片方または両方のFileDescription確保失敗 | 確保済みのdescriptionとpipe stateを解放し、`-1` |
| 2つ目のfd登録失敗 | 登録済みfdをdetachし、対応descriptionとpipe stateをreleaseして`-1` |

- 失敗時にFdTableへ新しいfdを残さず、呼出し前からあるfd・FileDescriptionの参照数・pipe内容を変更しない。
- rollback時にcleanupも失敗した場合は最初の生成失敗をsyscall結果として保つ。fdはdetachして再利用不能な状態にし、dangling参照を公開しない。cleanup不能なPipe資源は安全性を優先して固定領域内に保持し得るが、ユーザーへfdを返さない。
- 失敗時はfd番号を有効なものとして報告しない。既存のFdTable entryは変更しない。

## 7. Pipe/FileDescriptionの所有権

- pipe stateは有限のカーネル管理領域が所有する。`FileDescription`はread/write endpointを`FileInterface`として参照し、FdTableはFileDescriptionへの対応だけを保持する。
- read/write endpointの実体表現（1つのPipeに2 endpointを持たせるか、endpointごとのFileInterfaceにするか）は実装可能な最小方式を採用し、Processやsyscall層へ内部レイアウトを公開しない。
- 初回作成時はread endpointとwrite endpointが各1つ存在する。dup/spawnされた参照は既存のFileDescription refcountをretainし、endpointの有効参照として数える。
- 各FileDescriptionの最終releaseで対応するendpoint参照を1つ解放する。pipe stateは両endpointの最終参照がなく、IPC仕様上の待機者・操作参照もなくなった後に限り再利用する。具体的なreader/writerカウントと待機者寿命はIPC pipe仕様で定める。
- `close()`、`dup()`、`spawn()`の参照処理をpipe専用経路で重複実装しない。

## 8. syscall境界・エラー

- syscall番号42のdispatcherは2引数を`int32_t*`として取得し、`kernel::sys_pipe(user_fd_r, user_fd_w)`へ渡す。
- `kernel::sys_pipe()`は内部API呼出し、両出力先への直接書込み、成功0/失敗-1への変換を行う。
- syscall層はFileInterface種別判定、Pipe管理table操作、FileDescription参照数更新、FdTable内部配列への直接アクセスを行わない。
- ユーザーポインタの検証・fault recoveryは行わない。呼出し側が有効な出力先を渡す。不正pointerに起因するfaultやクラッシュを許容する。
- errnoやホストglobal `errno`をkernel内部状態として使わない。

## 9. 並行性と実行モード

- pipe pair生成中は、2 fdの空き確認からfd登録完了またはrollbackまでを1つの不可分な状態遷移として扱う。
- 現行の協調実行では、生成経路が自発的にyield/blockしない限り同一ProcessのFdTableを他Processが同時変更しない。将来プリエンプション/複数CPUを導入する場合は、Process FD管理とPipe管理で定める同期規則の中でpair確保・登録・rollbackを保護する。
- `NO_CONTEXT`を含め、ProcessとFdTableが利用可能ならpipe pairの作成自体はContext切替を必要としない。pipe read/writeがblockする動作は、Scheduler対応を定めるIPC pipe仕様に従う。
- pipe作成中にblocking、sleep、busy-waitを行わない。

## 10. 本書の対象外

- pipe bufferの容量・格納方式、read/writeの部分転送・blocking・EOF・writer不在時の挙動。
- reader不在時のwrite、シグナル、broken-pipe動作。
- Schedulerの`WAITING(IO)`状態、pipe read/writeのblock/wakeup、idleとの連携。
- spawn file actionsによるfdのclose・再配置。pipe端点の継承対象選択は`spawn_wait_implimentation_spec.md`のFDリストで行う。
- `pipe2()`、非blocking mode、atomic write size、複数段pipeline。

これらはIPC pipe I/OとSchedulerの別仕様で定める。`spawn()`のFD継承対象を選ぶ方法は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。本書だけでsh0 pipelineを有効化するものではない。

## 11. 確認項目

1. syscall番号42と2つの出力pointerを持つwrapper、dispatcher、kernel handlerの型が一致する。
2. 成功時`*fd_r`はread専用、`*fd_w`はwrite専用の異なるfdを示す。
3. 呼出し側が有効な2つの出力先を渡す前提でfd番号を書き戻す。不正pointer faultの回復やrollbackは行わない。
4. 空きfdが1つ以下の場合は失敗し、pipe、FileDescription、FdTableの状態を変更しない。
5. pipe、FileDescription、FdTableの各資源枯渇箇所でrollbackされ、片側fdやdangling endpointを残さない。
6. 作成したread/write fdは`dup()`および`spawn()`で既存の共有参照規則に従い、最後のendpoint参照release後にpipe stateが再利用可能となる。
7. pipe fdへの`lseek()`は失敗し、通常ファイルのfd positionや内容に影響しない。
8. Processがない文脈、fd table未初期化、Pipe pool枯渇で成功fdを返さない。

## 12. 関連仕様

- syscall番号、wrapper、dispatcher: [`syscall_entry_spec.md`](syscall_entry_spec.md)、[`syscall_generator_spec.md`](syscall_generator_spec.md)
- FileDescription、FdTable、FileInterface、標準FD: [`open_file_description_spec.md`](open_file_description_spec.md)
- `open()` / `close()`とFileDescriptionの最終release: [`open_close_implementation_spec.md`](open_close_implementation_spec.md)
- `dup()`: [`dup_implementation_spec.md`](dup_implementation_spec.md)
- spawn時のFD継承: [`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)
- pipeによるshell pipelineの利用条件: [`../command/command_sh0_spec.md`](../command/command_sh0_spec.md)
