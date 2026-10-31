# `getpid()` システムコール実装仕様

## 1. 目的と範囲

本書は、呼び出し元Process自身のPIDを返す `getpid()` syscallのABIと責務を定める。PIDの割当・有効期間・Processの現在実行参照は [`process_management_spec.md`](process_management_spec.md) を正本とする。本書はその状態を読み取る syscall 境界を規定し、PIDの割当規則は変更しない。

## 2. syscall ABI

```cpp
int32_t sys_getpid();
```

- syscall番号20を割り当てる。V7の `getpid` と同番号である。
- 引数は取らない。
- 成功時は呼び出し元ProcessのPIDを `int32_t` として返す。
- 通常のProcess実行中に呼ばれることを前提とし、`getpid()` は通常の処理条件では失敗しない。errno機構や新たなエラー値は設けない。
- カーネル起動Contextなど、current Processが存在しない文脈からsyscall handlerを直接呼ぶことは契約外とする。

`int32_t` は syscall ABI の整数幅に従う。Process管理側のPID型から返却値へ変換する場合は、値が表現可能であることを確認する。PIDのABI範囲はプロセス管理仕様および公開 syscall 定義と一致させる。

## 3. 処理経路と責務

```text
btx::sys_getpid()
  -> syscall dispatcher
  -> kernel::sys_getpid()
  -> ProcessManager::current().pid()
```

- syscall handlerは引数なしの薄いアダプターとし、PIDの割当やProcess状態変更を行わない。
- handlerはProcessManagerの公開APIを通じて現在のProcessを取得し、そのPIDを返す。
- PIDテーブルを独自に検索したり、テーブル添字からPIDを推測したりしてはならない。
- ProcessManagerやProcessの内部状態を直接変更しない。

カーネルhandlerは引数なしとし、ProcessManagerの既存のcurrent Process取得経路を利用する。PID表を独自に検索したり、テーブル添字からPIDを推測したりしてはならない。syscall外からも現在のProcessのPIDを読む必要がある場合は、Process管理APIを直接利用し、syscall dispatcherを経由させない。

## 4. 呼び出し条件

`ProcessManager::current()` は、現在実行中のProcessが存在する文脈でのみ呼び出せる。したがって `sys_getpid()` も通常Processの実行中に呼び出されることを前提とする。カーネル起動コンテキストはProcessとして登録されないため、`startup()`中から本syscallを発行しない。

この前提を満たす通常呼び出しでは、PIDをそのまま返し、Process状態、親子関係、FD、cwd、Contextその他の資源を変更しない。

## 5. 初期段階と後続段階

- 初期単一Process段階では、`init`のPID（通常は1）を返す。
- `NO_CONTEXT`では子入口関数を親スタック上で呼んでいても、ProcessManagerのcurrent Processが子なら子PIDを返す。子から親へ戻った後は親PIDを返す。
- `HOST_UCONTEXT`では子の専用Context上で実行中に子PIDを返し、親Contextへ戻った後は親PIDを返す。Context切替や実行主体の選択を本syscallが行ってはならない。
- Scheduler導入後も、呼び出し時点で実行中のProcessのPIDを返す。

## 6. 受け入れ確認項目

- 引数なしのsyscallとしてdispatchされる。
- init実行中にinitのPIDを返す。
- `NO_CONTEXT`で子入口実行中に子PIDを返し、親へ復帰した後に親PIDを返す。
- `HOST_UCONTEXT`で子専用Context実行中に子PIDを返し、親Contextへ復帰した後に親PIDを返す。
- 呼び出しによってProcess状態、PID割当、FD、cwdその他のProcess資源が変化しない。
- syscall handlerがPID表を独自に検索せず、ProcessManagerの現在Process APIを使う。
- syscall ABIの戻り値幅へ変換する際に、未検証のnarrowingを行わない。
