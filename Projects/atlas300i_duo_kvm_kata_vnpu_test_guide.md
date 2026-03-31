# Atlas 300I Duo 在 KVM / vNPU / Kata 下的兼容性、可靠性与性能测试手册

## 1. 文档目的

这份手册用于指导你在 `Atlas 800 (Model 3000) + Atlas 300I Duo` 环境下，逐步完成以下验证：

1. `KVM` 单卡直通的兼容性、可靠性和性能
2. `KVM` 多卡直通的兼容性、可靠性和性能
3. `vNPU` 单卡切分的兼容性、可靠性和性能
4. `Kata` 安全容器的实验性验证

本手册的原则是：

- 先做裸机基线，再做虚拟化
- 先做官方明确支持的路径，再做实验项
- 每一步都要留日志和结果，避免“感觉变快/变慢”

---

## 2. 先说结论和边界

### 2.1 你现在已经完成的事情

你已经完成：

- `driver`
- `NPU firmware`
- `MCU`

并且已经观察到风扇转速恢复正常。这说明硬件底层使能已经基本就绪。

### 2.2 官方支持边界

当前测试要遵守下面这些边界：

- `KVM/libvirt` 的 `NPU 直通虚拟机` 是官方主线支持场景。
- `Atlas 300I Duo` 支持 `vNPU`。
- `Atlas 300I Duo` 的每个芯片最多支持切分为 `7` 路 `vNPU`。
- `Atlas 300I Duo` 的 2 个芯片工作模式需要保持一致，不要一边整卡、一边切分。
- `vNPU -> 虚拟机` 官方支持的是 `静态虚拟化`。
- 在虚拟机里安装的是 `NPU 驱动`，不是 `firmware`。
- 我没有查到昇腾官方明确写 `Kata Containers` 为受支持主线，因此 `Kata` 本手册按“实验性验证”处理，不作为第一阶段验收项。

### 2.3 术语说明

- `整卡`：一张物理 `Atlas 300I Duo` 卡
- `单芯片`：`300I Duo` 卡上的单个 `310` 系列芯片
- `vNPU`：昇腾虚拟化实例，不叫 `vGPU`
- `KVM 单卡`：把一个 PCIe NPU 设备直通给一个虚拟机
- `KVM 多卡`：把多个 PCIe NPU 设备直通给一个虚拟机，或者分给多个虚拟机

---

## 3. 整体测试路线图

严格按下面顺序走，不要跳步：

1. 固化环境版本和日志目录
2. 安装测试工具
3. 做裸机健康检查
4. 做裸机单卡性能基线
5. 做裸机多卡性能基线
6. 做 `KVM` 单卡直通
7. 做 `KVM` 多卡直通
8. 做 `vNPU` 容器模式
9. 做 `vNPU -> VM` 静态虚拟化
10. 最后再做 `Kata` 实验项

---

## 4. 测试前准备

### 4.1 建立统一工作目录

下面所有日志和结果，统一放到一个目录里。

```bash
export TEST_ROOT=/root/atlas300i_duo_virt_test
mkdir -p ${TEST_ROOT}/{env,logs,models,input,baseline,kvm_single,kvm_multi,vnpu,kata,report}
date | tee ${TEST_ROOT}/env/test_start_time.txt
```

### 4.2 固化当前系统和驱动信息

这一组命令的目的，是把“测试时到底是什么环境”完整记录下来。

```bash
uname -a | tee ${TEST_ROOT}/env/uname.txt
cat /etc/os-release | tee ${TEST_ROOT}/env/os-release.txt
lspci -nn | grep -i d500 | tee ${TEST_ROOT}/env/lspci_d500.txt
npu-smi info | tee ${TEST_ROOT}/env/npu-smi-info.txt
npu-smi info -l | tee ${TEST_ROOT}/env/npu-smi-info-l.txt
npu-smi info -m | tee ${TEST_ROOT}/env/npu-smi-info-m.txt
/usr/local/Ascend/driver/tools/upgrade-tool --device_index -1 --component -1 --version | tee ${TEST_ROOT}/env/upgrade-tool-version.txt
```

