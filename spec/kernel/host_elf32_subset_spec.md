# ホスト環境 ELF32 実行形式サブセット仕様

| 項目 | 内容 |
|---|---|
| 文書ID | HOST-ELF32-SUBSET-SPEC-001 |
| 版 | 1.0 |
| 対象 | ホスト環境エミュレーション上の32bit guest実行イメージ |
| 関連仕様 | [`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)、[`shebang_execution_spec.md`](shebang_execution_spec.md)、[`process_context_implementation_spec.md`](process_context_implementation_spec.md)、[`host_executable_loader_design_memo.md`](host_executable_loader_design_memo.md) |
| 状態 | 仕様 |

## 1. 目的と範囲

本書は、`spawn()`がFilesystemから読み込むホスト環境用実行ファイルとして受理するELFサブセットを定める。ELFの汎用ローダ、動的リンク、任意のELF ABIを実装するものではない。対象guestはOSと同じ32bit x86/System V cdecl ABIでビルドし、ロード先に依存しない位置独立コードとする。

シバンの認識・interpreter選択・argv変換は[`shebang_execution_spec.md`](shebang_execution_spec.md)に従う。シバンでないファイルは本書のELF検査へ進む。

## 2. 32bit linker出力の確認結果

`tmp/loader_test/guest.c`を次の条件で32bit objectへコンパイルし、GNU `ld -m elf_i386`でリンクして確認した。

```text
gcc -m32 -fPIC -ffreestanding -nostdlib -O0 \
  -fno-stack-protector -fno-builtin -fno-unwind-tables \
  -fno-asynchronous-unwind-tables -fno-common -c guest.c
ld -m elf_i386 -z noexecstack -T guest.ld guest.o -o guest.elf
```

既存の`guest.ld`のままでは出力はELF32 little-endian `ET_EXEC`, `EM_386`で、program headerは単一`PT_LOAD`と`PT_GNU_STACK`になった。`PT_LOAD`の値は`p_offset=0x1000`, `p_vaddr=0`, `p_filesz=p_memsz=0x68`, `p_align=0x1000`, entryは0である。最終ELFの再配置は0件、未定義symbolはなく、sectionは`.text`, `.rodata`, `.got.plt`であった。

ただし、既存linker scriptの出力`PT_LOAD`は`RWE`であり、この仕様では受理しない。明示的な`PHDRS`でロードsegmentを`FLAGS(5)`（read/execute）に指定すると、同じguestから`PT_LOAD`=`R E`、`PT_GNU_STACK`=`RW`のELFを生成できた。したがってPHDR権限を明示したlinker scriptをguest build契約とする。`-z noexecstack`だけではload segmentのW+Xを防げない。

この確認は32bit linker出力の静的検査であり、32bit host上でのguest実行確認ではない。32bit ABIでの実行時PIC検証は受け入れ確認に残す。

## 3. ELF識別条件

ローダはELF構造体をnative C++ structとして直接読み込まず、Filesystemからbyte列として読み、指定byte orderと幅でfieldをdecodeする。整数演算・offset加算・範囲確認は加算や乗算の前にoverflowを検査する。

受理するELF headerはすべて次を満たす。

- `e_ident[EI_CLASS] == ELFCLASS32`
- `e_ident[EI_DATA] == ELFDATA2LSB`
- `e_ident[EI_VERSION] == EV_CURRENT`かつ`e_version == EV_CURRENT`
- `e_ident[EI_OSABI] == ELFOSABI_NONE`（System V）
- `e_type == ET_EXEC`
- `e_machine == EM_386`
- `e_ehsize == 52`
- `e_phentsize == 32`、`e_phnum == 2`
- `e_flags == 0`

`e_shoff`、`e_shnum`、`e_shstrndx`およびsection header tableは実行時ロードに使わない。section tableはbuild-time検査に使用できるが、実行時ローダはsection tableの存在を要求しない。

## 4. Program headerとload segment

program header table全体がファイル範囲内に収まることを検査する。2 entryの内訳は、順序を問わず`PT_LOAD`が1個、`PT_GNU_STACK`が1個でなければならない。他のprogram headerは拒否する。

### 4.1 `PT_LOAD`

唯一の`PT_LOAD`はすべて次を満たす。

- `p_vaddr == 0`。load addressに加えるrelative offsetを0基準で表す。
- `p_filesz > 0`かつ`p_filesz == p_memsz`。初期版では`.bss`を含むゼロ初期化領域を受け付けない。
- `p_flags`は`PF_R | PF_X`と完全一致する。`PF_W`を持つsegment、実行不可segment、未知permission bitを拒否する。
- `p_align == 0x1000`であり、`p_offset`と`p_vaddr`はそのalignmentに適合する。
- `p_offset + p_filesz`がoverflowせず、ファイルサイズ以下である。
- `p_memsz`は固定上限`MAX_GUEST_IMAGE_SIZE`以下である。初期値は1 MiBとする。

ELFの`e_entry`はload segment範囲内でなければならない。本仕様では`p_vaddr == 0`なので、実行時entryは`image_base + e_entry`となる。entry addressの計算が32bit guest address空間の範囲を超える場合は拒否する。

### 4.2 `PT_GNU_STACK`

`PT_GNU_STACK`は1個必須とし、`PF_X`を持ってはならない。guest stackはELF内からロードせず、Process Context仕様に従って別途用意する。ELFのstack segment sizeは使用しない。

## 5. 許可しない機能

次の機能は初期サブセットで受け付けない。関連program headerが存在するELFは§4により拒否される。

- `ET_DYN`、PIE、shared object
- 動的linker、`PT_INTERP`、`PT_DYNAMIC`
- load-timeまたはruntime relocation
- TLS、GNU_RELRO、複数`PT_LOAD`、追加のload segment
- writable load segment、`.data`、`.bss`など実行中に書き換える必要のあるimage data
- 外部symbol、外部library、PLT/GOTによる実行時解決
- 例外unwind、stack protector、CRT、host libc依存

PICであることはELF headerだけでは証明できない。guestのbuild規約とbuild-time検査で保証し、異なるload addressでの実行試験を行う。`-fPIC`だけを指定したことをPIC保証の根拠にしてはならない。

## 6. guest build契約

guestは32bit x86/System V cdecl ABIでビルドし、少なくとも次の制約を適用する。

- `-m32 -fPIC -ffreestanding -nostdlib`
- `-fno-stack-protector -fno-builtin -fno-unwind-tables -fno-asynchronous-unwind-tables -fno-common`
- ELF outputは`ld -m elf_i386`で生成し、`ET_EXEC`、VMA 0、単一RX `PT_LOAD`となるよう専用linker scriptでlinkする。
- `PT_GNU_STACK`をnon-executableにする。`-z noexecstack`を指定する。
- guestは読み取り専用のコード・データだけをimageに持つ。可変状態はProcess stack上に置く。
- guest entryおよびsyscall dispatch vectorは本OSが定めるguest ABIに従い、32bit幅を明示した型を使う。`long`をABI型として使用しない。

build artifactは`readelf`等で少なくとも次を検査する。

1. header識別値とprogram header構成が§3–4に合う。
2. final ELFに非空の`SHT_REL`または`SHT_RELA`がない。
3. 未定義の外部symbolがない。
4. load segmentがRWXまたは書込み可能でない。
5. `.data`、`.bss`、dynamic linkage、TLS等の未対応要素がない。

section tableをstripする場合もbuild-time検査を先に完了する。実行時loaderはsection名に依存しない。

## 7. spawn loaderの処理

Filesystem pathname対象がシバンでないと判定された場合、spawn loaderは次の順で処理する。

1. Filesystem APIでファイルを開き、ELF headerと2個のprogram headerを固定長bufferへ読み込む。
2. §3–5の形式・範囲・上限を検査する。ファイルサイズは既存Filesystem APIから取得する。
3. `p_memsz` byte以上のhost mappingを確保する。mapping addressはhost OSに選ばせ、`MAP_FIXED`相当で既存addressを上書きしない。
4. mappingをRWかつ非実行で作成し、`p_offset`から`p_filesz` byteを`image_base + p_vaddr`へ読み込む。
5. 読込完了後、mapping全体をR-Xへ変更する。W+X状態を作らない。
6. entry、guest argv、API vector、Context/stack、cwd、FDをProcessに関連付ける。
7. 全準備が成功してからProcessを`READY`として公開する。

page rounding、mapping byte数、file offset、entry計算の各算術はoverflowを事前検査する。Filesystem short read、mapping/protection失敗、close失敗、Process準備失敗ではProcessを公開せず、spawn失敗時のrollback契約に従う。host固有`mmap()`/`mprotect()`はhost backend内に閉じる。

シバン対象の場合はELF loaderへ渡さず、[`shebang_execution_spec.md`](shebang_execution_spec.md)のinterpreter解決・argv変換を行う。

## 8. Process・イメージ寿命

実行イメージmappingはProcess所有資源として扱い、guest命令が実行される全期間保持する。guestから`sys_exit()`した場合、またはentryがreturnした場合はcleanup/trampolineへ制御が戻り、guest image上の命令を再度実行しない状態になってからmappingを解放する。guest自身のstackや実行中コード上からmappingを解放してはならない。

Processがmappingを所有するfield、cleanup用Context、`NO_CONTEXT`での対応可否、reapとの厳密な順序はprocess context実装仕様で定める。これらが確定するまでは外部ELFを対象にしたProcess公開を実装しない。

## 9. 受け入れ確認項目

1. `tmp/loader_test/guest.c`を32bitでlinkし、修正済みPHDR linker scriptがELF32/i386、`ET_EXEC`、RX `PT_LOAD` 1個、non-executable `PT_GNU_STACK` 1個を生成する。
2. 既存の無指定PHDR linker scriptが生成するRWE `PT_LOAD`をローダが拒否する。
3. 最終ELFに再配置・未定義symbolがなく、サイズ・entry範囲検査が境界値で機能する。
4. guestを異なる複数のhost addressへmappingして同じ結果で実行できる。
5. guest実行中のmappingがRXであり、W+Xでない。
6. 不正magic/class/data/machine/type/version、program header構成、offset/size overflow、巨大segment、範囲外entry、W/X権限違反を拒否する。
7. short read、Filesystem error、mapping/protection/close failure時にProcess、FD、mappingを残さない。
8. guest entryからAPI vectorを経由し、syscall番号+最大4引数のdispatchが正しい32bit ABIで動作する。
9. guest returnと`sys_exit()`のどちらでも、親への復帰後にimageだけが適切なcleanup経路で解放される。

## 10. 検証上の制限

この仕様の根拠となる32bit実測はELFリンク出力と静的propertyの確認である。現行`tmp/loader_test/loader.c`は64bit host用で、guest entry prototypeも`long`およびsyscall番号+3引数を用いている。そのままでは本仕様の32bit guest ABI検証にならない。32bit guest API vectorを4 syscall引数対応・固定幅型へ改めた検証用loaderを別途用意し、実行時PIC・entry ABI・dispatchを確認する。
