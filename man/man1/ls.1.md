# ls(1)

## 名前
**ls** — fileまたはdirectoryの内容を列挙する

## 書式
`ls [-alp] [path ...]`

## 説明
引数なしでは`.`を表示します。directoryは1 entry 1行で記録順に出力し、ソートしません。fileを直接指定した場合はそのpathを表示します。複数pathは引数順に処理します。

## オプション
- `-a`: `.`で始まるentryを含めます。
- `-l`: mode、link数、UID、GID、size、mtime、nameを表示します。
- `-p`: directory名の末尾に`/`を付けます。

短いoptionは結合または分割して指定できます。optionはpathより前に置きます。再帰表示、`--`、単独`-`は非対応です。

## 終了ステータス
- `0`: エラーなし
- `1`: optionまたはI/O/stat等のエラー

## 使用例
```sh
ls
ls -alp /tmp
```