### 4.3 记录你需要长期保存的关键字段

每次测试报告都至少记录：

- 主机型号
- BIOS 版本
- iBMC 版本
- OS 版本
- 内核版本
- 驱动版本
- NPU 固件版本
- MCU 版本
- CANN 版本
- 测试模型名
- OM 文件名
- batch size
- 使用设备号
- 虚拟化模式
- 测试开始时间
- 测试结束时间

---

## 5. 安装测试工具

这一阶段的目标是把“能测性能和能测健康”的工具装起来。

### 5.1 需要的工具

你至少需要这三类：

- `CANN Toolkit`
- `310P ops`
- `ToolBox`

对应常见包名如下：

- `Ascend-cann-toolkit_8.5.0_linux-aarch64.run`
- `Ascend-cann-310p-ops_8.5.0_linux-aarch64.run`
- `Ascend-mindx-toolbox_<version>_linux-aarch64.run`

### 5.2 安装 CANN 和 ToolBox

如果这几个包已经下载到 `/root/cann85`，直接执行：

```bash
mkdir -p /root/cann85
cd /root/cann85

chmod +x Ascend-cann-toolkit_8.5.0_linux-aarch64.run
./Ascend-cann-toolkit_8.5.0_linux-aarch64.run --install

chmod +x Ascend-cann-310p-ops_8.5.0_linux-aarch64.run
./Ascend-cann-310p-ops_8.5.0_linux-aarch64.run --install

chmod +x Ascend-mindx-toolbox_*.run
./Ascend-mindx-toolbox_*.run --install
```

### 5.3 设置环境变量

```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/toolbox/set_env.sh
```

如果 `toolbox/set_env.sh` 因安全检查失败而报错，可以临时手动加路径：

```bash
export PATH=$PATH:/usr/local/Ascend/toolbox/latest/Ascend-DMI/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/dcmi
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/Ascend/driver/lib64
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/Ascend/driver/lib64/driver
```

### 5.4 安装 ais_bench

`ais_bench` 用于跑推理吞吐和时延。官方说明它依赖：

- 已安装 `toolkit` 或 `nnrt`
- 已安装 `Python3`
- 需要安装 `aclruntime` 和 `ais_bench`

如果你已经下载了 `whl` 包，推荐直接安装：

```bash
export CANN_PATH=/usr/local/Ascend/ascend-toolkit/latest
python3 -m pip install --upgrade pip wheel setuptools
python3 -m pip install ./aclruntime-*.whl
python3 -m pip install ./ais_bench-*.whl
```

如果你还没有 `whl` 包，就按官方 `ais_bench` 安装指南获取并安装，先不要跳过这一步。

### 5.5 验证工具是否可用

```bash
python3 -m ais_bench --help | tee ${TEST_ROOT}/env/ais_bench_help.txt
ascend-dmi -h | tee ${TEST_ROOT}/env/ascend_dmi_help.txt
virsh --version | tee ${TEST_ROOT}/env/virsh_version.txt
docker --version | tee ${TEST_ROOT}/env/docker_version.txt
containerd --version | tee ${TEST_ROOT}/env/containerd_version.txt
kata-runtime --version | tee ${TEST_ROOT}/env/kata_version.txt
```

记录：

- 哪些工具已经可用
- 哪些工具还没有安装
- `kata-runtime` 是否存在

---

## 6. 准备统一的测试模型和输入

### 6.1 为什么必须统一模型和输入

如果你每次测试的模型、输入数据、batch size 不一样，最终结果没有可比性。

所以请固定：

- 同一个 `.om` 模型
- 同一个输入目录
- 同一个 batch size
- 同一个 loop 次数

### 6.2 建议先用一个简单基线模型

建议第一轮测试先用：

- `ResNet50 OM`

原因：

- 官方有性能测试参考
- 输入简单
- 结果容易比对

### 6.3 定义统一变量

把自己的路径改进来：

