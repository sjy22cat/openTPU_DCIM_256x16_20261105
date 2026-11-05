# DCIM 256×16 服务器功能验证

工程根目录为`openTPU_256x16/openTPU`。可在Linux服务器用Verilator编译RTL并运行功能仿真，
无需FPGA、GPU、Vivado或DDR控制器IP。系统测试以行为存储模型连接真实的2048 bit MC接口，
验证loader、SEQ、DMA、ACT、DCIM/MXU、VPU、TMEM和QUANT。

产品RTL已完成此前的XSim回归；本次只调整测试参考转换、测试向量路径和运行工具，
三个修改后的系统测试平台均通过XSim语法编译，未重新运行完整回归。
本机没有可用的Verilator，以下入口尚未取得服务器实际运行结果。
不同Verilator版本的编译诊断应以服务器日志为准，不将准备好脚本视作验证通过。

## 1. 服务器环境

- Linux；Verilator 5.x，需支持`--binary`及`--timing`，建议使用较新的5.x版本。
- 支持C++20 coroutine的g++或clang++、GNU make；例如g++ 11及以上。
- Python建议3.11及以上，安装numpy、pytest；Python3.10另需tomli。
- 建议64 GiB或更多内存，预留数十GiB编译空间。这是起始资源建议，尚未测量本配置的Verilator峰值。
- 初次使用`--jobs 4`，各测试串行执行；内存不足时降为1或2，避免沿用原通用入口的`-j 0`。

在服务器项目根目录执行：

```bash
verilator --version
g++ --version
python3 -m venv .venv
source .venv/bin/activate
python -m pip install numpy pytest
export OTPU_TARGET=dcim256
python tools/check_dcim256_verilator.py --check-env
```

使用clang++时设置`CXX=clang++`。发行版预装的Verilator 4.x不适用此入口。
`--check-env`检查工具是否存在及Verilator主版本，实际C++20与RTL兼容性由编译步骤检查。

## 2. 打包与传输

在Windows项目根目录执行：

```powershell
python -X utf8 tools/package_dcim256.py
```

生成`build/dcim256_verilator_source.tar.gz`及同名`.sha256`文件。压缩包直接读取当前工作文件，
包含未提交及新增的RTL、target、软件、专用测试平台与运行脚本；不含`.git`、本机build、
模型权重或Vivado中间文件。不要只用`git archive HEAD`或上游仓库下载替代此包，这会漏掉当前改造。

用实际服务器账号和新建的接收目录替换下列占位符：

```powershell
scp build/dcim256_verilator_source.tar.gz build/dcim256_verilator_source.tar.gz.sha256 USER@HOST:/REMOTE/NEW_DIR/
```

服务器在该接收目录执行：

```bash
sha256sum -c dcim256_verilator_source.tar.gz.sha256
tar -xzf dcim256_verilator_source.tar.gz
cd openTPU
```

包内`SOURCE_MANIFEST.json`记录逐文件SHA256。应解压到独立目录，保留原服务器工程。
此包用于专用功能回归；没有打包上游全部测试、主机驱动或模型文件。

## 3. 运行顺序

先验证内存及握手，再编译系统。各测试单独生成可执行文件：

```bash
python tools/check_dcim256_verilator.py --top tb_tmem_wide tb_dma_wide tb_mc2048 tb_qst_collect --jobs 4
python tools/check_dcim256_verilator.py --top tb_dcim256 --jobs 4
python tools/check_dcim256_verilator.py --top tb_dcim256_vectors --jobs 4
python tools/check_dcim256_verilator.py --top tb_dcim256_sys tb_dcim256_control --jobs 4
```

也可一次运行：

```bash
python tools/check_dcim256_verilator.py --top all --jobs 4
```

| 测试顶层 | 检查内容 |
| --- | --- |
| tb_tmem_wide | 双池、bank冲突、读收集、写公平、返回保持和读副本一致性 |
| tb_dma_wide | 非对齐、掩码、缓冲回绕、跨池、TMEM反压及WAITW |
| tb_mc2048 | 异步请求返回、400笔并发事务、credit和公平调度 |
| tb_qst_collect | INT8写合并、转置、bank及槽冲突重放、byte mask及flush |
| tb_dcim256 | 最小系统链路，MM后接VPU并通过DMA写外存 |
| tb_dcim256_vectors | 自动生成393条指令及ISA参考，逐word比较4 MiB存储内容 |
| tb_dcim256_sys | M16/N256、KB=1/2/10/16数值与推进计数，覆盖-128权重 |
| tb_dcim256_control | 跨池GATHER、BAR、RLD及CHAIN重装载 |

支持`--lint-only`、`--compile-only`及`--run-only`，三者互斥。
`--run-only`要求源码和Verilator版本与构建记录一致；否则拒绝复用。
`--timeout 3600`为每个仿真的墙钟时限，C++构建可能更久；`--dry-run`只打印构建命令。
编译日志中的warning需检查；脚本未屏蔽RTL error，`-Wno-fatal`仅允许warning继续编译。

两个定向测试的FP32参考已改用独立的整数IEEE754编码，避免Verilator将shortreal提升为real。
此编码在XSim与原生shortreal转换对比37232项通过，覆盖当前测试数值及舍入边界；
`tb_dcim256_ref`仅用于XSim检查该参考函数，不属于Verilator测试集合。
产品RTL不需要为此替换运算逻辑。393指令测试仍使用Python ISA模拟器产生独立参考。

## 4. 结果与解释

每个测试需要进程成功退出且出现`DCIM256_TEST_PASS`。
`LINT_PASS`仅表示静态检查通过，`BUILD_PASS`仅表示编译成功，均不等于功能验证通过。

日志、源码指纹与汇总位于：

```text
build/verilator_dcim256/<TOP>/build.log
build/verilator_dcim256/<TOP>/run.log
build/verilator_dcim256/<TOP>/sources.json
build/verilator_dcim256/results.json
```

`results.json`记录最近一次命令选择的测试；分多条命令运行时，各测试日志仍保留。
数值测试的输入及参考在服务器自动生成，不依赖Windows已有build内容。
可补充运行：

```bash
python -m pytest --noconftest tests/test_compiler.py tests/test_waits.py::test_mx_qst_scale_rows -q
python tools/check_dcim256_model.py
python tools/audit_dcim256_qwen3.py
```

上述后两项分别是小型随机Qwen3结构的ISA执行和Qwen3-4B分配/工作量核算，均不是4B完整模型RTL推理。
Verilator通常采用二值模型，XSim的四值未知态检查仍有补充价值。
Verilator通过后仍需在选定器件上完成综合、时序与板级持续带宽验证。
