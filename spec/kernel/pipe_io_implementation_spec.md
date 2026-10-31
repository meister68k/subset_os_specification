# IPC pipe read/write実装仕様

## 1. 目的と適用範囲

本書は、[`pipe_implementation_spec.md`](pipe_implementation_spec.md)が作成するpipe endpointのread/write、有限バッファ、EOF、endpoint最終close、待機・起床を定める。pipeはbyte stream型IPCとし、read/write syscallと既存の`FileDescription -> FileInterface`経路で利用する。

syscall番号42、出力先fd pair、FdTableへの登録と生成rollbackは`pipe_implementation_spec.md`に従う。本書は`pipe2()`、メッセージ境界、シグナル、プリエンプションを導入しない。

## 2. 有限資源

pipe資源は固定長カーネル管理領域で所有し、一般heap allocationを使わない。

| 資源 | 上限 |
|---|---:|
| 同時pipe state数 | `MAX_PIPE_STATES = 4` |
| 1 pipeのbyte buffer | `PIPE_BUFFER_CAPACITY = 256` byte |
| 各pipeのread waiter数 | `MAX_PROCESSES` |
| 各pipeのwrite waiter数 | `MAX_PROCESSES` |

- 4 pipeのdata buffer合計は1024 byte。waiter配列は各固定長Process tableの上限を使う。
- `MAX_PROCESSES`の具体値はProcess管理仕様で定める。waiter数はProcess数を超えず、1 Processは同時に1つのsyscallでしかblockしない。
- pipe state pool枯渇時の`pipe()`失敗は[`pipe_implementation_spec.md`](pipe_implementation_spec.md)のrollback契約に従う。
- ring bufferの`head`、`used`、tail indexは`0..PIPE_BUFFER_CAPACITY-1`内で管理する。加算前に範囲を確認し、整数wraparoundを正しさに使わない。

## 3. Data modelとendpoint寿命

- 各pipe stateは256 byteのFIFO byte stream、読み出しendpoint参照の有無、書き込みendpoint参照の有無、read/write waiter FIFOを持つ。
- `pipe()`作成時はread endpoint 1つ、write endpoint 1つが開いている。read endpointは`O_RDONLY`、write endpointは`O_WRONLY`の別々の`FileDescription`で表し、両方から同一pipe stateを参照する。
- dup/spawnは通常のFileDescription retainを行い、同一endpointのopen lifetimeを延長する。duplicate fdごとにPipe endpointの別個のopenを作らない。
- endpointのopen状態が変わるのは、そのendpointを参照するFileDescriptionの最終`release()`時だけである。read/write endpointの最終releaseをそれぞれ一度だけPipe stateへ通知する。
- FdTableからfdをdetachしても、同じFileDescriptionを参照するfdが残る間はendpointをclose扱いにしない。最終close後はcounterを減らし、対応するwaiterを起床する。
- 両endpointの最終release後、操作中参照とwaiterがないことを確認してpipe stateを`FREE`へ戻す。buffer内容を保持したままslotを別pipeへ再利用しない。

`spawn()`は指定されたfdだけを継承する。`nullptr, 0`を指定した場合は親の全open fdを継承するため、親・子・兄弟Processが同じpipe端点を持つ場合、すべての参照がcloseされるまでendpointはopenのままである。EOFとreader消滅判定は実際に継承された参照も含めた最終FileDescription lifetimeに従う。pipelineの呼出し元は各子に必要なfdだけを指定し、不要なpipe端点を継承させない。

## 4. `read()`の動作

`sys_read(fd, buf, count)`は既存のFD共通経路からread endpointの`FileInterface::read()`へ到達する。

1. access modeがreadを許可しない場合は通常のFileDescription規則により`-1`。
2. `count == 0`なら状態を変更せず`0`を返す。waitしない。
3. bufferに1 byte以上ある場合、`min(count, used)` byteをFIFO順にコピーし、消費byte数を返す。要求`count`全体が満たされなくても、到着済みbyteがあれば直ちに短いreadを返す。
4. bufferが空で、write endpointがopenなら、呼出しProcessを`WAITING(IO)`として登録しSchedulerへ制御を渡す。block中にbusy-waitしない。
5. Processが再開したら状態を再確認する。データがあれば手順3、write endpointがclose済みで空ならEOFとして`0`を返す。
6. write endpointが最終close済みでもbufferにデータが残る間は、全buffer dataを順に読み出してから`0`を返す。

pipe readは通常ファイルpositionを持たず、`lseek()`は失敗する。

## 5. `write()`の動作