```bash
export MODEL_OM=${TEST_ROOT}/models/resnet50_bs1.om
export INPUT_DIR=${TEST_ROOT}/input
export OUTPUT_DIR=${TEST_ROOT}/report
export LOOP=200
export WARMUP=10
export BS=1
```

如果你没有现成输入文件，也可以先用纯推理随机输入做冒烟：

```bash
python3 -m ais_bench --model ${MODEL_OM} --device 0 --loop ${LOOP} --warmup_count ${WARMUP} --batchsize ${BS} --pure_data_type random
```

---

## 7. 第 1 阶段：裸机健康检查

这一阶段的目标是先确认“硬件健康、驱动正常、工具能跑”，不要一上来就做 KVM。

### 7.1 设备识别检查

```bash
lspci -nn | grep -i d500 | tee ${TEST_ROOT}/baseline/lspci_d500.txt
npu-smi info | tee ${TEST_ROOT}/baseline/npu_smi_info.txt
npu-smi info -l | tee ${TEST_ROOT}/baseline/npu_smi_info_l.txt
npu-smi info -m | tee ${TEST_ROOT}/baseline/npu_smi_info_m.txt
```

你要确认：

- 所有预期的 `d500` 都能看到
- `npu-smi info` 不报错
- `NPU ID` 数量和预期一致
- 芯片状态正常

### 7.2 版本兼容性检查

```bash
ascend-dmi -c | tee ${TEST_ROOT}/baseline/ascend_dmi_compat.txt
```

你要记录：

- 是否通过版本兼容性检查
- 是否有驱动、固件、ToolBox、CANN 的版本不匹配提示

### 7.3 裸机健康诊断

如果你想做更完整的硬件自检，可以补一轮诊断：

```bash
ascend-dmi --dg --device 0 | tee ${TEST_ROOT}/baseline/ascend_dmi_diag_dev0.txt
```

如果有多个设备，可逐个执行：

```bash
ascend-dmi --dg --device 1 | tee ${TEST_ROOT}/baseline/ascend_dmi_diag_dev1.txt
ascend-dmi --dg --device 2 | tee ${TEST_ROOT}/baseline/ascend_dmi_diag_dev2.txt
ascend-dmi --dg --device 3 | tee ${TEST_ROOT}/baseline/ascend_dmi_diag_dev3.txt
```

你要记录：

- 是否有诊断失败项
- 是否有温度、功耗、链路、内存相关告警

---

## 8. 第 2 阶段：裸机单卡性能基线

这一阶段的目标是建立“后续所有虚拟化场景都要对比的基线”。

### 8.1 选择一张卡

先用 `NPU ID 0` 做基线。

### 8.2 跑单卡基线

有输入文件时：

```bash
python3 -m ais_bench \
  --model ${MODEL_OM} \
  --input ${INPUT_DIR} \
  --device 0 \
  --loop ${LOOP} \
  --warmup_count ${WARMUP} \
  --batchsize ${BS} \
  --display_all_summary true \
  --output ${TEST_ROOT}/baseline \
  --output_dirname bm_single_dev0 | tee ${TEST_ROOT}/baseline/bm_single_dev0_console.log
```

没有输入文件时：

```bash
python3 -m ais_bench \
  --model ${MODEL_OM} \
  --device 0 \
  --loop ${LOOP} \
  --warmup_count ${WARMUP} \
  --batchsize ${BS} \
  --pure_data_type random \
  --display_all_summary true \
  --output ${TEST_ROOT}/baseline \
  --output_dirname bm_single_dev0 | tee ${TEST_ROOT}/baseline/bm_single_dev0_console.log
```

### 8.3 同步记录设备状态

建议在另一个终端里同时采样：

```bash
watch -n 2 npu-smi info
```

或者定时落盘：

```bash
for i in $(seq 1 60); do
  date >> ${TEST_ROOT}/baseline/bm_single_dev0_npu_smi.log
  npu-smi info >> ${TEST_ROOT}/baseline/bm_single_dev0_npu_smi.log
  sleep 5
done
```

