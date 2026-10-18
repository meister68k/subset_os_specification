# `times()` システムコール実装仕様

## 1. 目的と範囲

本書は、呼び出し元Processとその終了済み子Processの累積実行時間、およびOS起動後の経過時間を取得する `times()` を定める。HALのtickは [`subset_os_overview_spec.md`](../subset_os_overview_spec.md) の契約に従い、1 msを1単位とする。UNIX時刻は返さない。

実行時間の計上はProcess管理/Schedulerの責務とする。HALはtick値を提供するだけで、Process別のCPU時間を算出しない。

## 2. syscall ABI

```cpp
struct tms {
    int32_t tms_utime;
    int32_t tms_stime;
    int32_t tms_cutime;
    int32_t tms_cstime;
};

int32_t sys_times(struct tms* buffer);
```

- 成功時、`buffer` に呼び出し元のuser/system累積tickと、終了済み子Processのuser/system累積tickを格納する。
- 成功時の戻り値はシステム起動後の経過tick数を `int32_t` で返す。
- 失敗時は `-1`。errno機構は追加しない。
- 公開ABIのフィールド幅・符号は上記の固定幅型とし、ホストの `long` 幅に依存させない。
- syscall番号は本書では割り当てない。

tick値を `int32_t` に表現できない場合は、変換前に失敗とし、出力bufferを変更しない。出力の部分更新を許さないため、全フィールドと戻り値を事前に検証してからbufferへ書き込む。

## 3. 時間の定義と計上

- `tms_utime` はProcessがユーザーコードを実行した累積tick数。
- `tms_stime` はProcessのためにカーネルコードを実行した累積tick数。
- `tms_cutime` / `tms_cstime` は、`wait()`により回収された子Processのuser/system時間の累計。
- `times()`を呼ぶsyscall処理自体の時間は、返却するスナップショットに含めない。
- 1 ms未満の実行時間はtick分解能で切り捨てる。カウンタの加算・集計でoverflowする場合は飽和させず失敗として扱う。
- tickの計測基準点はHAL実装に委ねられる。戻り値はそのtick基準の経過値であり、壁時計時刻ではない。

Processのuser/system区分を実行文脈やSchedulerで正しく計測できる段階まで、カウンタを更新する仕組みが必要である。単に `HAL::get_tick_count()` の差分を呼び出しProcessのCPU時間とみなしてはならない。実行時間カウンタの保存場所とuser/system区分はProcess管理/Scheduler仕様で定める。

## 4. 処理経路と責務

```text
btx::sys_times(buffer)
  -> syscall dispatcher
  -> kernel::sys_times(buffer)
       -> buffer検証
       -> ProcessManagerから時間スナップショットを取得
       -> HAL::get_tick_count()
       -> bufferへ全フィールドを書き込む
```

- syscall handlerはユーザーbufferを検証し、Process管理APIからスナップショットを取得して結果を返す。
- 実行時間と子Process累積値の更新・所有はProcess管理/Schedulerに置く。
- HAL tickを読む際は `HAL::init_hal()` 完了後であること。
- ユーザー領域の検証・コピー方法はsyscall入口仕様に従う。無効なbufferへの直接書き込みでカーネルを破壊してはならない。
- API形状は実装時に既存の依存注入方式へ合わせてよいが、handlerがProcess内部フィールドを直接変更してはならない。

## 5. エラーと部分更新

- `buffer == nullptr`、書き込み可能範囲の検証失敗、Processの時間スナップショット取得失敗、または戻り値/フィールド幅への変換失敗は `-1` とする。
- 失敗時はbufferの内容を変更しない。
- 成功時は全4フィールドを一貫した同一スナップショットとして返す。
- 経過tickの `uint32_t` 循環はHAL契約に従う。絶対値としての経過時間を `int32_t` で表せない範囲の戻り値は失敗とする。

## 6. 初期段階と後続段階

現行の初期単一Process/同期spawn段階では、ユーザー実行時間とカーネル実行時間を区別して測るScheduler計測契約がありません。この段階では `times()` は未対応とし、`-1` を返してbufferを変更しない。HAL tickを使った壁時計経過時間だけを返して、実行時間フィールドを推測で埋めてはならない。

SchedulerがProcessごとの実行区間を計測できる段階で本syscallを有効化する。子の時間は `wait()`による回収時に親のchild-time累計へ一度だけ加算し、同じ子を重複集計しない。集計overflow時の`wait()`との連携は、Process管理仕様で失敗・回収順序を定める必要がある。

## 7. 受け入れ確認項目

- null/不正bufferでは `-1` を返し、bufferを変更しない。
- 成功時に4つの時間フィールドとシステム起動後tickを返す。
- syscall自身の処理時間を取得時点のスナップショットへ混入させない。
- `HAL::get_tick_count()` の値をProcess CPU時間として流用しない。
- 子Processの時間はwait回収時に一度だけ親のchild累計へ反映される。
- フィールド、戻り値、累積加算のoverflowを検出し、暗黙のnarrowingやwraparoundをしない。
- 初期段階では未対応として失敗し、bufferを変更しない。
