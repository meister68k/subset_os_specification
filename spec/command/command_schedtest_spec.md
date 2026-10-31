# 協調Scheduler試験用 `schedtest` コマンド仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | BTX-SCHEDTEST-001 |
| 版 | 0.1（実装前） |
| 対象 | ホスト・マイコン共通の内蔵ユーザーバイナリ `schedtest` |
| 位置づけ | 複数Process間の協調yieldと再開を目視確認する開発・デバッグ用コマンド |
| 依拠する仕様 | [`cooperative_scheduler_spec.md`](../kernel/cooperative_scheduler_spec.md)、[`sleep_implementation_spec.md`](../kernel/sleep_implementation_spec.md)、[`spawn_wait_implimentation_spec.md`](../kernel/spawn_wait_implimentation_spec.md)、[`command_ps_debug_spec.md`](command_ps_debug_spec.md)、[`syscall_entry_spec.md`](../kernel/syscall_entry_spec.md) |

## 1. 目的と範囲

`schedtest`自身が2つのworker Processをspawnし、各workerが複数回`sys_sleep(0)`を呼び出す。workerが出力する進捗行を観察して、協調SchedulerがProcessを切り替え、yieldしたworkerを後から再開することを確認する。さらに試験中に`ps`を起動し、Process名`schedtest-a`および`schedtest-b`がProcess一覧に表示されることを確認する。

`sh0`は通常1つの外部コマンドを終了まで待つため、sh0から逐次起動するだけでは複数の子Processを同時にREADYにできない。本コマンドが自分で複数workerを生成することで、この制約下でも切替を確認できるようにする。

本コマンドが確認するのは明示的なyieldによる協調切替である。正数sleepのtimer起床、プリエンプション、複数CPU実行は対象外とする。Processごとのstack境界やContext情報の確認は[`command_ps_debug_spec.md`](command_ps_debug_spec.md)の責務とする。

## 2. 起動形式とworkerモード

```text
schedtest
```

- 通常起動では引数なしとする。余分な引数はusage errorとし、workerを生成しない。
- 内蔵バイナリ名`"schedtest"`を通常起動名とし、`"schedtest-a"`および`"schedtest-b"`をworker用aliasとして、すべて同じ入口関数へ対応付ける。すべてのターゲットで各名前を`kernel::user_binary()`から検索できるようにする。aliasはProcess名を区別するために用いる。
- 将来ファイルシステム上のpathnameを導入する場合のpathnameは`/bin/schedtest`とする。
- 同じ入口関数をworkerモードにも使う。worker起動は`argv[1]`の予約値`--worker`で識別し、`argv[2]`のworker ID（`A`または`B`）と`argv[3]`の反復数を検証する。
- `schedtest`自身がworker用argvを固定領域で構成し、`spawn("schedtest-a", worker_argv_a, nullptr, 0)`および`spawn("schedtest-b", worker_argv_b, nullptr, 0)`相当でworkerを生成する。Process名はspawnのname引数から記録されるため、`ps`ではそれぞれ`schedtest-a`、`schedtest-b`と表示される。動的メモリは使わない。
- worker以外に、内蔵バイナリ`ps`を`spawn("ps", ps_argv, nullptr, 0)`相当で診断用子Processとして起動する。`ps`はworkerを両方生成した後、親がwaitする前に生成する。
- workerと`ps`は標準出力・標準エラーを使うため、全open fdの継承を明示的に要求する。`nullptr, 0`は全継承を表す。
- workerモードをsh0から直接起動する用途は定義しない。workerの引数不正は診断して失敗statusで終了する。

## 3. 固定試験条件

- worker数は2、worker IDは`A`と`B`とする。
- 各workerの反復数は3とする。値は本仕様の固定値であり、コマンドラインから変更できない。
- Process名確認用の`ps`子Processを1つ生成し、親がwaitを始める前にworker 2つとpsをすべてREADYにする。
- 各workerは各反復で進捗行を1行標準出力へ書き、その後、最終反復でなければ`btx::sys_sleep(0)`を1回呼ぶ。
- 進捗行は`worker A pid=2 round=1/3\n`相当の形式とし、worker ID、実際のPID、1始まりの反復番号を含める。PIDの桁数は固定せず、符号なし10進表記で出力する。
- 各行は1回の`sys_write()`要求で出力する。短い正の部分書込みがあれば残りを続けて書き、0・負値・要求を超える戻り値は出力失敗とする。
- workerの標準出力には試験行を出す。子`ps`の標準出力にはProcess一覧を出す。試験全体の診断は標準エラーfd=2に出す。

