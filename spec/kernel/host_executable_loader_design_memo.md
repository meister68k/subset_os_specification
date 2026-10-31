# ホスト環境のディスク実行ファイルローダー検討メモ

| 項目 | 内容 |
|---|---|
| 文書ID | HOST-EXEC-LOADER-MEMO-001 |
| 版 | 0.1（検討メモ） |
| 対象 | ホスト環境エミュレーション上のFilesystem実行ファイルと`spawn()` |
| 状態 | 設計検討中。規範的な実装仕様ではない |
| 関連資料 | [`../../tmp/loader_test/VERIFICATION.md`](../../tmp/loader_test/VERIFICATION.md)、[`host_elf32_subset_spec.md`](host_elf32_subset_spec.md)、[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)、[`process_context_implementation_spec.md`](process_context_implementation_spec.md)、[`syscall_entry_spec.md`](syscall_entry_spec.md) |

## 1. 目的と範囲

ホスト環境エミュレーションで、ディスクイメージ内のコマンド実行ファイルをFilesystem APIから読み込み、Processの実行対象として`spawn()`する方式を検討する。

本メモはホスト環境のみを対象とし、マイコン上の実行形式・メモリ配置・Context切替方式を定めない。loader_testが示す「ファイル読込、メモリ配置、実行属性への変更、entry関数呼び出し」の流れを参考にするが、host OSの`open()`でディスクイメージ外のファイルを読むのではなく、OSのFilesystem APIを介して読み込む。

## 2. 現行仕様との関係

- `spawn(name, argv, inherit_fds, inherit_fd_count)`は内蔵バイナリ登録名との完全一致を先に調べ、一致すれば内蔵プログラムを直接起動する。一致しなければ`name`をFilesystem pathnameとして解決する。
- 内蔵バイナリ名に`/`を含めない。`!echo`等は単なる登録名であり、予約名前空間ではない。内蔵名とpathnameの解決優先順位、シバン文法、argv変換の規範仕様は[`shebang_execution_spec.md`](shebang_execution_spec.md)を参照する。
- Filesystem pathnameは絶対pathおよび呼出し元cwd基準の相対pathを扱い、`spawn()`自身は`PATH`検索を行わない。
- syscall ABIの引数数・syscall番号は変更しない。
- 本メモは引き続きhost実行イメージ形式、32bit ABI、マッピング、Process内イメージ寿命等の検討に限定する。

## 3. loader_testから利用できる設計要素

[`../../tmp/loader_test/loader.c`](../../tmp/loader_test/loader.c)は、指定ファイルを読み、匿名RW mappingへコピーし、mappingをRXへ変更した後、mapping先頭を関数ポインタとして呼び出す。guestは`kvec`を介してloader側の関数を呼び、guestからhostへ戻った値を再度渡す。

[`../../tmp/loader_test/guest.ld`](../../tmp/loader_test/guest.ld)はimageをVMA 0から並べ、[`../../tmp/loader_test/Makefile`](../../tmp/loader_test/Makefile)は`.text`、`.rodata`、`.data`をraw binaryへ出力する。検証記録では位置独立性、リロケーションなし、実行中にW+Xとしないことが確認項目となっている。

この検証は現在のOSビルドABIをそのまま検証したものではない。記録対象のloader_testはx86-64ホスト上の検証であり、OSの32bitビルドへ適用するには、loader・guestを32bitで作り直し、calling conventionと生成コードを改めて検証する必要がある。

## 4. 32bit実行ABIに関する条件

- OSとguest executableは同一の32bit x86 ABI、endianness、calling convention、構造体ABIでビルドする。
- syscall番号・戻り値に`int32_t`、pointer値に`uintptr_t`を用いる。syscall ABIに`long`を使わない。
- 現在のsyscallには4個の引数を持つ`sys_spawn()`がある。loader_testのdispatch関数はsyscall番号の後に3引数までしか渡せないため、そのままでは不足する。
- 検討するguest API vectorのdispatch形状は、syscall番号と4個の引数を受け取る関数である。

```cpp
int32_t (*dispatch)(
    int32_t syscall_number,
    uintptr_t arg0,
    uintptr_t arg1,
    uintptr_t arg2,
    uintptr_t arg3
);
```

32bitビルドでは`uintptr_t`は32bit pointer値を表す。実際の関数宣言、vector構造体、entry引数、ユーザー側`system_call()` wrapperとのABI一致は別途確定する。

## 5. 実行形式の決定

実行形式は独自flat headerではなく、`ET_EXEC`のELF32/i386 subsetとする。guestはPICを必須とし、ELFはarchitecture/ABIの識別、load segment、entry pointの表現および構造検査に使う。runtime relocationは行わない。正式なELF許可範囲とloader検証条件は[`host_elf32_subset_spec.md`](host_elf32_subset_spec.md)に定める。

