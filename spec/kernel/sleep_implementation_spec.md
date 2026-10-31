# `sleep()` システムコール実装仕様

## 1. 目的と範囲

本書は、呼び出し元Processを指定時間だけ実行対象から外す `sleep()` の契約を定める。時間単位は概要仕様のHAL tick（1 tick = 1 ms）とし、実際の待機・起床・実行権移譲はSchedulerとProcess管理が担う。

## 2. syscall ABI

```cpp
int32_t sys_sleep(uint32_t ticks);
```

- syscall番号は **35** とする。番号35はV6由来だが、本OSのsyscall ABIとして採用する。
- `ticks` は待機時間をHAL tick単位で指定する。
- `ticks == 0` は時間待ちをせず、協調スケジューラへのyieldを要求する。
- `ticks > 0` は少なくとも指定期間だけ呼び出し元を実行可能状態から外す。
- 成功時は `0`、失敗時は `-1` を返す。現行ABIにerrno設定は追加しない。

## 3. 責務と処理

```text
btx::sys_sleep(ticks)
  -> syscall dispatcher
  -> kernel::sys_sleep(ticks)
  -> Scheduler / ProcessManager sleep-or-yield API
       -> HAL::get_tick_count()
```

Handlerは引数検証と内部API呼び出しに限り、Process状態やwake deadlineを直接変更しない。待機状態への遷移、実行権移譲、起床判定はScheduler/Process管理に置く。HALは時刻源を提供し、Processやsyscallの概念を持たない。

正のtick数について、待機開始時に `HAL::get_tick_count()` を一度読み、期限を `start + ticks` の単純加算で保持しない。`uint32_t` tickの循環を考慮し、起床時に `uint32_t(now - start) >= ticks` で期限到来を判定する。待機時間はtick分解能の範囲であり、起床が指定時刻より遅れることは許容するが、早く起床してはならない。

## 4. 実行段階ごとの動作

### 4.1 初期同期・`NO_CONTEXT`互換段階

[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md) が定める初期段階では独立スタック、Schedulerによるdispatch、コンテキストスイッチを実装しない。この段階では `sleep()` による子Processへの制御移譲および時間待ちは未対応であり、`sys_sleep()` は `-1` を返し状態を変更しない。起動コンテキストや実行Processのない文脈からも成功扱いにしない。

### 4.2 ホスト`HOST_UCONTEXT`

- `ticks == 0` は現在Processを `RUNNING -> READY` とし、[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)に従ってSchedulerへ実行権を渡す。
- `ticks > 0` は現在Processを `WAITING` / `WaitReason::SLEEP` とし、開始tickと待機tick数を関連付けてSchedulerへ実行権を渡す。
- 全ProcessにREADYがなくsleep待ちが残る場合、Schedulerは`HAL::wait_for_tick_change(observed_tick)`でblocking待機する。HAL契約は[`host_hal_tick_wait_spec.md`](host_hal_tick_wait_spec.md)に従う。
- 他Processを実行している間に期限を過ぎたProcessは、そのProcessからSchedulerへ制御が戻った後の期限確認で`READY`へ移す。協調実行のため、実行権を自発的に返さないProcessをsleep期限が割り込んではならない。
- 起床判定には`uint32_t(now - start) >= ticks`を用い、指定tick数より早く呼出し元を再開しない。起床の遅れは許容する。

### 4.3 `NO_CONTEXT`・未対応Context backend

- `NO_CONTEXT`では`sys_sleep()`は引数値によらず`-1`を返し、Process状態を変更しない。
- 将来のContext backendでHALにtick待機相当の非busy-wait機構がない場合も、正数`ticks`を`WAITING`に登録せず`-1`を返す。

### 4.4 タイマ割込みbackend

MCUのタイマ割込みによるScheduler起床を導入する場合も、待機期限の意味とtick単位は変わらない。割込み文脈からProcess選択やContext切替を直接行わない。割込みの登録・通知、割込み文脈で許される処理、idle中のCPU待機方法は対象HAL仕様で定める。

## 5. エラーと状態保証

- Processが存在しない、Schedulerが利用できない、または待機状態を安全に登録できない場合は `-1` を返し、Process状態を変更しない。
- `ticks == 0` のyield失敗も `-1` とする。
- 待機登録に失敗した場合、Processを `WAITING` のまま残さない。
- sleep Processが`WAITING`へ遷移した後にHAL tick待機が回復不能エラーを返した場合、当該syscallから`-1`を返すのではなくScheduler fatal errorとして扱う。WAITING Processを起床させて成功扱いにしない。
- 完了後は呼び出し元Processを再び実行中として扱い、成功なら `0` を返す。
- シグナルによる待機中断や残り時間の返却は対象外とする。

## 6. 受け入れ確認項目

- `HOST_UCONTEXT`の協調Schedulerで`ticks == 0`はyieldし、時間待ちの期限を作らない。
- 初期同期段階では `sleep()` が `-1` を返し、Process状態と実行主体を変えない。
- `HOST_UCONTEXT`では正数sleepが`WaitReason::SLEEP`で待機し、期限より前に起床しない。
- `NO_CONTEXT`およびHAL tick待機機構のないbackendではsleepが`-1`を返し、状態を変更しない。
- tick値の `uint32_t` wraparoundをまたぐ待機を正しく判定する。
- 待機登録失敗時にProcessがWAITINGへ取り残されない。
- 待機可能Processがない場合にbusy-waitしない。
