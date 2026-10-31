# tail(1)

## 名前
**tail** — ファイルの末尾を出力する

## 書式
`tail [-c count] [file]`

## 説明
末尾10行、または`-c`で指定した末尾byte数を標準出力へ出力します。pathnameを指定した場合、`-c`では`SEEK_END`で末尾を得て、開始位置へseekしてから読みます。byte列は変更しません。

fileを省略するか`-`を指定すると、標準入力から読み込みます。標準入力はEOFまで読み込み、末尾を保持する固定領域は512 byteです。入力が512 byteを超えるとEOFまで読み捨てた後に失敗し、部分的な結果は出力しません。pathnameを指定する場合はregular file 1個を対象とします。複数file、directory、followは非対応です。countはASCII数字の非負10進数で、`INT32_MAX`以下です。標準入力では`lseek()`を使いません。

## オプション
- `-c count`: 末尾count byteを出力します。count 0なら出力byteはありません。

## 終了ステータス
- `0`: 成功
- `1`: 引数、file、seek/read/write/close等のエラー

## 使用例
```sh
tail file
tail -c 32 file
echo hello | tail
echo hello | tail -c 3
```
