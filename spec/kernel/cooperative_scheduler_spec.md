# 協調Scheduler仕様書

## 1. 目的と適用範囲

本書は、単一CPU上でProcessを協調的に切り替えるSchedulerの動作を定める。Processの生成・状態・親子関係・終了資源の所有は[`process_management_spec.md`](process_management_spec.md)、実行Contextとスタックは[`process_context_implementation_spec.md`](process_context_implementation_spec.md)、syscall境界は各syscall仕様に従う。

SchedulerはProcessを選択し、ProcessManagerとContext層を用いて実行を切り替える。ユーザーコードの任意の時点を割込みで中断するプリエンプション、複数CPU、優先度、タイムスライス、シグナルによる中断は対象外とする。Processはyield、wait、sleep等の明示的なブロック操作を行ったときにのみ実行権を渡す。

## 2. 実行モード

| モード | Scheduler動作 |
|---|---|
| `HOST_UCONTEXT` | ProcessごとのContextを使い、本書の協調切替を行う。PID 1の`init`もProcess専用Contextとスタックで実行する |
| `NO_CONTEXT` | 既存のwait駆動互換動作を維持し、本書のScheduler切替を行わない。`sleep()`は未対応として失敗する |
| 将来のマイコンContext backend | 本書と同じProcess状態・選択・待機規則をCPU固有Context実装で実現する |

`NO_CONTEXT`はデバッグ・互換用の旧実行方式であり、Schedulerの機能同等性を保証しない。ターゲットでSchedulerを有効にする場合は、そのターゲットのContext backendを必須とする。

ホストでは`BTX_USE_UCONTEXT=1`を`HOST_UCONTEXT`、`BTX_USE_UCONTEXT=0`を`NO_CONTEXT`として扱う。`HOST_UCONTEXT`を選んだビルドでは協調Schedulerを有効にする。MC68000/Cortex-M0で有効化する際は対応するCPU固有Context backendを必須とし、ホスト`ucontext`依存を含めない。

## 3. Processの実行と切替

### 3.1 起動

カーネル初期化はProcessManagerにPID 1の`init`を登録し、標準FDとProcess専用Contextを準備した後、`READY`としてSchedulerのrun loopを開始する。`init`入口をカーネル起動スタックから直接呼び出してはならない。Schedulerのrun loopはカーネル起動Context上で動作し、ユーザーProcessのContextとの切替先となる。最初のdispatch時に`init`を選択し、`current()`を設定する。

Processが初めて選択されたときは初期Contextから入口関数を実行する。yieldまたはblock後に再選択されたProcessは、Contextを保存した位置から再開する。

### 3.2 Ready選択順

- 実行対象は`ProcessState::READY`のProcessに限る。`CREATING`、`WAITING`、`EXITING`、`ZOMBIE`、`FREE`は選択しない。
- 選択は固定長Process tableを使ったラウンドロビンとする。直前に選択したslotの次から昇順に探索し、末尾の次は先頭へwrapする。
- 起動直後、まだProcessをdispatchしていない場合はslot 0から探索を開始する。
- 1回の探索で各slotを高々1回調べる。slot cursorの加算は上限を検査し、整数wraparoundに依存しない。
- 選択されたProcessを`READY -> RUNNING`へ遷移させ、ProcessManagerの`current()`を更新してからContextへ切り替える。
- ProcessからScheduler Contextへ戻るときは、戻り先のProcessをScheduler内部で特定できる状態に保持した後、ProcessManagerの`current()`を「実行中Processなし」にする。Scheduler Contextでは`current()`を呼び出さず、次のProcessをdispatchする直前に設定する。
- 同一ProcessしかREADYでない場合、そのProcessを再選択してよい。yieldは少なくともSchedulerへ制御を返し、他にREADYがあればそのProcessへ機会を与える。
- 優先度、実行時間、生成順を選択基準に追加しない。wait対象の子を優先する特別規則も設けない。

