# schedtest(1)

## 名前
**schedtest** — 協調Schedulerのyieldと再開を確認する

## 書式
`schedtest`

## 説明
デバッグ用コマンドです。worker A/Bを各3回進捗出力するようspawnし、各workerは最初の2回の出力後に`sleep(0)`でyieldします。さらに`ps`を起動してProcess名を表示します。全子Processをwaitで回収します。

引数は受け付けません。`HOST_UCONTEXT`での協調切替確認用です。`NO_CONTEXT`ではyield未対応のため、失敗終了が期待されます。

## 終了ステータス
- `0`: workerのyieldと`ps`が成功
- `1`: 引数、spawn/wait、worker、出力等のエラー
