# デバイスドライバ管理仕様書

## 1. 目的と適用範囲

本書は Ancient UNIX風サブセットOS の**キャラクタデバイス番号とデバイスドライバ実体との対応付け**、検索API、初期化・寿命の共通規約を定義する。

対象は、既存の `FileInterface` 実装である `ConsoleDevice`、`RedConsoleDevice`、`NullDevice`、`ZeroDevice`、および将来追加される同種のデバイスである。

個々の `read()` / `write()` 動作は各デバイス仕様に従う。`FileDescription`・`FdTable`・参照数の管理は `open_file_description_spec.md`、全体規約は `AGENTS.md` に従う。ファイルシステム内部の `BlockDevice` は本登録テーブルの対象外とし、現行の抽象を変更しない。

本段階では `/dev/*` のパス名解決、デバイスノード生成、`mknod()`、キャラクタデバイスの通常の `open()` 経路、動的なドライバ登録・解除は実装しない。

## 2. 設計の原則

- major番号は**ドライバの登録先**を識別する。番号は不連続でよく、登録数に比例した小さなテーブルを用いる。
- minor番号はドライバ内の装置を識別するが、**現段階で有効なのは0のみ**とする。
- 登録されたmajor番号は相互に重複してはならない。
- デバイス実体はカーネル存続期間中有効なものとする。登録テーブルはそれらを所有・破棄しない。
- 検索は静的テーブルの線形探索とし、動的メモリ確保や汎用ドライバマネージャは導入しない。
- `FileInterface` に `virtual void init()` を追加し、登録デバイスの起動時初期化を共通の仮想呼び出しで行う。major/minor番号や登録管理機能は追加しない。
- デバイス番号や登録テーブルを `FdTable` に持ち込まない。

## 3. デバイス番号

デバイス番号の論理構成は以下とする。

| 項目 | 型 | 制約 |
|---|---|---|
| major | `uint16_t` | 登録テーブルにある番号のみ有効。欠番を許容 |
| minor | `uint16_t` | 現段階では `0` のみ有効 |

実際の番号割当はプラットフォーム／カーネル構成で確定する。以下は**例示であって確定値ではない**。

| major（例） | minor | ドライバ |
|---:|---:|---|
| 1 | 0 | `ConsoleDevice` |
| 5 | 0 | `RedConsoleDevice` |
| 10 | 0 | `NullDevice` |
| 11 | 0 | `ZeroDevice` |

同じ `FileInterface` 実体を複数のmajor番号に関連付けてもよい。ただし、major番号自体の重複は禁止する。

## 4. 静的登録テーブル（実装内部）

major番号とデバイス実体の対応付けには、`std::array` による静的登録テーブルを用いる。**登録テーブル `G_char_devices` およびその要素型は実装ファイル内部に隠蔽し、公開ヘッダでは宣言しない。** `CharDeviceTable<N>` のようなテーブル管理クラスやSingletonパターンは導入しない。

以下は概念的な実装例であり、major番号は確定値ではない。具体的な `FileInterface` 派生オブジェクトは、別途構築されたカーネル共通デバイス実体を参照する。

```cpp
// device_driver.cpp（実装例）
#include <array>
#include <cstdint>
#include <functional>
#include <optional>

namespace kernel {
namespace {

struct CharDeviceEntry {
    uint16_t major;
    std::reference_wrapper<FileInterface> device;
};

// 具体的なデバイス実体と同じ翻訳単位に置く等、
// 静的初期化順序と寿命を保証する。
const std::array<CharDeviceEntry, 4> G_char_devices{{
    { 1,  G_console_device },
    { 5,  G_red_console_device },
    { 10, G_null_device },
    { 11, G_zero_device },
}};

} // namespace

void initialize_devices() {
    // 登録順に呼ぶ。同一実体が複数登録されていても重複排除しない。
    for (const CharDeviceEntry& entry : G_char_devices) {
        entry.device.get().init();
    }
}

std::optional<std::reference_wrapper<FileInterface>>
find_char_device(uint16_t major, uint16_t minor) {
    if (minor != 0) {
        return std::nullopt;
    }
    for (const CharDeviceEntry& entry : G_char_devices) {
        if (entry.major == major) {
            return entry.device;
        }
    }
    return std::nullopt;
}

} // namespace kernel
```