### 8.4 这一阶段必须记录什么

- 平均吞吐
- 平均时延
- 执行是否报错
- 温度是否异常
- 功耗是否异常
- 运行过程中是否掉卡

这一组数据以后是所有 `KVM`、`vNPU`、`Kata` 的对照基线。

---

## 9. 第 3 阶段：裸机多卡性能基线

这一阶段要回答两个问题：
1. 多卡能不能同时稳定跑
2. 总吞吐是否接近线性增长

### 9.1 方法一：分别启动多个进程

这是最容易看清每张卡表现的方式。

```bash
mkdir -p ${TEST_ROOT}/baseline/multi

nohup python3 -m ais_bench --model ${MODEL_OM} --device 0 --loop ${LOOP} --warmup_count ${WARMUP} --batchsize ${BS} --pure_data_type random --output ${TEST_ROOT}/baseline/multi --output_dirname dev0 > ${TEST_ROOT}/baseline/multi/dev0.log 2>&1 &
nohup python3 -m ais_bench --model ${MODEL_OM} --device 1 --loop ${LOOP} --warmup_count ${WARMUP} --batchsize ${BS} --pure_data_type random --output ${TEST_ROOT}/baseline/multi --output_dirname dev1 > ${TEST_ROOT}/baseline/multi/dev1.log 2>&1 &
nohup python3 -m ais_bench --model ${MODEL_OM} --device 2 --loop ${LOOP} --warmup_count ${WARMUP} --batchsize ${BS} --pure_data_type random --output ${TEST_ROOT}/baseline/multi --output_dirname dev2 > ${TEST_ROOT}/baseline/multi/dev2.log 2>&1 &
nohup python3 -m ais_bench --model ${MODEL_OM} --device 3 --loop ${LOOP} --warmup_count ${WARMUP} --batchsize ${BS} --pure_data_type random --output ${TEST_ROOT}/baseline/multi --output_dirname dev3 > ${TEST_ROOT}/baseline/multi/dev3.log 2>&1 &
wait
```

### 9.2 方法二：一个命令指定多个设备

```bash
python3 -m ais_bench \
  --model ${MODEL_OM} \
  --device 0,1,2,3 \
  --loop ${LOOP} \
  --warmup_count ${WARMUP} \
  --batchsize ${BS} \
  --pure_data_type random \
  --display_all_summary true \
  --output ${TEST_ROOT}/baseline \
  --output_dirname bm_multi | tee ${TEST_ROOT}/baseline/bm_multi_console.log
```

### 9.3 这一阶段要记录什么

- 每张卡单独吞吐
- 多卡总吞吐
- 各卡是否有明显偏差
- 是否有某张卡温度特别高
- 是否出现掉卡、报错、hang 住

---

## 10. 第 4 阶段：KVM 单卡直通

这一阶段的目标是回答：

- 一张 NPU 直通给一个 VM，能否稳定工作
- 性能相对裸机回退多少

### 10.1 先检查 KVM 基础能力

```bash
virsh --version | tee ${TEST_ROOT}/kvm_single/virsh_version.txt
virt-host-validate | tee ${TEST_ROOT}/kvm_single/virt_host_validate.txt
dmesg | grep -Ei 'iommu|smmu|vfio' | tee ${TEST_ROOT}/kvm_single/iommu_check.txt
```

### 10.2 选定要直通的设备

```bash
lspci -nn | grep -i d500 | tee ${TEST_ROOT}/kvm_single/host_d500_before.txt
```

假设你选中的 BDF 是：

- `0000:3b:00.0`

### 10.3 从宿主机上解绑并交给 libvirt

把 BDF 转成 `virsh` 节点名：

- `0000:3b:00.0` 对应 `pci_0000_3b_00_0`

执行：

```bash
virsh nodedev-dumpxml pci_0000_3b_00_0 | tee ${TEST_ROOT}/kvm_single/pci_0000_3b_00_0.xml
virsh nodedev-detach pci_0000_3b_00_0
```

