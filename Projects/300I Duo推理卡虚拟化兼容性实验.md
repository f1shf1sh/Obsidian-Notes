步骤：
  1. 裸机基线
     先把单卡、双卡、多卡、温度、功耗、吞吐、时延基线跑出来。没有这一步，后面 KVM/Kata 的结果都没法判断。
  2. KVM 单卡整卡直通
     先测一张卡直通 1 台 VM，只做兼容性和性能回归。
  3. KVM 多卡
     再测 1 VM -> 多卡，以及 多 VM -> 每 VM 1 卡。
  4. vNPU
     先在宿主机容器里验证切分，再做 vNPU -> VM。
  5. Kata
     只在前面都稳定后再做，而且先测整卡，不要一上来就测 vNPU + Kata。
# 裸机
## 裸机单卡
```sh
uname -a
cat /etc/os-release

lspci -nn | grep -i d500
npu-smi info
npu-smi info -l
npu-smi info -m

virsh --version
docker --version 2>/dev/null || true
containerd --version 2>/dev/null || true
kata-runtime --version 2>/dev/null || true

python3 -m ais_bench --help >/dev/null 2>&1 && echo ais_bench_ok || echo ais_bench_missing
ascend-dmi -h >/dev/null 2>&1 && echo ascend_dmi_ok || echo ascend_dmi_missing
```
 
# KVM + QEMU

## 虚拟化核心组件安装
```sh
# 1. 安装QEMU组件
yum install -y qemu

# 2. 安装libvirt组件
yum install -y libvirt

# 3. 启动libvirtd服务
systemctl start libvirtd
```
# Kata Container