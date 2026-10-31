# load(1)

## 名前

**load** — Filesystem上の実行イメージをロードして実行する（デバッグ用）

## 書式

```text
load pathname [args ...]
```

## 説明

**load**は、第1引数`pathname`で指定したFilesystem上の実行イメージを読み込み、**load**自身のProcess内で実行します。実行イメージはELF32のサブセット形式に限定されます（`spec/kernel/host_elf32_subset_spec.md`参照）。

- 外部イメージロード機構の検証・デバッグ用のコマンドであり、**load**での実行は新しいProcessを作りません。guestと**load**は1つのProcessとして動きます。`getpid()`は**load**のProcessのPIDを返します。
- `pathname`は相対pathnameでは現在のディレクトリ（cwd）から解決します。`PATH`による探索は行いません。内蔵プログラム名の照合も行いません。
- 第2引数以降は順序を変えずguestへ渡されます。guestの`argv[0]`は指定された`pathname`の文字列そのままです。
- guestはentryからのreturnで終了し、その戻り値が**load**の終了ステータスになります。実行イメージはguestの実行が終わった後に解放されます。
- `sleep()`やpipeの読み書きをguestから呼んだ場合の待機は、通常のProcessと同じ契約に従います。

## オプション

ありません。`-`で始まる引数もpathnameまたはguest引数として扱います。`-`、`--`に特別な意味はありません。

## 非対応

- シバン（先頭が`#!`のファイル）は実行形式不正として扱います。シバン解決は`spawn()`側の機能です。
- 内蔵プログラム名（`!echo`等）での起動はpathname解決に任せます（照合しません）。

## エラー

- 起動に失敗した場合は標準エラーに`load: `から始まる診断を出します。失敗した場合、実行イメージのmappingとfdは後始末され、Process状態には残骸が残りません。
- guestの実行が始まった後の暴走・スタック越境は回収されず、致命的なhost異常終了となります。

## 終了ステータス

| 値 | 意味 |
|---:|---|
| guestの戻り値 | guestのentryが戻った場合。その値をそのまま返します |
| `126` | 使用方法の誤り（引数なし、pathnameが空文字列） |
| `127` | 起動不能（open失敗、実行形式不正・シバン、引数上限超過、mapping/読込/protect/close失敗） |

guestの戻り値が`126`や`127`でも、**load**自身の失敗と同じ値になります。区別が必要な場合は標準エラーの出力を参照してください。

## 使用例

```sh
# /bin/guest を実行
load /bin/guest

# 引数を渡して実行（guestのargvは /bin/guest hello world）
load /bin/guest hello world

# 終了ステータスの確認
load /tmp/myguest; echo $?
```

## 注記

**load**は`spawn()`への外部イメージロード統合の前段の検証用コマンドで、統合完了後もデバッグ用に維持されます。guestの実行には、ELF32サブセット仕様のguest build契約（`-m32 -fPIC -ffreestanding -nostdlib`等、`.data`・`.bss`なし、単一RX `PT_LOAD`、non-executable `PT_GNU_STACK`）でビルドした実行イメージが必要です。**load**を実行しているProcessの表示名はguest実行中も`load`のままです。