workerが進捗行を出力してからyieldするため、`HOST_UCONTEXT`の通常条件ではA/Bの行が交互に現れる。最初のworkerはSchedulerの現在cursorに従うため、Aから始まるかBから始まるかは規定しない。`ps`は標準出力へProcess一覧を出すため、その一覧がworkerの進捗行の間に現れる。行の間に他のProcessの出力が割り込む環境では、全出力が厳密な交互表示になることまでは保証しない。

## 4. 親Processの処理

1. `argc == 1`を検証し、標準出力・標準エラーが利用可能であることを前提に試験を開始する。
2. worker A、worker Bの順で、3反復のworker引数を持つProcessをspawnする。
3. worker両方のspawn成功後、`ps`をspawnする。これにより`ps`実行中に両workerのProcess名がProcess tableに存在する。
4. 3つのspawnが成功した後、`sys_wait()`を3回成功するまで呼び出し、終了した子を回収する。waitが負値を返した場合は未回収の子を残した状態で同じwaitを再試行し、成功前に次のプロンプトへ戻らない。
5. 各wait成功時に返されたPIDが、未回収のworker A、worker B、またはpsのPIDと一致することを検証する。workerおよびpsのstatusが0でない子があれば試験失敗として記録する。
6. 3つの子を回収後、両workerのyieldが成功し、psが正常終了していれば終了status 0、その他は1で終了する。

`spawn()`は子をREADYにするだけであり、親はworker 2つとpsをspawnしてからwaitを始める。`HOST_UCONTEXT`では親がwaitでblockした後にSchedulerが3つの子をdispatchする。psの実行時には、Process tableに親`schedtest`とworker`schedtest-a`、`schedtest-b`が載り、各Process名を確認できる。子終了時は既存のwait/syscall仕様に従って親が起床し、終了statusを回収する。

### 4.1 spawn途中の失敗

- いずれかのspawnが失敗した場合、診断し、すでに生成したすべての子（workerまたはps）をwaitして回収してからstatus 1で終了する。
- 失敗回収中の`wait()`負値は同じ子に対して再試行する。回収できていない子を残したまま正常終了・再プロンプト復帰してはならない。
- 生成失敗時に追加Processをspawnしたり、別の待機方式へfallbackしたりしない。

## 5. workerの処理

worker入口は以下を実行する。

1. `argc == 4`、`argv[1] == "--worker"`、IDが`A`または`B`、`argv[3] == "3"`であることを検証する。
2. `btx::sys_getpid()`で自身のPIDを得る。取得失敗なら診断可能な範囲でfd=2へ出してstatus 1で終了する。
3. 反復番号1から3まで、PIDと反復番号を含む1行をfd=1へ出力する。
4. 反復1および2の出力後に`btx::sys_sleep(0)`を呼ぶ。戻り値0なら次の反復へ進み、失敗ならfd=2へ診断してstatus 1で終了する。
5. 3行を出力し、2回のyieldが成功したらstatus 0で終了する。

workerは正数sleep、busy-wait、HAL tickの直接参照、Scheduler内部APIを使用しない。workerはFDを開閉せず、親から継承した標準FDを使う。

## 6. 実行モードと結果の解釈

- `HOST_UCONTEXT`で協調Schedulerが有効な場合、workerは独立Contextで実行される。進捗行がworker間で交互に現れ、各workerが3行出した後status 0で回収されれば、複数READY Process間のyield・再開の確認となる。
- 同じ試験中にspawnした`ps`はProcess名`schedtest-a`、`schedtest-b`を含むProcess一覧をfd=1へ出力する。`ps`が実行される前に両workerが終了しないよう、3つの子を同時にREADYにしてからwaitを開始する。Schedulerのラウンドロビンにより、両workerが少なくとも最初のyieldまで進んだ後もpsは選択可能であり、両名を一覧で観察できる。
- `NO_CONTEXT`では`sys_sleep(0)`は未対応として失敗する。workerは失敗診断と非0 statusで終了し、親は子を回収して`schedtest`を失敗終了する。これをScheduler成功と誤認してはならない。
- 他のContext backendでは、`sys_sleep(0)`とSchedulerが本仕様と同等の協調切替を実装している場合に限り、同じ期待結果を適用する。
- `schedtest`はContext/stack境界の値を独自に取得・検証しない。起動した`ps`は既存の`command_ps_debug_spec.md`に従ってProcess情報と境界情報を表示する。