- `N` は登録件数であり、最大major番号ではない。欠番は空きスロットとして確保しない。
- 登録内容は静的に確定し、動的登録・解除、`add()` / `remove()` / `clear()` は設けない。
- major番号の重複は禁止する。重複の有無は構成レビューまたはテストで確認する。重複時の検索結果に依存してはならない。
- 各エントリは `FileInterface` 実体への非所有参照を保持し、null登録は許さない。
- `G_char_devices` は命名規約に従うが、外部リンケージを与えない。カーネルの他の翻訳単位から直接参照してはならない。
- デバイス実体の寿命はカーネル稼働期間中保証する。テーブル構築時にデバイスの `init()` やI/Oは行わない。

## 5. デバイス検索API

**公開インターフェース**は、`kernel`名前空間にある次の2関数とする。

```cpp
// device_driver.h
#include <cstdint>
#include <functional>
#include <optional>

namespace kernel {
void initialize_devices();

std::optional<std::reference_wrapper<FileInterface>>
find_char_device(uint16_t major, uint16_t minor);
}
```

`FileInterface` の宣言・インクルードはプロジェクトのヘッダ構成に従う。両関数とも登録テーブルの内部表現を公開しない。`initialize_devices()` と `find_char_device()` は同じ実装ファイル（例：`device_driver.cpp`）に定義する。

- `minor != 0` なら `std::nullopt` を返す。
- `minor == 0` かつmajorが登録されていれば、その `FileInterface` 実体への非所有参照を返す。
- 未登録majorなら `std::nullopt` を返す。
- 内部の静的テーブルを先頭から線形探索する。検索では登録内容、デバイス状態、FDおよび参照カウントを変更しない。
- 未登録majorと非対応minorを区別する専用エラー型は設けない。
- 検索APIはデバイスの `init()` を呼ばない。

利用例：

```cpp
std::optional<std::reference_wrapper<FileInterface>> dev =
    kernel::find_char_device(major, minor);
```

テストでは公開APIから本番登録内容を検査する。登録テーブルを差し替えられるテスト専用公開APIは設けない。

## 6. デバイス初期化API

```cpp
namespace kernel {
void initialize_devices();
}
```

- カーネル起動処理が `HAL::init_hal()` の完了後に一度呼び出す。`FileDescription` と標準FDの初期化・initプロセスの起動より前に完了させる。
- 非公開の `G_char_devices` を先頭から末尾まで走査し、各エントリの `FileInterface::init()` を登録順に一回ずつ呼び出す。検索APIを経由せず、テーブルの参照をそのまま用いる。
- **同じデバイス実体が複数のmajor番号に登録されている場合も、エントリごとに `init()` を呼ぶ。** 管理機構側は重複排除・初期化済み判定を行わない。重複呼び出しへの対応は各デバイスドライバの責務とする。
- 戻り値は `void`。現段階の共通初期化APIでは失敗通知・再試行・逆順終了処理を定義しない。
- 動的なドライバ登録、ファイルディスクリプタ確保、参照数操作、read/write呼び出しは行わない。
- 呼び出し元が起動時に一回実行することを想定する。誤って再度呼び出された場合にも各エントリの `init()` を再実行するため、必要な多重初期化対策はドライバ側で行う。

## 7. デバイスドライバ共通規約

1. 対象は既存の `FileInterface` を継承し、同インターフェースの `read()` / `write()` 契約を守る。
2. `FileInterface` に `virtual void init() {}` を追加する。標準実装は何もしない。個々のデバイスは必要に応じて `void init() override` を実装する。これは起動時のデバイス初期化であり、open時の初期化ではない。
3. デバイス初期化は `kernel::initialize_devices()` が登録テーブル経由で行う。検索APIは `init()` を呼ばない。複数回の `init()` を安全に扱う必要がある場合は各ドライバが対応する。
4. 登録テーブルはデバイスのI/O成否やファイル位置を処理しない。`FileDescription` がアクセスモード等を管理し、デバイスは自身のI/Oを実装する。
5. `FileDescription` の最終 `release` で、静的に共有されたデバイス実体を破棄しない。
6. 例外・RTTI・実行時ヒープ確保を必要とする仕組みは追加しない。

## 8. 起動時初期化との関係

初期化の概念的な順序は以下とする。

