# ホストHAL tick・待機仕様書

## 1. 目的と適用範囲

本書は、Linuxホスト環境でHALの単調tickを取得し、協調Schedulerが実行可能Processのない間にbusy-waitせず次のtickまで待機する方法を定める。HALはホスト時間源とホストスレッドの待機だけを担当し、Process、Scheduler状態、sleep期限を管理しない。

Process選択、sleep期限の記録・満了判定、起床状態への遷移は[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)および[`sleep_implementation_spec.md`](sleep_implementation_spec.md)に従う。MC68000/Cortex-M0のtimer・idle実装は本書の対象外である。

## 2. HAL API

```cpp
namespace HAL {

uint32_t get_tick_count();
bool wait_for_tick_change(uint32_t observed_tick);

}
```

### 2.1 `get_tick_count()`

- `HAL::init_hal()`完了後に利用できる。
- ホストの`CLOCK_MONOTONIC`を基準とし、HAL初期化時点をtick 0とする。
- 経過時間を1 ms単位に切り下げて`uint32_t`で返す。値は`uint32_t`範囲で循環する。
- `CLOCK_REALTIME`やUNIX epochを使用しない。時刻合わせ等による壁時計の変更でtickが逆行・飛躍してはならない。
- 本APIはProcessやSchedulerを参照せず、呼び出すたびにホスト単調時計からtick値を得る。tick専用スレッドや1 msごとのポーリングループを作らない。
- `clock_gettime()`が失敗してtick値を得られない場合、偽の0や以前の値を返してはならず、ホストHALのfatal errorとして扱う。待機中に発生した時計読出しエラーは`wait_for_tick_change()`の`false`で通知する。

### 2.2 `wait_for_tick_change(observed_tick)`

- `HAL::init_hal()`完了後、Schedulerの通常実行文脈から呼び出す。
- 現在tickが`observed_tick`と異なる場合はブロックせず`true`を返す。
- 現在tickが`observed_tick`と同じ場合は、少なくとも次のtick境界までホストOS上で呼出しスレッドをブロックする。戻るときにtickが変化していれば`true`を返す。
- 待機中に複数tick進んでもよい。呼び出し元は返却後に`get_tick_count()`を再取得し、経過時間と各sleep期限を再評価する。
- 回復不能な時計読出しまたはホスト待機エラー時は`false`を返す。HALはProcess状態を変更しない。`EINTR`は回復可能な中断として扱い、tickを再確認して未変化なら同じ次tick境界まで待機を続ける。
- `observed_tick`とtickの比較は値の不一致を検出するためだけに使う。待機期限やsleep時間の計算は行わない。
- APIはtick変化のイベントを保存・配送する機構ではない。`observed_tick`を渡すことで、呼出し前にtickが進んでいた場合の取りこぼしを防ぐ。

## 3. Linuxホストでの実装契約

- `init_hal()`で`clock_gettime(CLOCK_MONOTONIC, ...)`を読み、ホストtickの基準時刻を保存する。
- `get_tick_count()`は基準時刻からの経過ミリ秒を算出して`uint32_t`へ変換する。加算・減算・変換の範囲を検査し、符号付きoverflowや未定義動作に依存しない。
- `wait_for_tick_change()`は単調時計の現在値と保存した基準時刻から次の1 ms tick境界を求め、`clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, ...)`相当の絶対時刻待機でブロックする。
- 絶対待機が`EINTR`で中断された場合は現在tickを再確認する。tick未変化なら同じ絶対境界を使って再開し、相対時間を足し直して待機時間を延長しない。
- tickが進む前に待機境界へ達した場合は、時計を再確認し、tickが変化するまで必要に応じて次の境界を待つ。busy-waitへ切り替えない。
- ホスト固有の`clock_gettime()`、`clock_nanosleep()`等はHAL実装ファイルに閉じ込める。Scheduler、ProcessManager、syscall handler、ユーザーコマンドから直接呼び出さない。
- HAL待機中にユーザーProcessや別のProcess Contextを実行しない。協調Schedulerがホストスレッドをブロックし、OSのスレッドスケジューラへCPUを返す。

Linuxの`CLOCK_MONOTONIC`は不連続な壁時計変更で後退せず、`clock_nanosleep()`の`TIMER_ABSTIME`は指定した絶対時刻まで待機する。`EINTR`時に同じ絶対時刻で再試行することで、相対待機の繰り返しによる遅延の累積を避ける。ホストAPIの詳細はLinux man-pagesの[`clock_gettime(3)`](https://www.man7.org/linux/man-pages/man3/clock_gettime.3.html)および[`clock_nanosleep(2)`](https://man7.org/linux/man-pages/man2/clock_nanosleep.2.html)を参照する。

## 4. Schedulerとの接続

Schedulerは次の順序で待機する。

1. dispatch前にHAL tickを取得し、期限満了した`WAITING(SLEEP)`を`READY`へ戻す。
2. READY Processがあればラウンドロビン規則でdispatchする。
3. READY Processがなく、SLEEP待ちProcessが1つ以上あれば、現在tickを`observed_tick`として`HAL::wait_for_tick_change(observed_tick)`を呼ぶ。
4. 戻り値が`true`なら手順1へ戻る。tickを再取得し、期限を満了したProcessのみ起床させる。
5. 戻り値が`false`ならSchedulerを異常終了させ、カーネルのfatal error経路へ移行する。WAITING Processを成功扱いで再開したり、sleep syscallへ成功を返したりしない。

SchedulerはREADY Processがあるときに本APIを呼ばない。実行中Processが明示的にyield/block/exitしてSchedulerへ制御を戻した後に期限を再評価するため、協調Schedulerの非プリエンプティブ性は変わらない。Processがyield等を行わず実行を続ける場合、他Processの期限が過ぎてもそのProcessを割り込まない。

## 5. 動作段階

- ホストの`HOST_UCONTEXT`では本書のHAL機能を使用できる。これにより正数`sleep()`をScheduler仕様に従って実装可能とする。
- `NO_CONTEXT`ではSchedulerが本APIを呼ばず、`sleep()` syscallは既存仕様どおり未対応とする。
- マイコンは`wait_for_tick_change()`相当の非busy-wait timer/idle機構を実装するまで正数`sleep()`を有効化しない。CPU固有の待機命令や割込み処理をこのホスト仕様へ持ち込まない。

## 6. 確認項目

1. 初期化後のtickが1 ms単位で進み、`CLOCK_REALTIME`変更による影響を受けない。
2. `wait_for_tick_change(current_tick)`が次のtickまでブロックし、成功時にtickが変化している。
3. 呼び出し前にtickが進んでいれば、待機せず`true`を返す。
4. 待機中に複数tickが経過しても`true`を返し、呼び出し側が現在tickで期限を再評価する。
5. `EINTR`後も同じ絶対tick境界まで待ち、相対待ち時間を積算しない。
6. 回復不能な待機失敗が`false`となり、Process状態を変更しない。
7. 全Processがsleep中のSchedulerでhost CPUをbusy-waitせず、期限経過後にProcessが起床する。
8. READY Processが存在する間はHAL tick待機を行わない。
9. HAL以外の共通コードからホスト固有clock/sleep APIへの直接依存がない。
