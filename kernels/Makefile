# ============================================================
# GPGPU Kernel Makefile
# 自动发现 *.S，编译为 *.bin，并产出 *.dump 便于检查
# ============================================================

CROSS    := riscv64-unknown-elf
CC       := $(CROSS)-gcc
OBJCOPY  := $(CROSS)-objcopy
OBJDUMP  := $(CROSS)-objdump

# 统一编译选项：RV32 + FP + zicsr/zifencei + M 扩展
# 注：GCC 12+ 才把 zicsr/zifencei 从 I 拆出来单写；GCC 10 直接写 rv32imf 即可
ARCH     := rv32imf
ABI      := ilp32f
CFLAGS   := -march=$(ARCH) -mabi=$(ABI) -nostdlib -Wl,-Ttext=0x0

# 自动发现所有 .S 源文件
SRCS     := $(wildcard *.S)
ELFS     := $(SRCS:.S=.elf)
BINS     := $(SRCS:.S=.bin)
DUMPS    := $(SRCS:.S=.dump)

# 默认目标：编译所有 kernel + dump
.PHONY: all
all: $(BINS) $(DUMPS)

# 单个 kernel 的编译规则
%.elf: %.S
	$(CC) $(CFLAGS) -o $@ $<

%.bin: %.elf
	$(OBJCOPY) -O binary $< $@

%.dump: %.elf
	$(OBJDUMP) -d $< > $@

# 清理
.PHONY: clean
clean:
	rm -f *.elf *.bin *.dump

# 列出当前会编译哪些 kernel
.PHONY: list
list:
	@echo "Source files:  $(SRCS)"
	@echo "Binaries:      $(BINS)"
	@echo "Dumps:         $(DUMPS)"

# 单独构建某个 kernel：make NAME.bin 或 make NAME.dump
# 由 pattern rule 自动支持，无需显式列出

.PHONY: deploy
deploy:
	scp -P 12055 *.bin root@localhost:/tmp/