```text
カーネルのC++ランタイム／静的オブジェクトの構築
    -> HAL::init_hal()
    -> kernel::initialize_devices()
       （G_char_devicesを登録順に走査して各デバイスのinit()を呼ぶ）
    -> FileDescription管理テーブルと init用FdTableの準備
    -> initialize_standard_fds(...)
    -> initプロセスの実行開始
```

HALに依存するデバイスは `HAL::init_hal()` の後で初期化する。`NullDevice`・`ZeroDevice` など、HAL不要のデバイスの `init()` は空でもよい。初期化の段取りはカーネル起動処理が明示する。登録テーブルへの動的な登録処理は設けない。`initialize_devices()` が `FileInterface::init()` を仮想呼び出しする。

`initialize_standard_fds()` は既存の仕様どおり、fd 0/1/2へ与えられた `FileInterface&` を関連付ける。**標準FD初期化は検索テーブルを経由する必要がない**。必要なら呼び出し側が `kernel::find_char_device()` の結果を渡せるが、この使い方は必須としない。

## 9. 将来のデバイスファイルとの接続境界

`/dev/zero` 等のパス名を使う将来の `open()` では、ファイルシステムが解決したデバイスノードから `(major, minor)` を取得し、`kernel::find_char_device()` で `FileInterface&` を得る。その後の `alloc_file_description(target, flags)` と `FdTable::alloc(...)` は既存のFD管理APIに委ねる。

```text
open("/dev/zero", flags, mode)
  -> パス名解決 / デバイスノード（将来）
  -> (major, minor)
  -> kernel::find_char_device(major, minor)
  -> FileInterface&
  -> alloc_file_description(target, flags)
  -> FdTable::alloc(...)
  -> fd
```

これは**責務分担の将来像**であり、現在の `/dev` ノード表現・`mknod()`・デバイスファイルのopen時エラー規約を決定するものではない。現段階のテストは検索結果の参照を直接FD管理APIへ渡してよい。

## 10. テスト要件

| 区分 | 検証事項 |
|---|---|
| 正常検索 | 登録された各majorとminor=0で、正しいデバイス実体への参照が得られる |
| 欠番 | 最大・最小登録番号の間にある未登録majorは `nullopt` |
| 範囲 | `uint16_t` の境界番号（例: 0、65535）も登録有無に応じて安全に処理できる |
| minor | 登録済みmajorでもminor=1や65535は `nullopt` |
| 無変更 | 反復検索でデバイス参照と登録テーブルが変わらない |
| 構成検査 | major番号が重複しないことを確認する（参照メンバによりnull登録は不可） |
| テーブル隠蔽 | 登録テーブルへ直接アクセスせず公開検索APIのみで検索できる |
| 初期化 | `initialize_devices()` が登録順に各エントリの `init()` を呼び、HAL依存ドライバではHAL初期化が先行する |
| 同一実体の重複登録 | 同じ実体が複数エントリにあってもエントリごとに `init()` が呼ばれ、管理機構が重複排除しない |
| 多重初期化 | 同じ実体への複数回の `init()` の安全性が必要なら、そのデバイス自身のテストで検証する |
| 初期化の責務 | `initialize_devices()` は検索結果・FD・参照数を変更せず、`find_char_device()` は初期化しない |
| FD結合 | `kernel::find_char_device()`で検索した参照を用いて `FileDescription` を確保し、任意fdに登録して対象ドライバへI/Oを委譲できる |
| 責務分離 | 検索処理だけでは `FileDescription` や `FdTable`、参照カウントが変更されない |

テストでは実際のデバイス番号が未確定である点に注意し、採用された構成に即して期待値を設定する。ホストテストでは必要に応じてダミー `FileInterface` を利用し、登録構成に応じた検索結果を検証してよい。

## 11. 今回決定しない事項

- 個々のmajor番号の本番割当、およびデバイス番号の永続形式・inode上の符号化。
- minor番号1以上の解釈と、複数装置を扱うドライバの設計。
- `/dev` ノード作成、パス名解決、`mknod()`、デバイスノードの権限。
- ブロックデバイス向けmajor/minor登録方式。
- ドライバの動的登録・ホットプラグ。
- キャラクタデバイス専用の `open` / `close` / `ioctl` APIの追加。

これらは必要になった時点で、既存のFD・FileSystem仕様と整合を取って検討する。
