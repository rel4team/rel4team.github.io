---
weight: 1
bookCollapseSection: false
bookFlatSection: true
title: "使用reL4运行相应的seL4测例"
---

# 使用reL4运行相应的seL4测例

以下是相关代码
```
# 首先在本地环境中下载相关仓库
# 如果只运行不进行push，下面repo init的链接可以改成https://github.com/reL4team2/rel4-dev-repo.git，否则您需要具有该仓库的权限
mkdir rel4_dev && cd rel4_dev
repo init -u git@github.com:rel4team2/rel4-dev-repo.git --repo-url=https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/
repo sync

# 拉取镜像并进入docker中（这里最后的点是上面定义的文件夹名的路径）
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/rel4team2/build-scripts/refs/heads/master/scripts/start_docker.sh | bash -s -- .
# 或者其实可以第二种方式构建（遇到curl的网络问题可以使用）
cat ./build-scripts/scripts/start_docker.sh | bash -s -- .
docker exec -u ${USER} -it rel4_dev bash

# 在docker中运行sel4-test测试
# -p参数指定platform，目前可选为spike（riscv64）和qemu-arm-virt（aarch64）
# -m参数指定MCS特性（混合关键系统）开启（on）与否（off）
# -s参数指定是否使用SMC特性开启（on）与否（off）
# --arm_ptmr=on参数开启arm平台下的KernelArmExportPCNTUser，默认关闭
# --arm_pcnt=on参数开启arm平台下的KernelArmExportPTMRUser，默认关闭
# 另外，home的版本有些问题，需要改回去

cd rel4_kernel
cargo update home@0.5.11 --precise 0.5.5
./build.py -p spike -m off -s on&& cd build && ./simulate && cd ..

# 如果需要进行提交，exit命令离开docker，进入对应的文件夹进行提交
```