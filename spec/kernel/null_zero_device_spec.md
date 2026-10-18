# NullDevice / ZeroDevice 仕様書

## 1. 目的・適用範囲

本書はAncient UNIX風サブセットOSのFD管理機構の動作検証に用いる、`/dev/null`相当の`NullDevice`と`/dev/zero`相当の`ZeroDevice`を定義する。両者は既存の`FileInterface`を実装するカーネル内部デバイスである。

当面はテストコードから直接`FileDescription`へ結び付ける。`/dev/null`や`/dev/zero`というパス名での`open()`、デバイスノード、`mknod`、デバイスファイルシステムは実装対象外とする。この名称は動作を説明する便宜上の名称である。

本書にない共通事項は`open_file_description_spec.md`および`AGENTS.md`に従う。本書はそれらを変更しない。

## 2. クラスと公開API

```cpp
class NullDevice final : public FileInterface {
public:
    void init() override;
    int32_t read(void* buf, uint32_t count) override;
    int32_t write(const void* buf, uint32_t count) override;
};

class ZeroDevice final : public FileInterface {
public:
    void init() override;
    int32_t read(void* buf, uint32_t count) override;
    int32_t write(const void* buf, uint32_t count) override;
};
```

- `init()`は呼び出し可能な空実装（戻り値なし）とする。カーネル起動時は`HAL::init_hal()`後、`kernel::initialize_devices()`が登録テーブル経由で呼び出す。単体テストでは直接呼び出してよい。現在はHALに依存しない。
- デバイスは内部に可変状態を持たず、動的メモリ確保をしない。`init()`を繰り返しても動作に変化はない。
- それぞれのデバイス実体は、必要ならカーネル全体で1個を共有してよい。グローバル実体を置く場合は`G_null_device`、`G_zero_device`とする。
- 本デバイスには固有の`close()`、`seek()`、cleanup用の公開APIを設けない。

## 3. read/write動作

| 操作 | `NullDevice` | `ZeroDevice` |
|---|---|---|
| `read(buf, count)` | データを読み込まず、常に`0`（EOF）を返す | `buf`先頭から`count`バイトを`0x00`で埋め、`count`を返す |
| `write(buf, count)` | データを使用せず、`count`を返す | データを使用せず、`count`を返す |
| `count == 0` | `read`、`write`とも`0` | `read`、`write`とも`0` |

いずれのデバイスもファイルポジションを持たない。I/Oによって`FileDescription`のoffsetを進める必要はない。`lseek()`は両デバイスとも対象外であり、seek可否の共通機構が確定した時点で扱いを定める。

### 3.1 引数の契約

- `int32_t`の非負戻り値で処理バイト数を表せるように、有効な`count`は`0`〜`INT32_MAX`とする。これを超える要求は成功扱いにせず、上位層の入力検証で拒否する。
- `ZeroDevice::read()`で`count > 0`の場合、`buf`は`count`バイト書き込み可能な有効領域を指す必要がある。`count == 0`ならバッファへアクセスしない。
- `NullDevice::read()`および両デバイスの`write()`では、バッファの内容を参照しない。従って本デバイス自身はバッファをデリファレンスしない。ただしsyscall入口でのユーザーバッファ検証を省略してよいという意味ではない。
- `nullptr`、不正アドレス、過大`count`に対して返す具体的な負のエラー値は、現行`FileInterface`仕様で未確定である。本書では新しいエラー番号を定義しない。**本デバイスAPIへ渡す前に、呼び出し側が上記前提を検証する**。後日、共通エラー規約が確定した場合はそれに従う。

## 4. FD機構への接続と寿命

両デバイスは`FileInterface&`として既存の`alloc_file_description(FileInterface& target, uint32_t flags)`へ渡す。通常の`FdTable::alloc()`で任意の空きfdに登録できる。

```text
read(fd) / write(fd)
  -> FdTable
  -> FileDescription（アクセスモードを検証）
  -> FileInterface
  -> NullDevice / ZeroDevice
```

- `O_RDONLY`、`O_WRONLY`、`O_RDWR`のアクセス可否は`FileDescription`が判定する。デバイス実装側で重複して管理しない。
- `dup()`や`spawn()`に伴う共有・参照カウントは既存のFileDescription管理APIの責務とする。
- デバイス実体は、それを参照するすべての`FileDescription`より長く存続させる。
- 最後のfdをcloseした際も、`FileDescription`だけを解放対象とし、共有中の`NullDevice`/`ZeroDevice`実体は破棄しない。
- カーネル起動時のデバイス初期化は`kernel::initialize_devices()`が担当する。テスト用fdへの接続は引き続きテストコードの責務であり、標準入出力の`initialize_standard_fds()`を変更しない。

## 5. テスト要件

単体テストおよびFD機構の結合テストとして、少なくとも以下を確認する。

1. `NullDevice::read()`は正の`count`でも`0`を返し、バッファを変更しない。
2. `ZeroDevice::read()`は要求された範囲だけを`0x00`で埋め、範囲の前後は変更せず、要求バイト数を返す。
3. 両デバイスへの`write()`は要求バイト数を返す。データの内容に依存しない。
4. `count == 0`ではread/writeとも`0`を返し、メモリにアクセスしない。
5. `init()`前後および複数回の`init()`で、動作が変わらない。
6. それぞれ異なるfdに登録し、syscall→FdTable→FileDescription→FileInterface経由で期待どおり動作する。
7. `O_RDONLY`のwrite、`O_WRONLY`のreadはFileDescription層で拒否され、デバイスへ到達しない。
8. 同一FileDescriptionを複数fdで共有し、`retain`→`detach`→`release`により参照数・最終解放・スロット再利用が正しく動く。
9. `MAX_FD`および`MAX_OPEN_FILES`の枯渇・失敗時に不正な参照や途中登録が残らない。
10. `count > INT32_MAX`および無効バッファは、実際に経由する上位境界の検証仕様に従って拒否される（共通エラー規約の確定後に具体的な戻り値を試験する）。

## 6. 実装上の制約

- C++の例外、RTTI、ヒープ確保を使用しない。
- `FileInterface`で既に許可されている仮想メソッド以外の抽象階層を追加しない。
- `ZeroDevice::read()`は`count > 0`のときのみバッファをゼロクリアする。`NullDevice`はバッファへのアクセスを必要としない。
- HAL、ファイルシステム、ホストOSのfd・標準入出力には依存しない。
- syscall ABI、FileDescriptionの参照数管理、既存エラー規約を変更しない。

## デバイス登録テーブルからの初期化（共通規約）

`FileInterface`には`virtual void init() {}`が定義され、各デバイスの`init() override`はその実装である。カーネル起動時は`kernel::initialize_devices()`が非公開の登録テーブルを走査し、登録エントリごとに`init()`を呼ぶ。同じ実体が異なるmajor番号で複数回登録されていても管理機構側では呼び出しを省略せず、重複初期化の安全性は各ドライバが保証する。`find_char_device()`は初期化を行わず、fdのopenごとにも`init()`を呼び出さない。詳細は`device_driver_spec.md`に従う。
