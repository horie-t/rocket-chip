Rocket Chip Generator 解説
=====================

このリポジトリは、「[Rocket Chip Generator](https://github.com/freechipsproject/rocket-chip)」のコードを解説するため、及びコードに詳細なコメントを追加するためにフォークしたものです。解説は、[Wikiページ](./wiki)を参照してください。コードのコメントの追加状況は[コミットログ](https://github.com/horie-t/rocket-chip/commits/master)を参照してください。
他に、英語のRocket Chipの資料は、[technical report](http://www.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-17.html)にあります。

## 目次

+ [手早く始める](#quick) 早いところ詳細にまで飛び込んでしまいたい人向け。このリポジトリの正確な内容は後回しにして。
+ [Rocket chip generatorの内容は?](#what)
+ [How should I use the Rocket chip generator?](#how)
    + [Using the cycle-accurate Verilator simulation](#emulator)
    + [Mapping a Rocket core down to an FPGA](#fpga)
    + [Pushing a Rocket core through the VLSI tools](#vlsi)
+ [How can I parameterize my Rocket chip?](#param)
+ [Contributors](#contributors)

## <a name="quick"></a> 手早く始める

### コードをチェックアウト

    $ git clone https://github.com/ucb-bar/rocket-chip.git
    $ cd rocket-chip
    $ git submodule update --init

### 環境変数、RISCVの設定

このrocket-chipリポジトリをビルドするには、RISCV環境変数を設定する必要があります。この変数は、riscvツールをインストールするディレクトリを示しています。

    $ export RISCV=/path/to/install/riscv/toolchain

このriscvツールのリポジトリは、すでにrocket-chipリポジトリの中に、Gitのサブモジュールとして含まれています。riscvツールは、 *必ず* このバージョンを使用しなければいけません:

    $ cd rocket-chip/riscv-tools
    $ git submodule update --init --recursive
    $ export RISCV=/path/to/install/riscv/toolchain
    $ export MAKEFLAGS="$MAKEFLAGS -jN" # Nは、ホストシステムのCUPコアの数
    $ ./build.sh
    $ ./build-rv32ima.sh #(32ビット版RISC-Vを使う場合のみ)

より詳細については(もしくは、何か問題があれば)、この
[riscv-tools/README](https://github.com/riscv/riscv-tools/blob/master/README.md) をあたって下さい。

### インストールに必要な依存関係

このリポジトリを使うには、いくつかの追加パッケージをインストールする必要があります。
ここに全ての依存関係を並べるよりは、各サブプロジェクトの適切なREADMEの節を参照してください:

* [riscv-tools "Ubuntu Packages Needed"](https://github.com/riscv/riscv-tools/blob/priv-1.10/README.md#quickstart)
* [chisel3 "Installation"](https://github.com/ucb-bar/chisel3#installation)

### プロジェクトのビルド

最初に、C言語のシミュレータをビルドします:

    $ cd emulator
    $ make

もしくは、VCSシミュレータをビルドします:

    $ cd vsim
    $ make

どちらの場合でも、アセンブリ言語のテストセット、もしくは簡単なベンチマークを実行できます。
(Nコアのホストシステムを使っていると仮定します):

    $ make -jN run-asm-tests
    $ make -jN run-bmark-tests

VCDフォーマットの波形を生成できるCシミュレータをビルドするには:

    $ cd emulator
    $ make debug

そして、Cシミュレータ上でアセンブリ言語のテストを実行し、波形を生成するには:

    $ make -jN run-asm-tests-debug
    $ make -jN run-bmark-tests-debug

FPGA、もしくはVLSIで合成可能は、Verilogファイルを生成するには(`vsim/generated-src` に出力されます):

    $ cd vsim
    $ make verilog


### リポジトリを最新に保つ

あなたのリポジトリをGitHubリポジトリにアップデートする場合は、サブモジュールやツールもアップデートする必要があります。

    $ # このリポジトリの最新バージョンのファイルを取得する
    $ git pull origin master
    $ # サブモジュールを適切なバージョンにする
    $ git submodule update --init --recursive

riscvツールのバージョンが変わったら、[riscv-tools/README](https://github.com/riscv/riscv-tools/blob/master/README.md) にあるように、riscvツールを再コンパイルして、インストールすべきです。

    $ cd riscv-tools
    $ ./build.sh
    $ ./build-rv32ima.sh # (32ビット版RISC-Vを使う場合のみ)

## <a name="what"></a> Rocket chip generatorの内容は?

このrocket-chipリポジトリは、いつくかのサブ・リポジトリを指しているメタ・リポジトリです。サブ・リポジトリには、[Git submodules](http://git-scm.com/book/en/Git-Tools-Submodules) を使用しています。
これらのリポジトリは、SoCの設計を生成し、テストするツールが含まれます。
このリポジトリも、RTLを生成するために使うコードを含みます。
ハードウェアの生成は、[Chisel](http://chisel.eecs.berkeley.edu) を使って行われます。Chiselは、Scalaに埋め込むハードウェア構築言語(hardware construction language)です。(訳注: ハードウェア記述言語、hardware description language、HDLではありません)
rocket-chip generatorは、Scalaのプログラムで、Chiselコンパイラを起動して、完全なSoCを記述したRTLを出力します。
次の節からは、このリポジトリの内容を解説します。

### <a name="what_submodules"></a>Gitのサブモジュール

[Git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules) allow you to keep a Git repository as a subdirectory of another Git repository.
For projects being co-developed with the Rocket Chip Generator, we have often found it expedient to track them as submodules,
allowing for rapid exploitation of new features while keeping commit histories separate.
As submoduled projects adopt stable public APIs, we transition them to external dependencies.
Here are the submodules that are currently being tracked in the rocket-chip repository:

* **chisel3**
([https://github.com/ucb-bar/chisel3](https://github.com/ucb-bar/chisel3)):
The Rocket Chip Generator uses [Chisel](http://chisel.eecs.berkeley.edu) to generate RTL.
* **firrtl**
([https://github.com/ucb-bar/firrtl](https://github.com/ucb-bar/firrtl)):
[Firrtl (Flexible Internal Representation for RTL)](http://bar.eecs.berkeley.edu/projects/2015-firrtl.html)
is the intermediate representation of RTL constructions used by Chisel3.
The Chisel3 compiler generates a Firrtl representation,
from which the final product (Verilog code, C code, etc) is generated.
* **hardfloat**
([https://github.com/ucb-bar/berkeley-hardfloat](https://github.com/ucb-bar/berkeley-hardfloat)):
Hardfloat holds Chisel code that generates parameterized IEEE 754-2008 compliant
floating-point units used for fused multiply-add operations, conversions
between integer and floating-point numbers, and conversions between
floating-point conversions with different precision.
* **riscv-tools**
([https://github.com/riscv/riscv-tools](https://github.com/riscv/riscv-tools)):
We tag a version of the RISC-V software ecosystem that works with the RTL committed in this repository.
* **torture**
([https://github.com/ucb-bar/riscv-torture](https://github.com/ucb-bar/riscv-torture)):
This module is used to generate and execute constrained random instruction streams that can
be used to stress-test both the core and uncore portions of the design.

### <a name="what_packages"></a>Scalaパッケージ

独立したGitリポジトリに追随するサブモジュールに加えて、rocket-chipのコードは、いくつかのScalaパッケージとして分割できます。
これらのパッケージは、 src/main/scala ディレクトリで見つかります。
これらのパッケージのいくつかは、ジェネレータの構成のためのScalaユーティリティを提供し、その他が、実際のChiselのRTL ジェネレータ自身です。
ここに、各パッケージの手短な説明を挙げます:

* **amba**
このRTLパッケージは、AMBAプロトコル(AXI4、AHB-liteとAPBを含む)のバスの実装を生成するのに使います。
* **config**
このユーティリティ・パッケージは、ジェネレータを構築するためのScalaのインターフェイスを提供します。インターフェイスは、動的スコープのパラメータ・ライブラリを通して提供されます。
* **devices**
このRTLパッケージは、周辺機器の実装を含みます。これらにはデバッグ用モジュールと様々なTileLinkのスレーブが含まれます。
* **diplomacy**
このユーティリティ・パッケージは、2つのフェーズによるハードウェアのelaborationを許可する事によってChiselを拡張します。(This utility package extends Chisel by allowing for two-phase hardware elaboration) 拡張には、いくつかのパラメータが動的にモジュール間を調整する事が含まれます。 diplomacyの詳細は、[この論文](https://carrv.github.io/papers/cook-diplomacy-carrv2017.pdf) を参照の事。
* **groundtest**
このRTLパッケージは、合成可能なハードウェア・テスターを生成します。このテスターは、ランダム化したメモリアクセスのストリームを発行する事によって、コア外のメモリ階層のストレステストを実行します。
* **interrupts**
* **jtag**
このRTLパッケージは、JTAGバス・インターフェイスの定義を提供します。
* **regmapper**
このユーティリティ・パッケージは、メモリマップド・レジスタにアクセスするための標準的なスレーブ・デバイスを生成します。
* **rocket**
このRTLパッケージは、Rocketイン・オーダー・パイプライン・コアを生成し、L1命令、データキャッシュも生成します。
このライブラリは、chip generatorから使われる事を意図し、コアをメモリシステムの中でインスタンス化し、外界へ接続される事を意図しています。
* **subsystem**
このRTLパッケージは、他パッケージからの様々なコンポーネントを結びつけて、完全なコア構造体(coreplex)を生成します。コンポーネントに含まれるのは、タイル状に並べられた、Rocketコア、システム・バス・ネットワーク、データ一貫性エージェント(coherence agents)、デバッグ用デバイス、割り込みハンドラ、外部向けの周辺部、クロック間接続(clock-crossers)、TileLinkから外部バスプロトコル(例えば、AXIもしくはAHB)への変換器です。
* **system**
このトップ・レベルのユーティリティ・パッケージは、Chiselを起動して、特定のcoreplexの構成をelaborateします。同時に、適切なテスト用副生成物もelaborateします。
* **tile**
このRTLパッケージは、コアと組み合わせてtileを構築するコンポーネント、例えばFPUやアクセラレータを含みます。
* **tilelink**
このRTLパッケージは、TileLinkプロトコルのバスの実装を生成する手順を使用します。また、様々なアダプタとプロトコル変換器を含みます。
* **unittest**
このユーティリティ・パッケージは、個別モジュールの合成可能なハードウェア・テスターを生成するためのフレームワークを含みます。
* **util**
このユーティリティ・パッケージは、様々な共通のScalaとChiselのコンストラクタを含みます。このコンストラクタは、複数の他パッケージに渡って再利用されます。

### <a name="what_else"></a>他リソース

Scalaの外部では、完全なSocの実装を生成するための様々なリソースと、生成した設計のテストを提供します。

* **bootrom**
BootROMに含まれる、第一段階のブートローダのためのソース。
* **csrc**
Verilatorシミュレーションで使用するためのCソース
* **emulator**
Verilatorシミュレーションがコンパイル、実行されるディレクトリ。
* **project**
Scalaのコンパイル、ビルドのためのSBTによって使われるディレクトリ。
* **regression**
継続的インテグレーションと、ナイトリー・リグレッションを定義する。
* **scripts**
シミュレーションの出力の解析、もしくはソース・ファイルの内容を扱うユーティリティ。
* **vsim**
Synopsys VCSシミュレーションがコンパイル、実行されるディレクトリ。
* **vsrc**
インターフェイス、ハーネスとVPIを含むVerilogソース。


### <a name="what_toplevel"></a>トップ・レベルの設計の拡張

[ここの説明](https://github.com/ucb-bar/project-template) で、あなた独自設計のカスタム・デバイスをどのように作るかを見てください。

## <a name="how"></a> How should I use the Rocket chip generator?

Chisel can generate code for three targets: a high-performance
cycle-accurate Verilator, Verilog optimized for FPGAs, and Verilog
for VLSI. The rocket-chip generator can target all three backends.  You
will need a Java runtime installed on your machine, since Chisel is
overlaid on top of [Scala](http://www.scala-lang.org/). Chisel RTL (i.e.
rocket-chip source code) is a Scala program executing on top of your
Java runtime. To begin, ensure that the ROCKETCHIP environment variable
points to the rocket-chip repository.

    $ git clone https://github.com/ucb-bar/rocket-chip.git
    $ cd rocket-chip
    $ export ROCKETCHIP=`pwd`
    $ git submodule update --init
    $ cd riscv-tools
    $ git submodule update --init --recursive riscv-tests

Before going any further, you must point the RISCV environment variable
to your riscv-tools installation directory. If you do not yet have
riscv-tools installed, follow the directions in the
[riscv-tools/README](https://github.com/riscv/riscv-tools/blob/master/README.md).

    export RISCV=/path/to/install/riscv/toolchain

Otherwise, you will see the following error message while executing any
command in the rocket-chip generator:

    *** Please set environment variable RISCV. Please take a look at README.

### <a name="emulator"></a> 1) Using the high-performance cycle-accurate Verilator

Your next step is to get the Verilator working. Assuming you have N
cores on your host system, do the following:

    $ cd $ROCKETCHIP/emulator
    $ make -jN run

By doing so, the build system will generate C++ code for the
cycle-accurate emulator, compile the emulator, compile all RISC-V
assembly tests and benchmarks, and run both tests and benchmarks on the
emulator. If Make finished without any errors, it means that the
generated Rocket chip has passed all assembly tests and benchmarks!

You can also run assembly tests and benchmarks separately:

    $ make -jN run-asm-tests
    $ make -jN run-bmark-tests

To generate vcd waveforms, you can run one of the following commands:

    $ make -jN run-debug
    $ make -jN run-asm-tests-debug
    $ make -jN run-bmark-tests-debug

Or call out individual assembly tests or benchmarks:

    $ make output/rv64ui-p-add.out
    $ make output/rv64ui-p-add.vcd

Now take a look in the emulator/generated-src directory. You will find
Chisel generated Verilog code and its associated C++ code generated by
Verilator.

    $ ls $ROCKETCHIP/emulator/generated-src
    DefaultConfig.dts
    DefaultConfig.graphml
    DefaultConfig.json
    DefaultConfig.memmap.json
    freechips.rocketchip.system.DefaultConfig
    freechips.rocketchip.system.DefaultConfig.d
    freechips.rocketchip.system.DefaultConfig.fir
    freechips.rocketchip.system.DefaultConfig.v
    $ ls $ROCKETCHIP/emulator/generated-src/freechips.rocketchip.system.DefaultConfig
    VTestHarness__1.cpp
    VTestHarness__2.cpp
    VTestHarness__3.cpp
    ...

Also, output of the executed assembly tests and benchmarks can be found
at emulator/output/\*.out. Each file has a cycle-by-cycle dump of
write-back stage of the pipeline. Here's an excerpt of
emulator/output/rv64ui-p-add.out:

    C0: 483 [1] pc=[00000002138] W[r 3=000000007fff7fff][1] R[r 1=000000007fffffff] R[r 2=ffffffffffff8000] inst=[002081b3] add s1, ra, s0
    C0: 484 [1] pc=[0000000213c] W[r29=000000007fff8000][1] R[r31=ffffffff80007ffe] R[r31=0000000000000005] inst=[7fff8eb7] lui t3, 0x7fff8
    C0: 485 [0] pc=[00000002140] W[r 0=0000000000000000][0] R[r 0=0000000000000000] R[r 0=0000000000000000] inst=[00000000] unknown

The first [1] at cycle 483, core 0, shows that there's a
valid instruction at PC 0x2138 in the writeback stage, which is
0x002081b3 (add s1, ra, s0). The second [1] tells us that the register
file is writing r3 with the corresponding value 0x7fff7fff. When the add
instruction was in the decode stage, the pipeline had read r1 and r2
with the corresponding values next to it. Similarly at cycle 484,
there's a valid instruction (lui instruction) at PC 0x213c in the
writeback stage. At cycle 485, there isn't a valid instruction in the
writeback stage, perhaps, because of a instruction cache miss at PC
0x2140.

### <a name="fpga"></a> 2) Mapping a Rocket core to an FPGA

You can generate synthesizable Verilog with the following commands:

    $ cd $ROCKETCHIP/vsim
    $ make verilog CONFIG=DefaultFPGAConfig

The Verilog used for the FPGA tools will be generated in
vsim/generated-src. Please proceed further with the directions shown in
the [README](https://github.com/ucb-bar/fpga-zynq/blob/master/README.md)
of the fpga-zynq repository.


If you have access to VCS, you will be able to run assembly
tests and benchmarks in simulation with the following commands
(again assuming you have N cores on your host machine):

    $ cd $ROCKETCHIP/vsim
    $ make -jN run CONFIG=DefaultFPGAConfig

The generated output looks similar to those generated from the emulator.
Look into vsim/output/\*.out for the output of the executed assembly
tests and benchmarks.

### <a name="vlsi"></a> 3) Pushing a Rocket core through the VLSI tools

You can generate Verilog for your VLSI flow with the following commands:

    $ cd $ROCKETCHIP/vsim
    $ make verilog

Now take a look at vsim/generated-src, and the contents of the
Top.DefaultConfig.conf file:

    $ cd $ROCKETCHIP/vsim/generated-src
    DefaultConfig.dts
    DefaultConfig.graphml
    DefaultConfig.json
    DefaultConfig.memmap.json
    freechips.rocketchip.system.DefaultConfig.behav_srams.v
    freechips.rocketchip.system.DefaultConfig.conf
    freechips.rocketchip.system.DefaultConfig.d
    freechips.rocketchip.system.DefaultConfig.fir
    freechips.rocketchip.system.DefaultConfig.v
    $ cat $ROCKETCHIP/vsim/generated-src/*.conf
    name data_arrays_0_ext depth 512 width 256 ports mrw mask_gran 8
    name tag_array_ext depth 64 width 88 ports mrw mask_gran 22
    name tag_array_0_ext depth 64 width 84 ports mrw mask_gran 21
    name data_arrays_0_1_ext depth 512 width 128 ports mrw mask_gran 32
    name mem_ext depth 33554432 width 64 ports mwrite,read mask_gran 8
    name mem_2_ext depth 512 width 64 ports mwrite,read mask_gran 8

The conf file contains information for all SRAMs instantiated in the
flow. If you take a close look at the $ROCKETCHIP/Makefrag, you will see
that during Verilog generation, the build system calls a $(mem\_gen)
script with the generated configuration file as an argument, which will
fill in the Verilog for the SRAMs. Currently, the $(mem\_gen) script
points to vsim/vlsi\_mem\_gen, which simply instantiates behavioral
SRAMs.  You will see those SRAMs being appended at the end of
vsim/generated-src/Top.DefaultConfig.v. To target vendor-specific
SRAMs, you will need to make necessary changes to vsim/vlsi\_mem\_gen.

Similarly, if you have access to VCS, you can run assembly tests and
benchmarks with the following commands (again assuming you have N cores
on your host machine):

    $ cd $ROCKETCHIP/vsim
    $ make -jN run

The generated output looks similar to those generated from the emulator.
Look into vsim/output/\*.out for the output of the executed assembly
tests and benchmarks.

## <a name="param"></a> How can I parameterize my Rocket chip?

By now, you probably figured out that all generated files have a configuration
name attached, e.g. DefaultConfig. Take a look at
src/main/scala/rocketchip/Configs.scala. Search for NSets and NWays defined in
BaseConfig. You can change those numbers to get a Rocket core with different
cache parameters. For example, by changing L1I, NWays to 4, you will get
a 32KB 4-way set-associative L1 instruction cache rather than a 16KB 2-way
set-associative L1 instruction cache.

Further down, you will be able to see two FPGA configurations:
DefaultFPGAConfig and DefaultFPGASmallConfig. DefaultFPGAConfig inherits from
BaseConfig, but overrides the low-performance memory port (i.e., backup
memory port) to be turned off. This is because the high-performance memory
port is directly connected to the high-performance AXI interface on the ZYNQ
FPGA. DefaultFPGASmallConfig inherits from DefaultFPGAConfig, but changes the
cache sizes, disables the FPU, turns off the fast early-out multiplier and
divider, and reduces the number of TLB entries (all defined in SmallConfig).
This small configuration is used for the Zybo FPGA board, which has the
smallest ZYNQ part.

Towards the end, you can also find that DefaultSmallConfig inherits all
parameters from BaseConfig but overrides the same parameters of
WithNSmallCores.

Now take a look at vsim/Makefile. Search for the CONFIG variable.
By default, it is set to DefaultConfig.  You can also change the
CONFIG variable on the make command line:

    $ cd $ROCKETCHIP/vsim
    $ make -jN CONFIG=DefaultSmallConfig run-asm-tests

Or, even by defining CONFIG as an environment variable:

    $ export CONFIG=DefaultSmallConfig
    $ make -jN run-asm-tests

This parameterization is one of the many strengths of processor
generators written in Chisel, and will be more detailed in a future blog
post, so please stay tuned.

To override specific configuration items, such as the number of external interrupts,
you can create your own Configuration(s) and compose them with Config's ++ operator

    class WithNExtInterrupts(nExt: Int) extends Config {
        (site, here, up) => {
            case NExtInterrupts => nExt
        }
    }
    class MyConfig extends Config (new WithNExtInterrupts(16) ++ new DefaultSmallConfig)

Then you can build as usual with CONFIG=MyConfig.

## <a name="contributors"></a> Contributors

Can be found [here](https://github.com/ucb-bar/rocket-chip/graphs/contributors).

## <a name="attribution"></a> Attribution

If used for research, please cite Rocket Chip by the technical report:

Krste Asanović, Rimas Avižienis, Jonathan Bachrach, Scott Beamer, David Biancolin, Christopher Celio, Henry Cook, Palmer Dabbelt, John Hauser, Adam Izraelevitz, Sagar Karandikar, Benjamin Keller, Donggyu Kim, John Koenig, Yunsup Lee, Eric Love, Martin Maas, Albert Magyar, Howard Mao, Miquel Moreto, Albert Ou, David Patterson, Brian Richards, Colin Schmidt, Stephen Twigg, Huy Vo, and Andrew Waterman, _[The Rocket Chip Generator](http://www.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-17.html)_, Technical Report UCB/EECS-2016-17, EECS Department, University of California, Berkeley, April 2016