### 10.4 生成 hostdev XML

```bash
cat > ${TEST_ROOT}/kvm_single/npu_3b00.xml <<'EOF'
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x3b' slot='0x00' function='0x0'/>
  </source>
</hostdev>
EOF
```

### 10.5 挂给虚拟机

假设虚拟机名字叫 `oevm1`：

```bash
virsh attach-device oevm1 ${TEST_ROOT}/kvm_single/npu_3b00.xml --config --live
virsh dumpxml oevm1 | tee ${TEST_ROOT}/kvm_single/oevm1_dumpxml_after_attach.xml
```

### 10.6 在虚拟机内做的事情

进入虚拟机后：

1. 安装 `NPU 驱动`
2. 不安装 `firmware`
3. 安装 `Toolkit + 310p-ops`
4. 安装 `ais_bench`

### 10.7 在虚拟机内验证识别

```bash
lspci -nn | grep -i d500
npu-smi info
npu-smi info -l
```

### 10.8 在虚拟机内跑同样的单卡基线

命令必须尽量和裸机一致：

```bash
python3 -m ais_bench \
  --model ${MODEL_OM} \
  --device 0 \
  --loop ${LOOP} \
  --warmup_count ${WARMUP} \
  --batchsize ${BS} \
  --pure_data_type random \
  --display_all_summary true \
  --output /root/vmtest \
  --output_dirname vm_single_dev0 | tee /root/vmtest/vm_single_dev0_console.log
```

### 10.9 单卡 KVM 的通过标准建议

建议按工程口径判断：

- 功能通过：`lspci`、`npu-smi`、`ais_bench` 都正常
- 稳定性通过：连续执行 `10` 次不掉卡、不报错
- 性能通过：吞吐达到裸机单卡的 `95%` 以上

---

## 11. 第 5 阶段：KVM 多卡直通

这一阶段分两条线：

1. `1 VM -> 多卡`
2. `多 VM -> 每 VM 1 卡`

### 11.1 路线 A：1 个虚拟机挂 2 张或更多卡

重复上面的 `nodedev-detach + attach-device`，把多个 BDF 挂给同一个 VM。

然后在 VM 内验证：

```bash
lspci -nn | grep -i d500
npu-smi info
npu-smi info -l
```

再执行多卡基线：

```bash
python3 -m ais_bench \
  --model ${MODEL_OM} \
  --device 0,1 \
  --loop ${LOOP} \
  --warmup_count ${WARMUP} \
  --batchsize ${BS} \
  --pure_data_type random \
  --display_all_summary true \
  --output /root/vmtest \
  --output_dirname vm_multi_2dev | tee /root/vmtest/vm_multi_2dev_console.log
```

### 11.2 路线 B：2 个虚拟机各挂 1 张卡

示例：

- `oevm1` 挂 `0000:3b:00.0`
- `oevm2` 挂 `0000:86:00.0`

然后分别在两个 VM 内同时跑相同基线。

### 11.3 这一阶段要记录什么

- 每个 VM 的吞吐
- 多 VM 总吞吐
- 是否出现卡间不均衡
- 是否出现某个 VM 稳定、另一个 VM 不稳定

---

## 12. 第 6 阶段：vNPU 容器模式

这一阶段的目标是验证：

- 单个芯片切分后能否稳定工作
- 切分后总吞吐和单 slice 吞吐表现如何

### 12.1 重要约束

先记住这几个硬约束：

- 切分前，确保目标芯片没有被 VM 或宿主机业务占用
- 同一卡上的两个芯片应保持一致模式
- 切分后，不要再把同一芯片当整卡使用

### 12.2 查看设备和芯片信息

```bash
npu-smi info -l | tee ${TEST_ROOT}/vnpu/npu_info_l.txt
npu-smi info -m | tee ${TEST_ROOT}/vnpu/npu_info_m.txt
```

### 12.3 设置 vNPU 模式

容器切分模式：

```bash
npu-smi set -t vnpu-mode -d 0 | tee ${TEST_ROOT}/vnpu/set_vnpu_mode_container.txt
```

