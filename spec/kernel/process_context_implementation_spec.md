# プロセスContext実装仕様書

## 1. 目的と適用範囲

本書は、`Process`ごとの実行Contextと専用スタックをホストLinux上で実装する方式、およびContextを生成しない互換ビルドを定める。通常のプロセス状態、PID、FD所有権、終了・回収規則は [`process_management_spec.md`](process_management_spec.md) に従う。Scheduler無効のwait駆動実行は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)、Scheduler有効時の選択・dispatchは[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)に従う。

ホストの `ucontext` は開発・デバッグ用バックエンドであり、本番マイコン向け実装ではない。Cortex-M0およびMC68000では将来CPU固有のContext実装（必要に応じてasm）を用意する。`ucontext_t`やホストOSのメモリ保護機構を共通Process APIやターゲットABIへ露出させない。

## 2. 実行モード

Context機能はビルド時に選択し、1つのカーネルイメージ内でProcessごとに混在させない。

| モード | Processの実行 | スタック・Context |
|---|---|---|
| `HOST_UCONTEXT` | SchedulerがREADY Processを選択し、yield/block時にProcessとScheduler間を協調的に切り替える | `init`を含む各Processが独立スタックと`ucontext_t`を持つ |
| `NO_CONTEXT` | 既存方式を維持し、親の`wait()`上で子入口を通常関数として呼ぶ | 子用スタック・Contextを作らない |

- ホストではコンパイル時設定 `BTX_USE_UCONTEXT` で選択する。`1`は`HOST_UCONTEXT`と協調Scheduler、`0`は`NO_CONTEXT`を選ぶ。初期値はビルド構成側で明示し、コード内で実行時自動判定しない。
- `BTX_USE_UCONTEXT=0` は `NO_CONTEXT` を選び、既存の同一スタック実行を保つ。
- `BTX_USE_UCONTEXT=1` はホストLinux専用とする。ホスト以外のターゲットでこの値を有効にしたビルドは、未対応バックエンドを黙って選ばずビルドエラーにする。MC68000/Cortex-M0でSchedulerを有効にする場合は、別のCPU固有Context設定を用意する。
- 将来のマイコン向けContextバックエンドは別のビルド設定と実装で追加する。ホスト用 `ucontext` ヘッダーやシンボルをターゲットビルドへ含めない。
- `HOST_UCONTEXT`のSchedulerモードでは本書のContextと[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)を組み合わせる。`NO_CONTEXT`は旧wait駆動互換方式であり、Schedulerおよび`sleep()`との機能同等性を要求しない。

## 3. Context層とProcessの境界

- `ProcessManager`はProcessの選択、状態遷移、現在Processの更新、生成失敗のrollbackを担う。
- Context層はContext初期化、保存、復元、専用スタックの確保と解放だけを担う。PID、FD、親子関係、終了状態、スケジューリング方針を管理しない。
- `ucontext_t`とホストメモリマッピング情報はホストContext実装内に隠す。Process公開APIにホスト依存型を含めない。
- Contextおよびスタックの所有権はProcessに対応付け、子Processが回収されContextが非実行状態になった後に一度だけ解放する。ProcessやContextの単純コピーは禁止する。
- `HOST_UCONTEXT`で初期化済みの子Contextは、その子の`Process`枠が生存する間、`ucontext_t`とスタック領域のアドレスを変更しない。

## 4. ホスト子スタック

- 各子Processに独立した有限長スタック領域を割り当てる。スタック容量はコンパイル時設定 `HOST_PROCESS_STACK_SIZE` とし、ページ境界・整数演算・`size_t`への変換を検証する。
- 容量の初期数値は 131072 バイト (128KiB) とする。未設定や不正値を無制限スタックとして扱わず、ホストContextモードの設定エラーとする。
- Linuxホストではページ単位のメモリマッピングを確保し、使用可能スタック領域の下側に少なくとも1ページのガード領域を設ける。ガード領域は `PROT_NONE` とし、子スタックが境界を越えたときホストOSが `SIGSEGV` を発生させる構成にする。
- 使用可能領域を `ucontext_t::uc_stack` に設定する。スタック方向、ページサイズ、アラインメント、サイズ加算と丸めはホストバックエンドに閉じ込める。
- マッピングまたは保護設定に失敗した場合、子生成を失敗させ、`spawn()`の他の準備済み資源もrollbackする。
- ガードページによる検出をマイコンへ一般化しない。MMU/MPU等の保護機構がないターゲットではスタック越境が周辺メモリを破壊し得ることを許容する。

## 5. Context生成と実行開始

`HOST_UCONTEXT`では、`spawn()`の生成処理中に子スタックを確保し、`getcontext()` / `makecontext()`相当で子Contextを初期化する。Contextおよびスタックの準備完了前にProcessを`READY`として公開しない。Scheduler有効時はPID 1の`init`にも同等の独立スタックと初期Contextを準備し、Scheduler run loopから開始する。