`tmp/loader_test/guest.c`を`gcc -m32 -fPIC -ffreestanding -nostdlib`でcompileし、`ld -m elf_i386`でlinkしてELF32 LSB, `ET_EXEC`, `EM_386`を確認した。program headerは1個の`PT_LOAD`と1個の`PT_GNU_STACK`となり、`PT_LOAD`は`p_offset=0x1000`, `p_vaddr=0`, `p_filesz=p_memsz=0x68`, `p_align=0x1000`, entryは0であった。最終ELFのrelocationは0件、undefined symbolもなかった。

既存`guest.ld`では`PT_LOAD`がRWEとなり、そのままでは不受理である。明示的PHDRSで`PT_LOAD FLAGS(5)`、`PT_GNU_STACK FLAGS(6)`を設定したテスト用linker scriptでは、load segmentをRX、stackをRW・非実行にできた。このlinker script要件をELF build contractに含める。

## 6. PICおよびbuild検証の残り

32bit objectを静的linkした結果は、最終ELFのrelocationが0件で、undefined symbolがなかった。ただし、ELFが`ET_EXEC`であることや最終relocationがないことだけでは、すべての機械語がload address非依存であると証明できない。guestを異なる実addressへmappingして動かす32bit実行検証と、4引数syscall対応のguest API vector検証は未実施である。

最終生成物はELF header/program header、relocation、undefined symbol、未対応sectionをbuild時に検査する。PIC性はbuild profileと複数addressでの実行試験を合わせて確認し、満たせない生成物を受理しない。

## 7. spawn時のロード・公開案

候補となる処理順序は次のとおり。

1. `name`が内蔵名かFilesystem pathnameかを解決する。
2. Filesystem pathnameの場合、既存Filesystem APIでファイルを開き、regular file・実行形式header・architecture・version・サイズ・entry offsetを検査する。
3. Process専用image mappingをhost backendで確保する。ロード時は書込み可能・非実行とし、payloadをFilesystem readで読み込む。
4. 読み込み完了後、image mappingを読取り・実行可能に変更する。書込み可能かつ実行可能な状態を作らない。
5. entry、コピー済み`argv`、guest API vector、Process Context/stack、cwd、指定FDを準備する。
6. すべての準備が成功した後にのみProcessを`READY`として公開する。
7. image mappingやContext等の準備に失敗した場合、Processを公開せず、今回確保した資源とFD参照をrollbackする。

hostの`mmap()`、`mprotect()`等はホスト実装内部に閉じ込め、ユーザー向けsyscall ABIにしない。mappingはOSに空きaddressを割り当てさせ、既存mappingを置き換える`MAP_FIXED`は通常経路で使わない。

## 8. Context・終了とimage寿命

- `HOST_UCONTEXT`ではkernel trampolineをContext開始点とし、guest entryを通常の関数呼び出しとして実行する。guest entryは初期案では`argc`、`argv`およびguest API vectorを受け取り、終了時にstatusを返す。
- entryへ直接命令ポインタを設定して飛び込む方式は、通常の関数callのstack規約・復帰先と異なるため採用しない。
- guestがreturnする場合は通常のProcess終了経路へ合流する。guestから`sys_exit()`された場合も既存のexit契約に従う。
- image mappingはguest命令が実行中の間保持する。guest code上またはguest stack上でmappingを解放してはならない。Processがscheduler/cleanup用Contextへ戻り、guest実行が停止した後に解放する。
- host simulatorとguestは同じhost address spaceを共有する。image mappingやContextを分けてもメモリ保護・sandboxにはならない。

具体的なentry ABI、`NO_CONTEXT`互換モードの扱い、Processがimage mappingを所有・解放する正確な状態遷移は未決である。

## 9. 確定前に必要な検証

1. loaderとguestをOSと同じ32bit ABIでビルドできることを確認する。
2. 32bitの最終imageについて、section配置、GOT、relocation、未定義symbol、生成命令を調査する。
3. 同一imageを異なるaddressへ複数同時mappingし、それぞれ正しく実行できることを確認する。
4. guest entryからdispatch vectorを通して既存syscallを呼び出す。最大4 syscall引数の経路も確認する。
5. guest returnと`sys_exit()`の両方で、親への制御復帰、wait status、image寿命が正しいことを確認する。
6. 不正header、範囲外entry offset、サイズ超過、短read、Filesystem error、mapping/protection failureがProcess・FD・mappingを残さないことを確認する。
7. 別Contextの複数guestを同時にREADYにし、mapping addressが異なっても正しくdispatchされることを確認する。

## 10. 主な未決事項

- 32bit guest entryおよび4引数syscall dispatch vectorの実行時検証。
- Processがexecutable mappingを所有するfieldとexit/reapまでの厳密な解放状態遷移。
- `HOST_UCONTEXT`以外のmodeでFilesystem executableを起動するか。

本メモは上記の未決事項を確定せず、ホスト環境の実装仕様書を作成する前の設計材料として扱う。