如果你的版本已经在目标模式，或者返回类似“当前状态已满足”，记录下来即可。

### 12.4 创建 vNPU

示例：在设备 `1`、芯片 `0` 上创建一个 `vir02`

```bash
npu-smi set -t create-vnpu -i 1 -c 0 -f vir02 | tee ${TEST_ROOT}/vnpu/create_vnpu_dev1_chip0_vir02.txt
```

如果要在同一芯片上继续切：

```bash
npu-smi set -t create-vnpu -i 1 -c 0 -f vir02 | tee -a ${TEST_ROOT}/vnpu/create_vnpu_dev1_chip0_vir02.txt
npu-smi set -t create-vnpu -i 1 -c 0 -f vir02_1c | tee -a ${TEST_ROOT}/vnpu/create_vnpu_dev1_chip0_vir02_1c.txt
```

查询切分结果：

```bash
npu-smi info -t info-vnpu -i 1 -c 0 | tee ${TEST_ROOT}/vnpu/info_vnpu_dev1_chip0.txt
```

### 12.5 建议优先测试的切分组合

建议优先从官方推荐的组合开始：

- `vir04 + vir04_3c`
- `vir04 + vir02 + vir02_1c`

不要一开始就测最碎的切法。

### 12.6 在容器里验证 vNPU

先用普通容器路径，而不是 Kata。

示例：

```bash
docker run --rm -it \
  --name ascend-vnpu-test \
  -e ASCEND_VISIBLE_DEVICES=0 \
  ubuntu:22.04 /bin/bash
```

容器里完成：

- `lspci` 或设备节点检查
- `npu-smi info`
- `ais_bench`

### 12.7 跑 vNPU 性能测试

保持和裸机同样的模型、同样的 loop、同样的 batch：

```bash
python3 -m ais_bench \
  --model ${MODEL_OM} \
  --device 0 \
  --loop ${LOOP} \
  --warmup_count ${WARMUP} \
  --batchsize ${BS} \
  --pure_data_type random \
  --display_all_summary true \
  --output ${TEST_ROOT}/vnpu \
  --output_dirname vnpu_single | tee ${TEST_ROOT}/vnpu/vnpu_single_console.log
```

### 12.8 vNPU 的重点记录项

- 每个 vNPU 的吞吐
- 切分后总吞吐
- 切分后总吞吐与整卡吞吐的差值
- 单个 vNPU 是否容易报错
- 多个 vNPU 并发是否稳定

---

## 13. 第 7 阶段：vNPU -> 虚拟机

这是官方支持的静态虚拟化路径。

### 13.1 切到虚拟机模式

```bash
npu-smi set -t vnpu-mode -d 1 | tee ${TEST_ROOT}/vnpu/set_vnpu_mode_vm.txt
```

### 13.2 重新创建 vNPU

示例：

```bash
npu-smi set -t create-vnpu -i 1 -c 0 -f vir02 | tee ${TEST_ROOT}/vnpu/create_vnpu_vm_dev1_chip0_vir02.txt
npu-smi info -t info-vnpu -i 1 -c 0 | tee ${TEST_ROOT}/vnpu/info_vnpu_vm_dev1_chip0.txt
```

### 13.3 把 vNPU 交给虚拟机

这一段建议严格按当前版本对应的官方 `Atlas 系列硬件产品 虚拟机配置指南` 操作。

你要验证的是：

- VM 内是否能看到 vNPU
- VM 内安装 vNPU 驱动后是否能正常 `npu-smi`
- `ais_bench` 是否能稳定跑通

### 13.4 这一阶段的重点

这个阶段主要看：

- 切分资源能否被 VM 正确识别
- 切分资源在 VM 内是否稳定
- 性能相比宿主机 vNPU 场景是否还有明显回退

---

## 14. 第 8 阶段：Kata 安全容器实验项

这一阶段不是第一轮主线，不要提前做。

### 14.1 为什么要放最后

原因很简单：

