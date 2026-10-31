# head(1)

## 名前
**head** — ファイルの先頭を出力する

## 書式
`head [-c count] [file]`

## 説明
先頭10行、または`-c`で指定した先頭byte数を標準出力へ出力します。行区切りはLFで、改行やbyteを補正しません。`-c`はbinary-safeです。短いreadはEOFとみなしません。

fileを省略するか`-`を指定すると、標準入力から読み込みます。pathnameを指定する場合はregular file 1個を対象とします。複数fileとdirectoryは非対応です。countはASCII数字の非負10進数で、`INT32_MAX`以下です。標準入力ではseekを使わず、入力を逐次読み込みます。

## オプション
- `-c count`: 先頭count byteを出力します。count 0なら出力byteはありません。

## 終了ステータス
- `0`: 成功
- `1`: 引数、file、read/write/close等のエラー

## 使用例
```sh
head file
head -c 32 file
echo hello | head
echo hello | head -c 3
```