### 3.3 Yield

実行中Processがyieldすると、状態を`RUNNING -> READY`、待機理由を`NONE`としてContextをSchedulerへ戻す。yieldしたProcessは他のREADY Processをラウンドロビンで選んだ後、再選択されるまで呼出し位置で停止する。

yieldの結果は、Processが後に再開したとき成功として呼出し元へ返る。yield操作そのものが失敗した場合はProcess状態と`current()`を変更しない。

## 4. `spawn()`、`wait()`、終了との連携

- Schedulerモードの`spawn()`はProcessとContextを準備して`READY`として公開し、子をその場で実行しない。親の`current()`は変わらない。
- Schedulerモードの`wait()`は子Processの終了を待つ。終了済みZOMBIEの子があればそれを返す。子が存在しない場合は`ECHILD`相当で失敗する。
- 子が存在するが終了していない場合、親を`RUNNING -> WAITING`、待機理由を`CHILD`としてからSchedulerへ制御を戻す。SchedulerはREADY Process（通常は子を含む）を選ぶ。
- 子が終了してZOMBIEになったとき、親が`WAITING(CHILD)`なら親を`READY`へ戻し、待機理由を`NONE`にする。親が再開してからZOMBIEの子を返す。
- `wait()`のsyscall出力先検証・status書込み・回収確定順は既存のwait仕様を維持する。出力失敗でZOMBIEを回収しない。
- 子の入口returnまたは`exit(status)`は終了経路へ入り、FD等のProcess資源を解放し、実行資源を安全なContextへ戻った後にZOMBIEとして確定する。終了したProcessへ再dispatchしない。
- `WAITING(CHILD)`中に、親が待つ対象となる子を持たず、他に実行可能Processも存在しない状態を作ってはならない。整合しない場合、無限dispatchやbusy-waitをせずScheduler異常として検出可能にする。

従来のwait駆動実行規則はScheduler無効の`NO_CONTEXT`互換モードに限って適用する。`HOST_UCONTEXT`でSchedulerを有効にした場合、本節が`spawn_wait_implimentation_spec.md`のwait駆動実行規則に優先する。

## 5. Blockと起床

- Processが待機する操作はProcessManagerを通じて`RUNNING -> WAITING`へ遷移してからSchedulerへ切り替える。syscall handlerがProcess状態を直接変更しない。
- 待機理由は`WaitReason`に記録する。Schedulerは待機理由ごとの条件が満たされた場合に限り`WAITING -> READY`へ遷移させ、待機理由を`NONE`に戻す。
- Schedulerで実装する待機理由は`CHILD`、`SLEEP`、および[`pipe_io_implementation_spec.md`](pipe_io_implementation_spec.md)で定めるpipeの`IO`待機とする。device I/O待機は対象外。
- Context復帰後、blockしたProcessは元のカーネル呼出し位置から続行する。別ProcessのContextから戻った値や状態を流用しない。

## 6. Tick、`sleep()`、idle

