# ps(1)

## 名前
**ps** — Process情報を表示する

## 書式
`ps`

## 説明
デバッグ用の内蔵コマンドです。Process tableの使用中Processをslot順に表示し、PID、PPID、名前、状態、Context有無、stack/guard範囲を出力します。境界情報が得られない場合もAPIのsentinel値を数値表示します。ユーザー向けsyscallは追加しません。

引数は受け付けません。stack使用量やhigh-water markは表示しません。

## 終了ステータス
- `0`: 一覧出力成功
- `1`: 引数、Process情報取得、または出力エラー
