# RedConsoleDevice 仕様書

## 1. 目的・適用範囲

本書は、Ancient UNIX風サブセットOSのFD管理機構を目視で検証するための、赤色文字出力デバイス`RedConsoleDevice`を定義する。`ConsoleDevice`とは独立して`FileInterface`を実装する、カーネル内部のテスト用デバイスである。

`RedConsoleDevice`への書き込みは、HALの文字出力にANSI SGR制御シーケンスを付加し、対応端末上で赤色表示する。デバイスは通常のコンソールと同じ出力経路を使うため、別の物理出力先を提供するものではない。fdごとの転送先の相違は文字色によって観測する。

当面はテストコードから`FileDescription`に直接登録する。デバイスノードやパス名による`open()`、汎用カラー端末、端末属性管理は本書の対象外とする。本書にないFD共通事項は`open_file_description_spec.md`および`AGENTS.md`に従い、両文書は変更しない。

## 2. クラスと公開API

```cpp
class RedConsoleDevice final : public FileInterface {
public:
    void init() override;
    int32_t read(void* buf, uint32_t count) override;
    int32_t write(const void* buf, uint32_t count) override;
};
```

- `init()`は現時点で何もしない`void`メソッドとする。`HAL::init_hal()`の後、`kernel::initialize_devices()`が登録テーブル経由で呼び出す。繰り返し呼んでも変化はない。
- 可変の内部状態やヒープ確保を必要としない。必要ならカーネル全体で単一インスタンスを共有してよい。グローバル実体を置く場合は`G_red_console_device`と命名する。
- `close()`、`seek()`、色変更用setter、独自のcleanupメソッドは設けない。

## 3. I/O仕様

### 3.1 `read(void* buf, uint32_t count)`

入力機能は持たない。`count`の値によらず`0`（EOF）を返し、`buf`にはアクセスしない。`HAL::getchar()`も呼び出さない。`count == 0`の場合も`0`を返す。

### 3.2 `write(const void* buf, uint32_t count)`

`count > 0`の場合、以下を**write呼び出し単位で**順番に`HAL::putchar(char)`へ出力する。

1. 赤文字指定: `ESC[31m`（バイト列 `1B 5B 33 31 6D`、5バイト）
2. `buf`の先頭から`count`バイト（無変換・順序維持）
3. SGRリセット: `ESC[0m`（バイト列 `1B 5B 30 6D`、4バイト）

成功時の戻り値は元のペイロードの`count`であり、追加した計9バイトの制御列は含めない。制御シーケンスはペイロード中の改行で区切らず、1回の`write()`の前後に各1回だけ挿入する。`count == 0`ではHALへ一切出力せず、`0`を返す。

`ESC[0m`は赤文字設定のみでなくSGR属性全般をリセットする。以前の表示属性の保存・復元は行わない。端末がANSI SGRに対応しない場合は制御バイトが見えたり、赤く表示されなかったりしてよい。

本デバイスは、書き込まれるペイロード自身のESCシーケンスを解析・無害化しない。そのためペイロードに別の属性変更シーケンスが含まれると、全ての文字が赤色になる保証はない。**通常のテスト文字列を赤色表示すること**を要件とする。

### 3.3 引数とエラーの契約

- `int32_t`の非負戻り値として扱える`count`は`0`〜`INT32_MAX`とし、超過する要求は成功扱いにせず上位境界で拒否する。
- `count > 0`の場合、`write()`の`buf`は`count`バイト読み取り可能でなければならない。`read()`は`buf`を参照しない。`count == 0`ではどちらもバッファに触れない。
- 不正ポインタ、過大`count`の具体的エラー値は共通`FileInterface`のエラー規約で未確定のため、本仕様では新しいエラー番号を設けない。呼び出し側は本APIの前提条件を満たすこと。syscall境界で要求されるユーザーポインタ検査は省略しない。
- 現在の`HAL::putchar(char)`は`void`を返すため、出力エラーや部分出力を検出・通知できない。**有効な呼び出しでは指定文字数を返す**契約とし、HAL出力の成功を検証できるという意味ではない。将来HALの契約を変更する場合は別途再検討する。