- Schedulerは単一の`uint32_t` HAL tickを時間基準とし、絶対期限の単純加算でwrapを処理しない。sleepの経過判定は`uint32_t(now - start) >= ticks`とする。
- `sleep(0)`はyieldと同じ動作をする。
- 正数の`sleep(ticks)`は`WAITING(SLEEP)`としてSchedulerへ制御を戻し、期限経過前にProcessをREADYへ戻さない。
- `HAL::get_tick_count()`はtick値を読むだけである。正数sleepを有効にするには、tickが非busy-waitで進み、全Process待機中にもSchedulerが期限後に再開できるHAL機能が必要である。
- ホスト`HOST_UCONTEXT`は[`host_hal_tick_wait_spec.md`](host_hal_tick_wait_spec.md)の`HAL::wait_for_tick_change()`を使い、正数sleepに対応する。`NO_CONTEXT`ではSchedulerを動かさず、正数・ゼロsleepとも未対応とする。
- READYがなく、SLEEP待ちProcessがあるとき、Schedulerはbusy-waitせず`HAL::wait_for_tick_change(observed_tick)`を呼ぶ。成功後はtickと全sleep期限を再評価する。待機失敗はScheduler異常としてfatal error経路へ移し、WAITING Processを成功扱いで再開しない。IO待ちも存在するときは、timer満了でSchedulerが再開した後、pipe条件も再評価する。
- READYがなくSLEEP待ちもない状態でpipe IO waiterが残る場合、pipe条件を満たす実行可能ProcessがないためScheduler deadlock/fatal errorとして検出する。READYもWAITINGも存在せず、PID 1も終了済みならrun loopを終了し、他の不整合なWAITING状態ならScheduler異常として扱う。
- タイマ割込みからProcess選択・Context切替を直接行うこと、プリエンプションは本段階で導入しない。割込みはtick源を更新・通知するにとどめ、Schedulerは通常実行文脈で再開する。

## 7. 責務と内部APIの形

以下は責務境界を示す概念APIであり、クラス名・シグネチャの確定ではない。

```cpp
class Scheduler {
public:
    SchedulerResult run();
    ScheduleResult yield_current();
    ScheduleResult block_current(WaitReason reason);
    void make_ready(Process& process);
};
```

- ProcessManagerがProcess状態、親子関係、終了、wait結果、current参照を所有する。
- SchedulerがREADY選択、dispatch順、待機解除の呼出し、Scheduler run loopを所有する。
- Context層がContextの保存・復元だけを行う。状態変更やProcess選択を行わない。
- HALがtick源とtick変化までのblocking待機を提供する。HALはProcess情報・sleep期限・起床条件を参照しない。
- `ScheduleResult`等の内部結果型や失敗表現は実装時に既存のエラー規約と合わせる。syscall ABIに新しい戻り値やerrnoを加えない。

固定長Process table以外にREADY queueやsleep queueを必須としない。初版はtable scanを用いてよく、別のキューを追加する場合も容量は`MAX_PROCESSES`以下とし、Process状態との二重登録を避ける。

## 8. 対象外と拡張境界

- timer割込みによる周期的dispatch、プリエンプション、実時間優先度、マルチCPU、同期ロック。
- device I/Oのblock/wakeup条件。pipeの条件は[`pipe_io_implementation_spec.md`](pipe_io_implementation_spec.md)に従う。
- シグナルによるwait/sleep中断、残り時間の返却。
- `NO_CONTEXT`でのContext切替エミュレーション。

これらを追加する場合もProcess状態の正本はProcessManager、CPU依存切替はContext層、時間源とidleはHALとし、syscall handlerにScheduler policyを複製しない。

## 9. 受け入れ確認項目

1. 初回dispatchでPID 1の`init`がProcess専用Context上で開始する。
2. READY Processをtable slot round-robinで選び、FREE/CREATING/WAITING/EXITING/ZOMBIEを実行しない。
3. yield後に他のREADY Processが実行され、yieldしたProcessが保存位置から再開する。
4. READYが1つのときyieldが安全に復帰し、slot cursor境界でwraparoundする。
5. 親のwaitで親がWAITING(CHILD)となり、子のexit後に親がREADYとなってwait呼出し位置から再開する。
6. status出力失敗時にZOMBIE子を保持し、再度waitして回収できる。
7. Process終了後、Context/stackを実行中に解放せず、終了Processを再dispatchしない。
8. `HOST_UCONTEXT`で`sleep(0)`がyieldし、正数sleepはHAL tickを待って指定期間より早く再開しない。
9. schedulerがREADYなしでbusy-waitせず、WAITING理由と残存Processの整合性を検出する。
10. `NO_CONTEXT`ビルドがSchedulerなしのwait駆動互換動作を維持する。
