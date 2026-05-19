# 易百纳 RK3588 开发板 Ubuntu 24.04 基础开发环境配置备忘

## 1. 文档目的

本文档用于记录易百纳（ebaina）定制 RK3588 开发板在 Ubuntu 24.04 系统下配置基础 C/C++ 开发环境的正确步骤。

配置完成后，开发板应具备以下能力：

- 使用 `gcc` 编译 C 程序；
- 使用 `g++` 编译 C++ 程序；
- 使用 `make` 构建 Makefile 工程；
- 使用 `gdb` 在板端调试 C/C++ 程序；
- 使用 `gdbserver` 支持远程调试；
- 使用 `cmake` 管理中大型 C/C++ 工程；
- 使用 `git` 管理和同步代码；
- 使用 `pkg-config` 获取第三方库的编译参数；
- 使用 `ninja` 加速 CMake 工程构建。

本文档仅记录已经验证可行的正确步骤，不记录中间排错过程。

---

## 2. 适用对象

本文档适用于如下开发板和系统环境：

| 项目 | 内容 |
|---|---|
| 开发板厂商 | 易百纳（ebaina） |
| 芯片平台 | Rockchip RK3588 |
| 板卡型号显示 | `rd-rk3588` |
| 系统版本 | Ubuntu 24.04.x LTS |
| CPU 架构 | `aarch64` / `arm64` |
| 包管理器 | `apt` / `apt-get` / `dpkg` |
| 默认源 | Ubuntu Ports，建议使用清华源 `ubuntu-ports` |
| 用户权限 | 建议使用 `root` 用户执行 |

---

## 3. 配置原则

易百纳定制系统中，大量系统包可能已经被 `apt-mark hold` 锁定。这通常是厂商为了固定内核、桌面环境、Rockchip 多媒体组件、GStreamer、MPP、RGA、OpenCV 等系统依赖版本。

因此，配置开发环境时应遵守以下原则：

1. **不要执行系统整体升级命令**；
2. **不要解除所有 hold 包**；
3. **只临时解锁 GCC 工具链相关的少量包**；
4. **安装完成后重新 hold 相关开发工具包**；
5. **每次安装新工具前，优先执行模拟安装，确认不会删除或降级系统包**。

严禁执行以下命令：

```bash
apt upgrade
apt dist-upgrade
apt full-upgrade
apt autoremove
apt-mark unhold '*'
apt-get --fix-broken install
apt-get install --allow-change-held-packages ...
```

---

## 4. 第一步：检查系统基本信息

执行以下命令，确认系统和板卡信息：

```bash
cat /etc/os-release
uname -m
cat /proc/device-tree/model 2>/dev/null | tr '\0' '\n'
which apt apt-get dpkg
df -h
free -h
```

正常情况下，应看到类似信息：

```text
Ubuntu 24.04.x LTS
aarch64
rd-rk3588
/usr/bin/apt
/usr/bin/apt-get
/usr/bin/dpkg
```

---

## 5. 第二步：查看 apt 软件源

Ubuntu 24.04 默认使用 deb822 格式的软件源配置文件。查看当前软件源：

```bash
cat /etc/apt/sources.list
cat /etc/apt/sources.list.d/ubuntu.sources
```

易百纳开发板上建议使用清华源，内容通常类似：

```text
Types: deb
URIs: http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

如果厂家已经配置好了源，不建议随意修改。

---

## 6. 第三步：备份当前 apt 状态

在修改 hold 状态和安装开发工具前，先备份当前状态：

```bash
mkdir -p /root/devtools-install-backup

apt-mark showhold > /root/devtools-install-backup/hold-before.txt

dpkg-query -W -f='${Package}\t${Version}\t${db:Status-Abbrev}\n' \
> /root/devtools-install-backup/dpkg-before.txt