## 4. FD機構・HALとの接続

```text
write(fd, buf, count)
  -> FdTable
  -> FileDescription（アクセスモードの検査）
  -> FileInterface
  -> RedConsoleDevice::write()
  -> HAL::putchar()（ANSI赤色指定／本文／SGRリセット）
```

- `alloc_file_description(FileInterface& target, uint32_t flags)`に`RedConsoleDevice`を渡し、通常の`FdTable::alloc()`で登録する。
- `O_RDONLY`、`O_WRONLY`、`O_RDWR`のアクセス可否は既存の`FileDescription`で判定する。本デバイスはフラグを保持・再判定しない。たとえば`O_RDONLY`で登録されたfdの`write`は本デバイスへ到達しない。
- `dup()`、`spawn()`、`close()`、参照数の管理は既存FD管理機構に委ねる。デバイス実体は参照するすべてのFileDescriptionより長く存続し、最後の`close()`で破棄しない。
- `ConsoleDevice`と同じHAL文字出力先を使用する。ゲストfd 2をホストstderrへ直結しない。カーネル診断出力とゲストI/Oは混同しない。
- ANSIコードを含むひとまとまりの出力について排他制御・原子性は提供しない。他の出力との割り込み・混在によって色が乱れる可能性がある。排他方針は将来の並行処理設計に委ねる。
- ファイルポジションは持たず、`FileDescription::offset_`を使用しない。`lseek()`の扱いは既存の共通仕様に従って後日決定する。

## 5. テスト要件

1. `init()`の前後、および複数回呼び出しても動作が変わらない。
2. `read()`は正の`count`でもEOF (`0`)を返し、`buf`とHAL入力には触れない。
3. `write("A", 1)`のHAL出力バイト列が`ESC[31mAESC[0m`（計10バイト）で、戻り値が`1`になる。
4. 複数文字・改行を含む1回のwriteでは、本文前後の制御列を各1回だけ送出する。本文のバイト内容と順序を保持する。
5. `write(buf, 0)`は`0`を返し、ANSIコードを含めHALへ何も出力しない。
6. 2回のwriteでは、それぞれ独立して赤色指定とリセットを行う。
7. モックHALの文字出力を記録し、ANSIコードを含む全バイト列を検証する。端末画面の色確認だけを自動テストの合否基準にしない。
8. `ConsoleDevice`、`NullDevice`、`ZeroDevice`等と異なるfdに割り当て、FD経由の委譲が正しいことを検証する。
9. `O_RDONLY`のwriteは`FileDescription`層で拒否され、本デバイスに到達しない。
10. 同一FileDescriptionの共有および`detach()`／`release_file_description()`後の解放・再利用は既存のFDテスト規約に従って検証する。
11. FDやFileDescriptionの枯渇時に途中登録や参照数の破綻が生じないことを確認する。
12. 過大`count`と不正バッファの拒否は、実際に通る上位境界の規約に従って確認する（共通エラー値が決まった時点で詳細化）。

## 6. 実装上の制約

- 既存の`FileInterface`境界以外に仮想関数・抽象クラスを増やさない。
- C++例外、RTTI、動的メモリ確保を使用しない。
- ANSI制御列は定数とし、文字出力はHALの公開APIのみを使う。ホストの`printf()`やstderrへ直接書き込まない。
- `FileDescription`、`FdTable`、syscall ABI、HALの既存公開APIを本デバイスのために変更しない。
- テスト用デバイスの導入を理由に`initialize_standard_fds()`の動作を変更しない。必要なら呼び出し側から任意のfdに登録する。

## デバイス登録テーブルからの初期化（共通規約）

`FileInterface`には`virtual void init() {}`が定義され、各デバイスの`init() override`はその実装である。カーネル起動時は`kernel::initialize_devices()`が非公開の登録テーブルを走査し、登録エントリごとに`init()`を呼ぶ。同じ実体が異なるmajor番号で複数回登録されていても管理機構側では呼び出しを省略せず、重複初期化の安全性は各ドライバが保証する。`find_char_device()`は初期化を行わず、fdのopenごとにも`init()`を呼び出さない。詳細は`device_driver_spec.md`に従う。