- 昇腾官方主线文档里，我没有查到 `Kata Containers` 的明确支持路径
- `Kata` 本身是轻量虚拟机容器
- 你现在同时在测 `NPU 直通 / vNPU / 容器 runtime / 安全容器`
- 如果一上来就做 `Kata`，一旦失败，你无法判断问题出在 NPU、KVM、vNPU、容器还是 Kata

### 14.2 Kata 的进入条件

只有当下面 4 项全部通过时，才开始 Kata：

- 裸机单卡通过
- 裸机多卡通过
- KVM 单卡通过
- vNPU 容器模式通过

### 14.3 Kata 这一阶段只验证什么

先只验证：

- 容器能否正常启动
- NPU 设备能否正确注入
- `npu-smi` 能否在容器内正常工作
- 一个最小 `ais_bench` 冒烟测试能否跑通

不要一上来就做长稳压测。

### 14.4 Kata 的建议测试顺序

1. 先在普通容器里验证设备注入
2. 再在同一台 VM 里安装 `kata-runtime`
3. 先跑不带 NPU 的 CPU 容器，确认 Kata 本身没问题
4. 再尝试带 NPU 设备的最小容器
5. 最后才做 `ais_bench`

### 14.5 Kata 最小检查项

```bash
kata-runtime --version
ctr version
docker version
```

记录：

- Kata 版本
- containerd 版本
- NPU 是否能被注入 Kata 容器
- `npu-smi` 是否工作

---

## 15. 可靠性测试怎么做

可靠性不要只跑一次。

### 15.1 最小可靠性回归建议

每个场景都做下面三类：

1. 重复启动测试
2. 长时间循环测试
3. 重启恢复测试

### 15.2 重复启动测试

目标：验证反复创建/销毁、反复启动是否稳定。

示例：

```bash
for i in $(seq 1 10); do
  echo "round=${i}" | tee -a ${TEST_ROOT}/report/repeat_start.log
  python3 -m ais_bench --model ${MODEL_OM} --device 0 --loop 20 --warmup_count 5 --batchsize ${BS} --pure_data_type random >> ${TEST_ROOT}/report/repeat_start.log 2>&1
done
```

记录：

- 第几轮失败
- 是否出现超时
- 是否出现 `npu-smi` 无法识别

### 15.3 长时间循环测试

目标：验证长稳。

示例：

```bash
for i in $(seq 1 100); do
  date >> ${TEST_ROOT}/report/long_run.log
  python3 -m ais_bench --model ${MODEL_OM} --device 0 --loop 50 --warmup_count 5 --batchsize ${BS} --pure_data_type random >> ${TEST_ROOT}/report/long_run.log 2>&1 || break
done
```

记录：

- 总运行时长
- 累计推理次数
- 第几轮失败
- 失败前后 `npu-smi info`

### 15.4 重启恢复测试

目标：验证宿主机或 VM 重启后，设备是否还能恢复。

每个场景至少做：

1. 测试前记录版本和状态
2. 重启
3. 再次运行 `npu-smi info`
4. 再跑一次最小 `ais_bench`

---

## 16. 性能结果如何判断

### 16.1 建议的对比方式

以“裸机单卡基线”为 `100%`，其他场景都和它比。

计算方式：

```text
性能保持率 = 场景吞吐 / 裸机单卡吞吐 * 100%
```

### 16.2 建议的工程判定线

以下不是官方承诺值，是工程上比较实用的经验口径：

- `KVM 单卡`：性能保持率 `>= 95%`
- `KVM 多卡`：总吞吐接近线性增长，达到理论累加值的 `>= 90%`
- `vNPU`：关注切分后总吞吐与单实例抖动，不要求和整卡绝对持平
- `Kata`：先看能否稳定跑通，再看性能

---

## 17. 建议使用的结果记录表

你可以把下面这个表复制到测试报告里。