`sys_write(fd, buf, count)`は既存のFD共通経路からwrite endpointの`FileInterface::write()`へ到達する。

1. access modeがwriteを許可しない場合は通常のFileDescription規則により`-1`。
2. `count == 0`なら状態を変更せず`0`を返す。waitしない。
3. reader endpointが最終close済みなら`-1`を返す。シグナル送信やProcess終了は行わない。
4. bufferに空きがあれば`min(count, free_space)` byteを書き込み、転送byte数を直ちに返す。空きが要求量より少ない場合もpartial writeを許可し、1回のwrite callでbufferを満たす以上にblockしない。
5. bufferが満杯でreader endpointがopenなら、呼出しProcessを`WAITING(IO)`として登録しSchedulerへ制御を渡す。空きができて再開したら条件を再確認し、少なくとも1 byteを格納できる状態で手順4を行う。
6. 満杯で待機中にreader endpointが最終closeされた場合、再開後に`-1`を返す。

writer間のwrite call単位のatomicityや`PIPE_BUF`相当の保証はしない。byteのFIFO順は保持する。大きなwriteはbuffer capacity以下のpartial writeになり得る。

## 6. EOF・reader消滅・close通知

| 状態 | read endpointの結果 | write endpointの結果 |
|---|---|---|
| bufferにdataがある | dataを返す。writer不在でも残dataを先に排出 | 空きがあればdataを受け付ける。reader不在なら失敗 |
| buffer空、writer endpoint open | 待機してdataまたはclose通知を待つ | 空きがあれば書込み可能 |
| buffer空、writer endpoint最終close済み | EOFとして`0` | 該当endpointは利用不能 |
| reader endpoint最終close済み | 該当endpointは利用不能 | `-1`。満杯待ちwriterも起床して失敗 |

- 最終writer close時、pipe stateのbufferは保持する。blocked readerをすべて起こし、bufferを消費した後EOFを返させる。
- 最終reader close時、pipeの未読dataは破棄してよい。blocked writerをすべて起こして`-1`を返させる。readerを再生成する機能はない。
- dupされたfdやspawn継承fdが1つでもendpointを保持する間、最終closeとはみなさない。
- syscall `close()`は通常のFileDescription最終release規則を使う。close専用のpipe参照カウント経路を重複実装しない。

## 7. Block/wakeupとScheduler連携

### 7.1 Waiter登録

- blockが必要なProcessをPipe state上のread waiterまたはwrite waiter FIFO末尾へ登録する。waiter recordはPIDと当該pipe/direction識別子を持ち、容量は`MAX_PROCESSES`とする。
- waiter登録後、ProcessManagerを通じて`RUNNING -> WAITING`、`WaitReason::IO`へ遷移し、Schedulerへ制御を返す。
- 条件検査、waiter登録、Process状態遷移はlost wakeupを起こさない一つの同期区間として行う。協調Schedulerではこの区間内でyieldしない。将来のpreemption/複数CPUではProcess/pipe同期仕様で保護する。
- Pipe lock/critical sectionを保持したままSchedulerへblockしない。sleepやHAL idle待ちもpipe lock内で行わない。
- waiter queueが満杯ならProcess状態やbufferを変更せず`-1`を返す。通常の固定Process tableでは1 Process 1 waiterの上限により発生しないが、境界確認のため検査する。

### 7.2 起床

- writeがbufferへdataを追加したら、read waiter FIFOの先頭Processを1つREADYにする。
- readがbufferからdataを消費したら、write waiter FIFOの先頭Processを1つREADYにする。
- 最終writer closeはread waiter全員をREADYにする。各Processは再開後bufferを再確認し、dataまたはEOFを返す。
- 最終reader closeはwrite waiter全員をREADYにする。各Processは再開後reader不在を検出して`-1`を返す。
- wake対象は当該pipe・directionのwaiterであり、別pipeや別のwait reasonのProcessを起こさない。ProcessManager/Schedulerの公開内部APIで状態を変更し、Process tableをPipeから直接操作しない。
- 起床したProcessはSchedulerの通常READY選択順に従う。pipe waiter間の厳密な完了順やpriorityは保証せず、waiterをFIFOで選んでwakeする。

Schedulerは`WaitReason::IO`をpipe read/writeについて有効化する。device I/Oのblock/wakeupは引き続き本書の対象外とする。waiterが残っていてREADYもtimer待ちProcessもない場合、内部pipe条件で将来進捗できない状態はScheduler deadlock/fatal errorとして検出する。busy-waitや無限dispatchで隠さない。

### 7.3 `NO_CONTEXT`とbackend未対応