- Contextの開始点はホスト専用トランポリンとする。ポインタを`makecontext()`の可変長整数引数へ直接変換して渡さず、開始時のProcess特定には現在Processまたは範囲検証済みのProcess table slotを使う。
- トランポリンは子のユーザープログラム入口を実行する。入口関数からのreturnはその戻り値を終了statusとして子を終了させる処理へ渡し、子Contextから通常returnしない。
- 子は`spawn()`から復帰するまで実行しない。SchedulerがREADYになった子を選択し、Scheduler Contextから子Contextへ切り替える。
- 協調SchedulerはyieldやblockでProcessが自発的に制御を返した場合だけ切り替える。タイマー割込み、プリエンプション、複数CPU並行実行は導入しない。
- `wait()`を子が呼ぶ場合、呼出し元をWAITINGにしてSchedulerへ制御を戻す。再開時は保存Contextからwait処理を続行する。

## 6. 終了、親への復帰、回収

- 子入口のreturnと子からの`exit(status)`は共通の子終了経路へ合流し、`int32_t` statusを保存する。FD参照の解放とProcess状態遷移はProcess管理層の規則に従う。
- Scheduler有効時のProcessは終了処理後、Scheduler Contextへ非復帰で切り替える。Context層は切替を行い、ZOMBIE化やFD解放などの終了ポリシーを実装しない。
- Processの実行スタックとContextは、そのProcess自身のスタック上では解放しない。Scheduler Contextへ戻った後、ProcessManagerが終了Processを再開不能と確認してスタックとContextを解放し、その後に子を`ZOMBIE`として確定し、待機中の親を起床させる。
- `ZOMBIE`は終了結果と親子関係を保持するが、実行スタック・Contextは保持しない。`wait()`の出力先不正で回収が遅れても、終了済みの実行資源は保持しない。
- 子が`wait()`されずに残る場合も、終了後のContextとスタックは解放済みとし、Process枠の回収までは終了情報だけを保持する。

## 7. スタック越境時の扱い

- ホスト子スタックがガード領域へ越境した場合、子Process単体の異常終了として回収・継続しない。
- ガード違反に対する専用`SIGSEGV`回復ハンドラや、シグナルハンドラからのProcessManager cleanupを実装しない。既定の致命的なホスト異常終了により、デバッガがfault address、Context、スタックマッピングを調査できる状態を保つ。
- ホストスタックマッピングの開始・終了アドレスとProcess識別情報を、デバッグ時に対応付けられるようにする。通常運転中に独自の回復動作やログ出力を挟まない。
- デバッグ用のスタック境界照会は[`process_debug_info_spec.md`](process_debug_info_spec.md)の共通診断APIを用いる。境界を提供できないバックエンドは境界値に同仕様のsentinelを設定する。`ucontext_t`や保存レジスタ値を内蔵コマンドへ直接公開しない。
- `NO_CONTEXT`では子専用スタックやガードページがないため、この保護は適用されない。
- 本仕様は悪意あるプログラムからの隔離やメモリ安全性を保証しない。

## 8. 資源失敗と生成rollback

子生成のどの段階で失敗しても、`READY`として外部公開する前に以下を元に戻す。

1. 確保済みスタックマッピングとContext資源
2. 継承済みFileDescription参照と子fd table
3. コピー済み引数および実行イメージ資源
4. `CREATING`状態とProcess slot

Context初期化失敗は通常の`spawn()`失敗として呼出し元へ返す。部分的な子Processや孤児スタックマッピングを残さない。既存のsyscall ABIの失敗値を変更しない。

## 9. 確認項目

1. `HOST_UCONTEXT`では異なる子が異なるスタック領域を使い、親の保存されたローカル変数と呼出し位置が復元される。
2. `spawn()`時点で子の入口は実行されず、`wait()`で選択された後に一度だけ実行される。
3. 子のreturnと`exit(status)`で親へ戻り、終了status、FD参照、ZOMBIE状態が既存契約に従う。
4. ネストしたspawn/waitで各親子のContextとcurrent Processが正しく復元される。
5. Process枠不足、スタック確保失敗、ガード設定失敗、Context初期化失敗、FD継承失敗が途中状態を残さない。
6. スタック越境試験は隔離した子プロセスのテスト実行器で行い、ガードページによりデバッガが捕捉できる致命的faultとなることを確認する。越境後のProcess回収は期待しない。
7. `NO_CONTEXT`では独立スタック・Context関連ヘッダー・ホストOSメモリ保護APIを参照せず、旧来の親スタック上でのwait駆動実行を維持する。
8. ホストContext有効・無効の両構成を個別にビルドでき、ターゲット向けビルドに`ucontext`依存が混入しない。