| 测试编号 | 场景 | 设备配置 | 模型 | Batch | Loop | 是否通过 | 平均吞吐 | 平均时延 | 异常现象 | 日志路径 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| BM-01 | 裸机单卡 | dev0 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| BM-02 | 裸机多卡 | dev0,1,2,3 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| VM-01 | KVM 单卡 | vm1->dev0 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| VM-02 | KVM 多卡 | vm1->dev0,1 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| VM-03 | 多VM分卡 | vm1->dev0 vm2->dev1 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| VNPU-01 | vNPU 容器 | vir02 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| VNPU-02 | vNPU->VM | vir02->vm | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| KATA-01 | Kata 整卡 | kata+dev0 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |
| KATA-02 | Kata vNPU | kata+vir02 | resnet50_bs1.om | 1 | 200 |  |  |  |  |  |

---

## 18. 出现问题时怎么定位

### 18.1 先判断属于哪一层

按这个顺序判断：

1. 硬件层
2. 驱动层
3. CANN 运行时层
4. 虚拟化层
5. 容器层
6. Kata 层

### 18.2 每层先查什么

硬件层：

```bash
lspci -nn | grep -i d500
npu-smi info
```

驱动层：

```bash
lsmod | grep -i ascend
npu-smi info
```

CANN 层：

```bash
env | grep -i ASCEND
python3 -m ais_bench --help
```

虚拟化层：

```bash
virsh dumpxml <vm_name>
virsh nodedev-list | grep pci_
```

容器层：

```bash
docker ps -a
ctr containers list
```

Kata 层：

```bash
kata-runtime --version
journalctl -u containerd --no-pager | tail -n 200
```

---

## 19. 官方资料入口

下面这些文档是本手册的主要依据：

- Atlas 300I Duo 基本规格  
  <https://info.support.huawei.com/enterprise/zh/doc/EDOC1100245754/c0338d05>

- Atlas 300I Duo 驱动和固件说明  
  <https://www.hiascend.com/doc_center/source/zh/Atlas%20300I%20Duo%20Inference%20Card/Atlas%20300I%20Duo%20Inference%20Card/apn_006.html>

- Atlas 300I Duo `ais_bench` 工具安装  
  <https://www.hiascend.com/doc_center/source/zh/Atlas%20300I%20Duo%20Inference%20Card/Atlas%20300I%20Duo%20Inference%20Card/apn_031.html>

- Atlas 300I Duo `ais_bench` 命令说明  
  <https://www.hiascend.com/doc_center/source/zh/Atlas%20300I%20Duo%20Inference%20Card/Atlas%20300I%20Duo%20Inference%20Card/apn_032.html>

- Atlas 300I Duo `ResNet50` 性能测试用例  
  <https://www.hiascend.com/doc_center/source/zh/Atlas%20300I%20Duo%20Inference%20Card/Atlas%20300I%20Duo%20Inference%20Card/apn_058.html>

- Atlas 系列硬件产品 25.2.0 虚拟机配置指南  
  <https://info.support.huawei.com/enterprise/zh/doc/EDOC1100494679/c887b2ca>

- 昇腾虚拟化实例支持场景  
  <https://www.hiascend.com/document/detail/zh/mindcluster/600/clustersched/usage/aviug/cpaug_0010.html>

- 创建 vNPU  
  <https://www.hiascend.com/document/detail/zh/mindcluster/600/clustersched/usage/aviug/cpaug_0011.html>

- ToolBox 安装  
  <https://www.hiascend.com/document/detail/zh/mindx-dl/500/toolbox/toolboxug/toolboxug_0004.html>

- Ascend DMI 环境要求  
  <https://www.hiascend.com/document/detail/zh/mindx-dl/300/dluserguide/toolboxug/toolboxug_0004.html>

---

## 20. 我建议你下一步怎么做

不要立刻做 `KVM`。

先按本手册只做下面 3 件事：

1. 安装 `Toolkit + 310p-ops + ToolBox + ais_bench`
2. 完成“第 1 阶段裸机健康检查”
3. 完成“第 2 阶段裸机单卡性能基线”

只要这 3 步没完成，后面所有 `KVM/vNPU/Kata` 的结论都不可信。