## 7. 入出力・終了status

| 状況 | 動作 |
|---|---|
| 両workerが合計4回のyieldを完了し、psが正常終了 | 無診断でstatus 0 |
| 引数不正 | usageをfd=2へ出し、worker・psを生成せずstatus 1 |
| workerまたはpsのspawn失敗 | fd=2へ診断し、既生成子Processを回収してstatus 1 |
| `sys_sleep(0)`失敗またはworker出力失敗 | workerが非0 statusで終了し、親は全子Process回収後status 1 |
| wait成功値が既知の子PIDと一致しない | 診断してstatus 1。未回収の子があればwaitを続行 |
| wait syscall失敗 | 同じ子のwaitを再試行し、成功するまで親は終了しない |
| 親の診断出力失敗 | 可能な範囲で子を回収してstatus 1 |

診断例：`schedtest: usage: schedtest\n`、`schedtest: spawn failed\n`、`schedtest: worker yield failed\n`、`schedtest: worker output failed\n`。文言は同等であればよい。診断writeは部分書込みを処理し、0または負値で停止する。診断失敗を再帰的に報告しない。

## 8. 使用API・実装制約

| API | 用途 |
|---|---|
| `btx::sys_spawn()` | 固定workerと`ps`を生成し、`nullptr, 0`で全FDを継承 |
| `btx::sys_wait()` | worker終了statusを回収 |
| `btx::sys_sleep(0)` | workerの協調yield |
| `btx::sys_getpid()` | worker識別用PID取得 |
| `btx::sys_write()` | 進捗行・診断の出力 |
| 内蔵バイナリ`ps` | Process名を表示する診断用子Process |

- 動的メモリ確保、host OS固有のthread/process API、`printf()`、カーネル内部Scheduler APIは使用しない。
- worker数と反復数は固定・有限とし、無制限ループやユーザー指定の資源消費を設けない。
- 文字列・数値出力は固定長bufferで構成し、境界確認後にwriteする。
- Process tableやContextを直接参照・変更しない。通常のsyscallとProcess入口関数を使用し、Process table情報は子`ps`を通して表示する。
- コマンド自身のFD table、cwd、親子関係以外のProcess状態を変更しない。

## 9. 受入確認項目

1. 登録名`schedtest`で通常起動し、余分な引数はworker・psをspawnせず拒否する。
2. 親がworker A/Bおよびpsをspawnしてからwaitし、3子を各一度回収する。
3. `ps`出力に親Process名`schedtest`とworker名`schedtest-a`、`schedtest-b`が含まれる。
4. `HOST_UCONTEXT`でworker A/Bの進捗が各3行出て、各workerが出力後に2回`sys_sleep(0)`を呼び、全行を出し終える。
5. Schedulerのラウンドロビン選択に従い、他のREADY Processがない通常条件でA/Bの進捗が交互に観察される。初回worker IDは固定しない。
6. 各workerのPIDがそれぞれ異なり、waitで返るPIDと終了statusが親の検証に使われる。psもstatus 0で終了する。
7. workerのyield失敗・出力失敗またはpsの失敗は非0 statusとなり、親が全子Processを回収して試験失敗を返す。
8. いずれかのspawn失敗時に、既生成子Processを回収し、子を放置しない。
9. `NO_CONTEXT`ではsleep(0)失敗を含むworker失敗として完了し、Scheduler成功と誤認しない。
10. 部分write、0/負値、要求を超えるwrite戻り値を安全に扱い、無限再試行しない。
11. 試験全体が固定2 worker・各3反復で終了し、timer待機やbusy-waitを行わない。

## 10. 対象外

- 任意のworker数・反復数の指定、ベンチマーク、CPU時間測定。
- 正数`sleep()`によるtimer起床試験、プリエンプション、優先度、複数CPU。
- `schedtest`自身によるContext/stack境界の取得・検証、Process table走査、`ps`出力形式の変更。
- `NO_CONTEXT`でのyield emulation、syscall未対応時の代替動作。
- シグナルによるworker中断や終了。
