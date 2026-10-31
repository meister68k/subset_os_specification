# init(1)

## 名前
**init** — `/etc/inittab`を順に読み、processを実行する

## 書式
起動時にOSから実行されます。

## 説明
極小のSysV形式initです。`/etc/inittab`を先頭から読み、有効な各行のprocessを起動し、その終了を待ってから次の行へ進みます。対応actionは`wait`のみです。runlevelsは読み飛ばします。`respawn`や常駐監視は行いません。

行形式は`id:runlevels:action:process`です。空行と行頭コメントは無視します。process引数の分割はspace/tabによる単純分割で、引用符、escape、pipe、redirect、変数展開はありません。

## 設定例
```text
boot::wait:/bin/echo Starting
shell::wait:/bin/sh0
```

## エラーと終了
不正行は診断してskipし、次行へ進みます。設定fileのopen/read errorは異常終了します。子の非0終了はinit自身の失敗とはせず、次行へ進みます。initの最終statusは仕様上暫定です。
