# 極小UNIX `sleep` コマンド仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-SLEEP-CMD-001 |
| 版 | 0.1（実装前） |
| 対象 | ユーザーランド内蔵バイナリ `sleep`（将来のpathnameは`/bin/sleep`） |
| 位置づけ | 指定秒数だけ呼び出しProcessを待機させるコマンド |
| 依拠する仕様 | [`sleep_implementation_spec.md`](../kernel/sleep_implementation_spec.md)、[`syscall_entry_spec.md`](../kernel/syscall_entry_spec.md)、[`spawn_wait_implimentation_spec.md`](../kernel/spawn_wait_implimentation_spec.md) |

## 1. 目的と範囲

単一の秒数引数を受け取り、ミリ秒単位のHAL tickへ変換して`sys_sleep()`を呼び出す。待機・yield・起床はカーネルのsleep syscallとSchedulerに委ね、コマンド自身はbusy-waitやHAL時刻の直接参照を行わない。

初版では整数秒1個のみを扱う。小数、単位suffix、複数引数、option、環境変数展開、ロケール依存の数値表記は対象外とする。

## 2. 起動形式

```text
sleep seconds
```

- `seconds`をちょうど1個指定する。引数なし、余分な引数、不明なoptionはusage errorとする。
- `seconds`はASCII数字`0`〜`9`のみからなる10進非負整数とする。先頭ゼロは許可する。符号、空白、小数点、指数表記、suffixは許可しない。
- 初期の内蔵バイナリ登録名は`"sleep"`とし、`kernel::user_binary("sleep")`で検索できるようにする。将来ファイルシステム上のpathnameを導入する場合は`/bin/sleep`とする。
- 現行`sh0`ではコマンド名`sleep`を内蔵バイナリ登録名`"sleep"`として起動する。`PATH`探索は行わない。

## 3. 引数検証とtick変換

- 秒数の上限は`UINT32_MAX / 1000`とする。現在のHAL契約では1 tick = 1 msのため、上限値は4,294,967秒である。
- 数値蓄積は各桁でoverflowを検査し、上限を超えた値をwraparoundやnarrowingで受け入れない。
- 検証後に`uint32_t ticks = seconds * 1000`を計算する。乗算前に上限を確認し、整数overflowを起こさない。
- `seconds == 0`は`ticks == 0`として`sys_sleep(0)`へ渡す。協調Schedulerが有効なら時間待ちなしでyieldし、Scheduler無効の互換段階ではsyscall仕様どおり失敗する。
- syscallの秒単位変換や正数待機期限をコマンド側で再実装しない。

## 4. 処理手順

1. `argc`が2であることを確認する（`argv[0]`を含む）。
2. `argv[1]`が空でなく、全byteがASCII数字であることを検証する。
3. 数値を上限`UINT32_MAX / 1000`以下として安全にparseする。
4. 秒数を1000倍して`uint32_t ticks`を作る。
5. `btx::sys_sleep(ticks)`を1回呼ぶ。
6. 戻り値が0なら何も出力せず終了値0で終了する。負値なら診断を標準エラーへ出し、終了値1で終了する。

`sys_sleep()`は番号35、引数`uint32_t ticks`、成功0・失敗-1である。動作段階ごとのSchedulerおよびHAL timer/idle要件は[`sleep_implementation_spec.md`](../kernel/sleep_implementation_spec.md)に従う。正数sleepがまだ有効でない段階では、syscallが-1を返すため本コマンドも失敗終了する。

## 5. 入出力・終了ステータス

| 状況 | 動作 |
|---|---|
| 引数形式不正、引数数不正 | usageをfd=2へ出し、`1`で終了。syscallを呼ばない |
| 数値overflow、秒数上限超過 | 範囲エラーをfd=2へ出し、`1`で終了。syscallを呼ばない |
| `sys_sleep()`成功 | 無出力で`0`終了 |
| `sys_sleep()`失敗 | 診断をfd=2へ出し、`1`終了 |

- usage例は`sleep: usage: sleep seconds\n`、範囲エラー例は`sleep: seconds out of range\n`、syscall失敗例は`sleep: sleep failed\n`とする。
- 診断は標準エラーfd=2へ出す。文言の厳密な表記は上記例と同等であればよい。
- 診断writeが部分書込みした場合は未送信分を再送する。0または負値で停止し、再帰的な診断や無限再試行をしない。
- コマンド自身はFDを開閉せず、Process状態、cwd、FD tableを直接変更しない。

## 6. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `btx::sys_sleep(uint32_t ticks)` | 待機またはyield |
| OSラッパ | `btx::sys_write()`相当 | 標準エラーへの診断 |
| ランタイム | `main()`の戻り値 | Process終了status |

- 動的メモリ確保、`FILE*`、`printf()`、host OS固有sleep API、`HAL::get_tick_count()`、カーネル内部Scheduler APIを使用しない。
- 数値parseとtick変換は固定幅整数で行い、乗算前に境界を検査する。
- `main()`の引数形式は内蔵バイナリ規約に従う。通常のProcessとして起動し、ProcessManagerの内部状態を直接変更しない。

## 7. 受入確認項目

1. `sleep 0`が`sys_sleep(0)`を一度呼び、結果に応じた終了値を返す。
2. `sleep 1`が`sys_sleep(1000)`を一度呼ぶ。
3. `UINT32_MAX / 1000`秒を受け付け、秒数上限の1超過をsyscall前に拒否する。
4. 先頭ゼロを許可し、負数、正符号、小数、空文字、非数字、数値overflowを拒否する。
5. 引数なし・余分な引数・option形式をusage errorにし、syscallを呼ばない。
6. `sys_sleep()`成功時は無出力で0、失敗時は診断して1となる。
7. `sys_sleep()`の失敗時もProcess状態とFD tableはコマンドによって変更されない。
8. 診断の部分writeを処理し、write失敗時に無限再試行しない。

## 8. 非対応機能

- 小数秒、`s`/`m`/`h`/`d`等のsuffix、複数時間引数の合算。
- option、`--`、環境変数、shell算術、ロケール依存表記。
- シグナルによる中断、残り時間の表示・返却。
- syscallを使わないhost sleepやbusy-waitへのfallback。