- `HOST_UCONTEXT`と、pipe I/O block/wakeupを実装した将来Context backendは本書のwait/wakeupを使用する。
- `NO_CONTEXT`またはI/O block/wakeup機能を持たないbackendでは、即時に処理できるread/writeは通常どおり行う。readがempty-openでblockを要する場合、またはwriteがfullでblockを要する場合は`-1`を返し、Process状態・waiter queue・pipe bufferを変更しない。
- backend未対応を理由にbusy-wait、sys_sleep loop、host thread waitを使わない。

## 8. read/write syscall共通規則

- `sys_read` / `sys_write`はfd解決、user buffer検査、FileDescriptionアクセスmode検査を行い、Pipeの種類を直接判定せずFileInterfaceへ委譲する。
- user bufferをpipe stateへ直接保持しない。block中は呼出しProcessのsyscall stack/contextを保存し、再開後に同じ呼出しのbufferとcountを再利用する。
- `count`は`uint32_t`。Pipeは最大256 byteを一度にbuffer内へ格納し、整数計算は転送前に境界を検証する。
- read/write failureは既存のI/O ABIに従って`-1`。成功値は実際に消費・格納したbyte数。EOFのreadは`0`。
- error後に未転送byteを成功値として返さない。partial transferが成功したcallは実際の転送数だけを返す。

## 9. 所有権・再利用・失敗保証

- Pipe stateのreader/writer open状態はendpoint FileDescriptionの最終releaseだけが更新する。`FdTable::detach()`自体やduplicate fd一つのcloseではendpointを早期close扱いにしない。
- blockしたProcessがあるpipe stateをfree/reuseしない。close通知でwaiterを起こし、waiterが再開してwait registrationを解除した後にstateをfree可能とする。
- wakeup処理中のPipe stateを別pipeへ再利用しない。pool slotの使用中/解放中状態を設け、二重releaseや二重wakeを防ぐ。
- buffer容量超過、waiter数超過、pool枯渇、参照counter上限では状態変更前に失敗するか、成功済みpartial transferだけを返す。counter overflow/underflowをwraparoundさせない。
- エラー伝搬にerrnoやC++ exceptionを使わない。既存FileInterface/I/Oの負値規約に従う。

## 10. 受入確認項目

1. pipeに書いたbyte列がFIFO順に読み出され、write/read境界をmessageとして扱わない。
2. emptyかつwriter openのreadがblockし、writerのdata追加後に再開して実データを返す。
3. bufferにdataがあるreadは要求countより短くても即時にpartial readを返す。
4. full buffer writeがblockし、readerがspaceを空けると起床してpartial writeを返す。
5. buffer capacity 256 byte、境界直前・満杯・空、wrap-around位置で範囲外アクセスやcount overflowがない。
6. writer endpoint最終close後、bufferの残dataを排出してからreaderがEOF 0を得る。空bufferなら直ちにEOFとなる。
7. reader endpoint最終close後、buffered dataを破棄し、blocked/次回writerが`-1`を得る。signalやProcess killは起きない。
8. dup/spawnしたendpoint参照を1つcloseしても、別参照が残る間はEOF/broken-reader条件にならない。
9. `count == 0`はblockせず0を返し、bufferとwaiter queueを変更しない。
10. 同一pipe/directionの複数waiterをFIFO順に起床し、close時は該当側waiter全員を起床する。
11. wait登録とwakeの境界でlost wakeupがなく、WaitReason不一致のProcessを誤ってREADYにしない。
12. `NO_CONTEXT`ではblockが必要なread/writeが`-1`となり、busy-waitせずpipe/Process状態を変更しない。
13. MAX_PIPE_STATES枯渇、waiter上限、最終参照解放、state再利用で資源漏れ・二重releaseがない。
14. `lseek()`はpipe endpointで失敗し、他のfd位置やbuffer dataを変更しない。

## 11. 関連仕様

- pipe endpoint pair作成とsyscall ABI: [`pipe_implementation_spec.md`](pipe_implementation_spec.md)
- FileInterface、FileDescription、fd参照と最終release: [`open_file_description_spec.md`](open_file_description_spec.md)、[`open_close_implementation_spec.md`](open_close_implementation_spec.md)
- `dup()`とFileDescription共有: [`dup_implementation_spec.md`](dup_implementation_spec.md)
- Process状態・wait reason: [`process_management_spec.md`](process_management_spec.md)
- Ready selection、block/wakeup、idle: [`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)
- spawn時のFD継承: [`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)
- sh0 pipelineの利用条件: [`../command/command_sh0_spec.md`](../command/command_sh0_spec.md)