cp -a /etc/apt/sources.list.d /root/devtools-install-backup/sources.list.d.before
```

说明：

- `hold-before.txt` 用于保存厂家原始 hold 包列表；
- `dpkg-before.txt` 用于保存安装前所有软件包版本；
- `sources.list.d.before` 用于保存软件源配置备份。

---

## 7. 第四步：更新 apt 索引

只更新软件包索引，不升级系统：

```bash
apt-get update
```

注意：`apt-get update` 只是更新本地软件包列表，不会安装或升级软件包。

---

## 8. 第五步：临时解锁 GCC 工具链相关包

由于易百纳出厂系统中部分 GCC 基础包可能被 hold，安装 `gcc/g++` 时可能出现依赖版本不一致。因此只临时解锁 GCC 工具链相关包：

```bash
apt-mark unhold \
gcc-13-base \
gcc-14-base \
cpp \
cpp-13 \
cpp-13-aarch64-linux-gnu \
cpp-aarch64-linux-gnu \
libgcc-s1 \
libstdc++6 \
libatomic1 \
libgomp1 \
libgfortran5
```

说明：

- 这里只解锁 GCC 工具链基础包；
- 不解锁 Rockchip 多媒体库、桌面环境、GStreamer、MPP、RGA、OpenCV 等厂家定制组件；
- 后续安装完成后会重新 hold 这些包。

---

## 9. 第六步：模拟安装 gcc、g++、make

正式安装前先模拟安装：

```bash
apt-get -s install --no-install-recommends \
gcc g++ make \
gcc-13 g++-13 \
gcc-13-aarch64-linux-gnu g++-13-aarch64-linux-gnu \
libgcc-13-dev libstdc++-13-dev \
gcc-13-base gcc-14-base \
cpp cpp-13 cpp-13-aarch64-linux-gnu cpp-aarch64-linux-gnu \
libgcc-s1 libstdc++6 libatomic1 libgomp1 libgfortran5
```

重点检查输出中是否出现以下危险信息：

```text
The following packages will be REMOVED
The following packages will be DOWNGRADED
```

如果没有删除和降级，只是少量 GCC 工具链相关包升级，以及新增编译器相关包，则可以继续正式安装。

---

## 10. 第七步：正式安装 gcc、g++、make

执行：

```bash
apt-get install -y --no-install-recommends \
gcc g++ make \
gcc-13 g++-13 \
gcc-13-aarch64-linux-gnu g++-13-aarch64-linux-gnu \
libgcc-13-dev libstdc++-13-dev \
gcc-13-base gcc-14-base \
cpp cpp-13 cpp-13-aarch64-linux-gnu cpp-aarch64-linux-gnu \
libgcc-s1 libstdc++6 libatomic1 libgomp1 libgfortran5
```

安装完成后验证：

```bash
which gcc g++ make
gcc --version
g++ --version
make --version
```

正常应看到：

```text
/usr/bin/gcc
/usr/bin/g++
/usr/bin/make
gcc 13.3.0
g++ 13.3.0
GNU Make 4.3
```

---

## 11. 第八步：安装 gdb

先模拟安装：

```bash
apt-get -s install --no-install-recommends gdb
```

确认无删除、无降级后，正式安装：

```bash
apt-get install -y --no-install-recommends gdb
```

安装完成后 hold：

```bash
apt-mark hold gdb
```

验证：

```bash
which gdb
gdb --version
```

---

## 12. 第九步：安装 CMake、Git、pkg-config

这三个工具是 SMT2.0 板端开发中非常常用的基础工具。

先模拟安装：

```bash
apt-get -s install --no-install-recommends cmake git pkg-config
```

确认无删除、无降级后，正式安装：

```bash
apt-get install -y --no-install-recommends cmake git pkg-config
```

安装完成后 hold：

```bash
apt-mark hold cmake git pkg-config
```

验证：

```bash
which cmake git pkg-config
cmake --version
git --version
pkg-config --version
```

---

## 13. 第十步：安装 gdbserver

如果后续需要在 PC 端使用 VS Code、CLion 或远程 gdb 调试开发板程序，建议安装 `gdbserver`。

先模拟安装：

```bash
apt-get -s install --no-install-recommends gdbserver
```

确认无删除、无降级后，正式安装：

```bash
apt-get install -y --no-install-recommends gdbserver
```

安装完成后 hold：

```bash
apt-mark hold gdbserver
```

验证：

```bash
which gdbserver
gdbserver --version
```

---

## 14. 第十一步：安装 ninja-build

`ninja-build` 是一种更快的构建工具，常用于 CMake 工程。

先模拟安装：

```bash
apt-get -s install --no-install-recommends ninja-build
```

确认无删除、无降级后，正式安装：

```bash
apt-get install -y --no-install-recommends ninja-build
```

安装完成后 hold：

```bash
apt-mark hold ninja-build
```

验证：

```bash
which ninja
ninja --version
```

---

## 15. 第十二步：重新 hold 工具链相关包

安装完成后，将新增和升级过的开发工具相关包重新 hold，防止后续误升级：

```bash
apt-mark hold \
gcc g++ make \
gcc-13 g++-13 \
gcc-13-aarch64-linux-gnu g++-13-aarch64-linux-gnu \
gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
libgcc-13-dev libstdc++-13-dev \
gcc-13-base gcc-14-base \
cpp cpp-13 cpp-13-aarch64-linux-gnu cpp-aarch64-linux-gnu \
libgcc-s1 libstdc++6 libatomic1 libgomp1 libgfortran5 \
binutils binutils-aarch64-linux-gnu binutils-common \
libasan8 libcc1-0 libhwasan0 libitm1 liblsan0 libtsan2 libubsan1 \
libbinutils libctf-nobfd0 libctf0 libgprofng0 libsframe1 \
gdb cmake git pkg-config gdbserver ninja-build
```

检查 hold 状态：

```bash
apt-mark showhold | grep -E "gcc|g\+\+|cpp|make|binutils|libgcc|libstdc|libatomic|libgomp|libgfortran|libasan|libcc1|libhwasan|libitm|liblsan|libtsan|libubsan|gdb|cmake|git|pkg-config|ninja"
```

检查 apt 依赖状态：

```bash
apt-get check
```

如果 `apt-get check` 没有报错，说明软件包依赖状态正常。

---

## 16. 第十三步：测试 C 程序编译

```bash
mkdir -p /root/dev_test
cd /root/dev_test

