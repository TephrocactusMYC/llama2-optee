### llama2-optee 运行记录

使用环境： Ubuntu 20.04 虚拟机

1. 配置最新版的optee qemu_v8环境，下载交叉编译工具链，不要编译整个项目
```
# 绝对路径为/home/jian/optee-elf
mkdir optee-elf
cd optee-elf
repo init -u https://github.com/OP-TEE/manifest.git -m qemu_v8.xml -b 4.2.0
repo sync
cd build
make toolchains
```

2. 克隆llama2-optee仓库
```
# 绝对路径为/home/jian/llama2-optee
git clone --recurse-submodules https://github.com/TephrocactusMYC/llama2-optee.git
```

3. 配置musl-libc的使用optee-elf的toolchains交叉编译，安装修改后的musl-libc
```
cd ./llama2-optee/elf-musl-libc
./configure --target=aarch64 --prefix=/musl CROSS_COMPILE=/home/jian/optee-elf/toolchains/aarch64/bin/aarch64-linux-gnu-
# 编译并安装
make
make install
```

4. 修改llama2.c中run.c文件并编译
```
cd /home/jian/llama2-optee/llama2.c
vim run.c
# 修改tokenizer路径
char *tokenizer_path = "/root/tokenizer.bin"; // 根据REE侧的文件路径调节
# 编译run.c
/musl/bin/musl-gcc ./run.c -O3 -fpie -pie -o run
```
注：这里遇到一个报错，根据提示向run.c中添加<stdint.h>头文件后解决
```
/run.c:452:39: error: unknown type name ‘int8_t’
  452 | void encode(Tokenizer* t, char *text, int8_t bos, int8_t eos, int *tokens, int *n_tokens) {
      |                                       ^~~~~~
./run.c:15:1: note: ‘int8_t’ is defined in header ‘<stdint.h>’; did you forget to ‘#include <stdint.h>’?
   14 |     #include <sys/mman.h>
  +++ |+#include <stdint.h>
   15 | #endif
```

5. 修改qemu安全内存大小
(1) 首先修改 `optee-elf/qemu/hw/arm/virt.c`，调整安全内存的位置，将大小调整为128MB
```
[VIRT_SECURE_MEM] =         { 0x40000000, 0x08000000 }, // 128MB
[VIRT_MEM] =                { 0x48000000, LEGACY_RAMLIMIT_BYTES - 0x08000000 },
```
(2) 修改 `optee-elf/u-boot/board/emulation/qemu-arm/qemu-arm.c`将内存范围扩大到0x48000000
```
.virt = 0x48000000UL,
.phys = 0x48000000UL,
```
(3) 修改 `optee-elf/u-boot/board/emulation/qemu-arm/qemu-arm.env`，将对应地址增加0x08000000
```
fdt_addr=0x48000000
scriptaddr=0x48200000
pxefile_addr_r=0x48300000
kernel_addr_r=0x48400000
ramdisk_addr_r=0x4C000000
```
(4) 修改 `optee-elf/u-boot/configs/qemu_arm64_defconfig`，将对应地址增加0x08000000
```
CONFIG_CUSTOM_SYS_INIT_SP_ADDR=0x48200000
CONFIG_SYS_LOAD_ADDR=0x48200000
```
(5) 修改 `optee-elf/u-boot/configs/qemu_arm_defconfig`，将对应地址增加0x08000000
```
CONFIG_CUSTOM_SYS_INIT_SP_ADDR=0x48200000
CONFIG_SYS_LOAD_ADDR=0x48200000
```
(6) 修改 `optee-elf/u-boot/include/configs/qemu-arm.h`将内存范围扩大到0x48000000
```
#define CFG_SYS_SDRAM_BASE		0x48000000
```
(7) 修改 `optee-elf/trusted-firmware-a/plat/qemu/qemu/include/platform_def.h`
```
#define NS_DRAM0_BASE			ULL(0x48000000)
#define NS_DRAM0_SIZE			ULL(0xB8000000)
#define SEC_SRAM_BASE			0x40000000
#define SEC_SRAM_SIZE			0x00100000
#define SEC_DRAM_BASE			0x40100000
#define SEC_DRAM_SIZE			0x07f00000
```

6. 用修改后的opteeXX替换原有目录，编译修改后的工程
```
cd /home/jian/optee-elf
# 删除原有文件
rm -rf optee_client/ optee_os/ optee_test/ optee_build/
# 复制修改后的目录
cp -r /home/jian/llama2-optee/elf-optee-client/ ./optee_client
cp -r /home/jian/llama2-optee/elf-optee-os/ ./optee_os
cp -r /home/jian/llama2-optee/elf-optee-test/ ./optee_test
cp -r /home/jian/llama2-optee/elf-build/ ./build
# 将elf-ta放入/optee_examples
cp -r /home/jian/llama2-optee/elf-ta ./optee_examples
# 编译
cd build
make run -f qemu_v8.mk
# 报错后重新运行 make run-only -f qemu_v8.mk
```

7. 把llama2.c运行所需的参数、词汇表以及编译得到的run.elf文件放入`optee-elf/out-br/target/root`下
```
# 获取参数
wget https://huggingface.co/karpathy/tinyllamas/resolve/main/stories15M.bin
mv ./stories15M.bin ./out-br/target/root
cp /home/jian/llama2-optee/llama2.c/tokenizer.bin ./out-br/target/root
```

8. 启动optee环境后，运行elf-ta
```
elf_ta_loader ./run /root/stories15M.bin
```