cat > hello.c << 'EOF'
#include <stdio.h>

int main(void) {
    printf("Hello RK3588 C development!\n");
    return 0;
}
EOF

gcc hello.c -o hello_c
./hello_c
file hello_c
```

正常输出：

```text
Hello RK3588 C development!
```

`file hello_c` 应显示为 ARM64 可执行文件：

```text
ELF 64-bit LSB executable, ARM aarch64
```

---

## 17. 第十四步：测试 C++ 程序编译

```bash
cd /root/dev_test

cat > hello.cpp << 'EOF'
#include <iostream>
#include <vector>
#include <string>

int main() {
    std::vector<std::string> words = {
        "Hello",
        "RK3588",
        "C++",
        "development!"
    };

    for (const auto& word : words) {
        std::cout << word << " ";
    }

    std::cout << std::endl;
    return 0;
}
EOF

g++ hello.cpp -o hello_cpp
./hello_cpp
file hello_cpp
```

正常输出：

```text
Hello RK3588 C++ development!
```

---

## 18. 第十五步：测试 gdb 调试

```bash
cd /root/dev_test

gcc -g hello.c -o hello_c_dbg

gdb -q ./hello_c_dbg
```

进入 gdb 后执行：

```gdb
break main
run
next
next
quit
```

如果退出时提示：

```text
A debugging session is active.
Quit anyway? (y or n)
```

输入：

```text
y
```

如果能够正常打断点、运行、单步执行，说明 gdb 调试环境正常。

---

## 19. 最终开发环境清单

配置完成后，开发板应具备如下工具：

| 工具 | 作用 |
|---|---|
| `gcc` | C 编译器 |
| `g++` | C++ 编译器 |
| `make` | Makefile 工程构建 |
| `gdb` | 板端调试 |
| `gdbserver` | 远程调试 |
| `cmake` | C/C++ 工程管理 |
| `git` | 代码版本管理 |
| `pkg-config` | 查询第三方库编译参数 |
| `ninja` | 快速构建工具 |

---

## 20. 一键配置脚本

下面脚本用于在新的易百纳（ebaina）RK3588 Ubuntu 24.04 开发板上一键配置基础开发环境。

使用方法：

```bash
nano /root/setup_ebaina_rk3588_dev_env.sh
```

将下面脚本内容复制进去，保存后执行：

```bash
chmod +x /root/setup_ebaina_rk3588_dev_env.sh
/root/setup_ebaina_rk3588_dev_env.sh
```

脚本内容如下：

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "============================================================"
echo " ebaina RK3588 Ubuntu 24.04 基础开发环境一键配置脚本"
echo "============================================================"

if [ "$(id -u)" -ne 0 ]; then
    echo "[ERROR] 请使用 root 用户执行该脚本。"
    exit 1
fi

echo "[INFO] 检查系统信息..."

if [ -f /etc/os-release ]; then
    . /etc/os-release
else
    echo "[ERROR] 未找到 /etc/os-release，无法判断系统版本。"
    exit 1
fi

ARCH="$(uname -m)"
BOARD_MODEL="$(cat /proc/device-tree/model 2>/dev/null | tr '\0' '\n' || true)"

echo "[INFO] 系统名称: ${PRETTY_NAME:-unknown}"
echo "[INFO] 系统代号: ${VERSION_CODENAME:-unknown}"
echo "[INFO] CPU架构: ${ARCH}"
echo "[INFO] 板卡型号: ${BOARD_MODEL:-unknown}"

if [ "${ID:-}" != "ubuntu" ]; then
    echo "[ERROR] 当前系统不是 Ubuntu，本脚本不继续执行。"
    exit 1
fi

if [ "${VERSION_CODENAME:-}" != "noble" ]; then
    echo "[ERROR] 当前系统不是 Ubuntu 24.04 noble，本脚本不继续执行。"
    exit 1
fi

if [ "${ARCH}" != "aarch64" ]; then
    echo "[ERROR] 当前架构不是 aarch64，本脚本不继续执行。"
    exit 1
fi

if ! command -v apt-get >/dev/null 2>&1; then
    echo "[ERROR] 未找到 apt-get，本脚本不继续执行。"
    exit 1
fi

BACKUP_DIR="/root/devtools-install-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "${BACKUP_DIR}"

echo "[INFO] 备份当前 apt hold 状态和软件包状态到: ${BACKUP_DIR}"

apt-mark showhold > "${BACKUP_DIR}/hold-before.txt" || true

dpkg-query -W -f='${Package}\t${Version}\t${db:Status-Abbrev}\n' \
> "${BACKUP_DIR}/dpkg-before.txt"

cp -a /etc/apt/sources.list.d "${BACKUP_DIR}/sources.list.d.before" 2>/dev/null || true
cp -a /etc/apt/sources.list "${BACKUP_DIR}/sources.list.before" 2>/dev/null || true

echo "[INFO] 更新 apt 软件包索引..."
apt-get update

echo "[INFO] 临时解除 GCC 工具链相关包的 hold 状态..."

apt-mark unhold \
gcc-13-base \
gcc-14-base \
cpp \
cpp-13 \
cpp-13-aarch64-linux-gnu \
cpp-aarch64-linux-gnu \
libgcc-s1 \
libstdc++6 \
libatomic1 \
libgomp1 \
libgfortran5 \
2>/dev/null || true

INSTALL_PKGS=(
    gcc
    g++
    make
    gcc-13
    g++-13
    gcc-13-aarch64-linux-gnu
    g++-13-aarch64-linux-gnu
    libgcc-13-dev
    libstdc++-13-dev
    gcc-13-base
    gcc-14-base
    cpp
    cpp-13
    cpp-13-aarch64-linux-gnu
    cpp-aarch64-linux-gnu
    libgcc-s1
    libstdc++6
    libatomic1
    libgomp1
    libgfortran5
    gdb
    cmake
    git
    pkg-config
    gdbserver
    ninja-build
)

echo "[INFO] 执行模拟安装，检查是否会删除或降级系统包..."

SIM_LOG="${BACKUP_DIR}/apt-simulate-install.log"

apt-get -s install --no-install-recommends "${INSTALL_PKGS[@]}" | tee "${SIM_LOG}"

if grep -E "The following packages will be REMOVED|The following packages will be DOWNGRADED" "${SIM_LOG}" >/dev/null 2>&1; then
    echo "[ERROR] 模拟安装结果显示存在删除或降级软件包的风险。"
    echo "[ERROR] 请查看模拟日志: ${SIM_LOG}"
    echo "[ERROR] 脚本已停止，未执行正式安装。"
    exit 1
fi

echo "[INFO] 模拟安装通过，开始正式安装基础开发工具..."

apt-get install -y --no-install-recommends "${INSTALL_PKGS[@]}"

echo "[INFO] 安装完成，重新 hold 开发工具和工具链相关包..."

apt-mark hold \
gcc g++ make \
gcc-13 g++-13 \
gcc-13-aarch64-linux-gnu g++-13-aarch64-linux-gnu \
gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
libgcc-13-dev libstdc++-13-dev \
gcc-13-base gcc-14-base \
cpp cpp-13 cpp-13-aarch64-linux-gnu cpp-aarch64-linux-gnu \
libgcc-s1 libstdc++6 libatomic1 libgomp1 libgfortran5 \
binutils binutils-aarch64-linux-gnu binutils-common \
libasan8 libcc1-0 libhwasan0 libitm1 liblsan0 libtsan2 libubsan1 \
libbinutils libctf-nobfd0 libctf0 libgprofng0 libsframe1 \
gdb cmake git pkg-config gdbserver ninja-build \
2>/dev/null || true

echo "[INFO] 检查 apt 依赖状态..."
apt-get check

echo "[INFO] 验证开发工具版本..."

which gcc g++ make gdb cmake git pkg-config gdbserver ninja

gcc --version | head -1
g++ --version | head -1
make --version | head -1
gdb --version | head -1
cmake --version | head -1
git --version
pkg-config --version
gdbserver --version | head -1 || true
ninja --version

echo "[INFO] 创建并编译 C/C++ 测试程序..."

TEST_DIR="/root/dev_test"
mkdir -p "${TEST_DIR}"
cd "${TEST_DIR}"

cat > hello.c << 'EOF'
#include <stdio.h>

int main(void) {
    printf("Hello RK3588 C development!\n");
    return 0;
}
EOF

gcc hello.c -o hello_c
./hello_c
file hello_c

cat > hello.cpp << 'EOF'
#include <iostream>
#include <vector>
#include <string>

int main() {
    std::vector<std::string> words = {
        "Hello",
        "RK3588",
        "C++",
        "development!"
    };

    for (const auto& word : words) {
        std::cout << word << " ";
    }

    std::cout << std::endl;
    return 0;
}
EOF

g++ hello.cpp -o hello_cpp
./hello_cpp
file hello_cpp

echo "============================================================"
echo " ebaina RK3588 基础开发环境配置完成"
echo "============================================================"
echo "[INFO] 备份目录: ${BACKUP_DIR}"
echo "[INFO] 测试目录: ${TEST_DIR}"
echo "[INFO] 后续请避免执行 apt upgrade / dist-upgrade / full-upgrade / autoremove。"
```

---

## 21. 后续注意事项

1. 后续如果需要安装新的开发库，应先执行模拟安装：

```bash
apt-get -s install --no-install-recommends 软件包名
```

2. 确认没有删除、降级、大量非预期升级后，再正式安装：

```bash
apt-get install -y --no-install-recommends 软件包名
```

3. 对于 SMT2.0 开发，后续如需安装 OpenCV、GStreamer、Rockchip MPP、RGA 相关开发库，应优先检查系统中是否已经由厂家预装，避免重复安装或升级厂家定制组件。

4. 不建议在该定制系统中随意安装大型桌面环境、Docker、Node.js、全量 `build-essential` 或未确认依赖关系的软件包。

5. 当前基础环境已经足够支撑 SMT2.0 的 C/C++ 程序开发、CMake 工程构建、板端调试和远程调试。
