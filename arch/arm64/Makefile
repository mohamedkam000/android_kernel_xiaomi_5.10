LDFLAGS_vmlinux	:=--no-undefined -X --pic-veneer

ifeq ($(CONFIG_RELOCATABLE), y)
LDFLAGS_vmlinux		+= -shared -Bsymbolic -z notext \
			$(call ld-option, --no-apply-dynamic-relocs)
endif

ifeq ($(CONFIG_ARM64_ERRATUM_843419),y)
  ifeq ($(call ld-option, --fix-cortex-a53-843419),)
$(warning ld does not support --fix-cortex-a53-843419; kernel may be susceptible to erratum)
  else
LDFLAGS_vmlinux	+= --fix-cortex-a53-843419
  endif
endif

ifeq ($(CONFIG_ARM64_USE_LSE_ATOMICS), y)
  ifneq ($(CONFIG_ARM64_LSE_ATOMICS), y)
$(warning LSE atomics not supported by binutils)
  endif
endif

cc_has_k_constraint := $(call try-run,echo				\
	'int main(void) {						\
		asm volatile("and w0, w0, %w0" :: "K" (4294967295));	\
		return 0;						\
	}' | $(CC) -S -x c -o "$$TMP" -,,-DCONFIG_CC_HAS_K_CONSTRAINT=1)

ifeq ($(CONFIG_BROKEN_GAS_INST),y)
$(warning Detected assembler with broken .inst; disassembly will be unreliable)
endif

KBUILD_CFLAGS	+= -mgeneral-regs-only	\
		   $(compat_vdso) $(cc_has_k_constraint)
KBUILD_CFLAGS	+= $(call cc-disable-warning, psabi)
KBUILD_AFLAGS	+= $(compat_vdso)

KBUILD_CFLAGS	+= $(call cc-option,-mabi=lp64)
KBUILD_AFLAGS	+= $(call cc-option,-mabi=lp64)

# Avoid generating .eh_frame* sections.
KBUILD_CFLAGS	+= -fno-asynchronous-unwind-tables -fno-unwind-tables
KBUILD_AFLAGS	+= -fno-asynchronous-unwind-tables -fno-unwind-tables

ifeq ($(CONFIG_STACKPROTECTOR_PER_TASK),y)
prepare: stack_protector_prepare
stack_protector_prepare: prepare0
	$(eval KBUILD_CFLAGS += -mstack-protector-guard=sysreg		  \
				-mstack-protector-guard-reg=sp_el0	  \
				-mstack-protector-guard-offset=$(shell	  \
			awk '{if ($$2 == "TSK_STACK_CANARY") print $$3;}' \
					include/generated/asm-offsets.h))
endif

# Ensure that if the compiler supports branch protection we default it
# off, this will be overridden if we are using branch protection.
branch-prot-flags-y += $(call cc-option,-mbranch-protection=none)

ifeq ($(CONFIG_ARM64_PTR_AUTH),y)
branch-prot-flags-$(CONFIG_CC_HAS_SIGN_RETURN_ADDRESS) := -msign-return-address=all
# We enable additional protection for leaf functions as there is some
# narrow potential for ROP protection benefits and no substantial
# performance impact has been observed.
ifeq ($(CONFIG_ARM64_BTI_KERNEL),y)
branch-prot-flags-$(CONFIG_CC_HAS_BRANCH_PROT_PAC_RET_BTI) := -mbranch-protection=pac-ret+leaf+bti
else
branch-prot-flags-$(CONFIG_CC_HAS_BRANCH_PROT_PAC_RET) := -mbranch-protection=pac-ret+leaf
endif
# -march=armv8.3-a enables the non-nops instructions for PAC, to avoid the
# compiler to generate them and consequently to break the single image contract
# we pass it only to the assembler. This option is utilized only in case of non
# integrated assemblers.
ifeq ($(CONFIG_AS_HAS_PAC), y)
asm-arch := armv8.3-a
endif
endif

KBUILD_CFLAGS += $(branch-prot-flags-y)

ifeq ($(CONFIG_AS_HAS_ARMV8_4), y)
# make sure to pass the newest target architecture to -march.
asm-arch := armv8.4-a
endif

ifeq ($(CONFIG_AS_HAS_ARMV8_5), y)
# make sure to pass the newest target architecture to -march.
asm-arch := armv8.5-a
endif

ifdef asm-arch
KBUILD_CFLAGS	+= -Wa,-march=$(asm-arch) \
		   -DARM64_ASM_ARCH='"$(asm-arch)"'
endif

ifeq ($(CONFIG_SHADOW_CALL_STACK), y)
KBUILD_CFLAGS	+= -ffixed-x18
endif

ifeq ($(CONFIG_CPU_BIG_ENDIAN), y)
KBUILD_CPPFLAGS	+= -mbig-endian
CHECKFLAGS	+= -D__AARCH64EB__
# Prefer the baremetal ELF build target, but not all toolchains include
# it so fall back to the standard linux version if needed.
KBUILD_LDFLAGS	+= -EB $(call ld-option, -maarch64elfb, -maarch64linuxb -z norelro)
UTS_MACHINE	:= aarch64_be
else
KBUILD_CPPFLAGS	+= -mlittle-endian
CHECKFLAGS	+= -D__AARCH64EL__
# Same as above, prefer ELF but fall back to linux target if needed.
KBUILD_LDFLAGS	+= -EL $(call ld-option, -maarch64elf, -maarch64linux -z norelro)
UTS_MACHINE	:= aarch64
endif

ifeq ($(CONFIG_LD_IS_LLD), y)
KBUILD_LDFLAGS	+= -z norelro
endif

CHECKFLAGS	+= -D__aarch64__

ifeq ($(CONFIG_DYNAMIC_FTRACE_WITH_REGS),y)
  KBUILD_CPPFLAGS += -DCC_USING_PATCHABLE_FUNCTION_ENTRY
  CC_FLAGS_FTRACE := -fpatchable-function-entry=2
endif

# Default value
head-y		:= arch/arm64/kernel/head.o

ifeq ($(CONFIG_KASAN_SW_TAGS), y)
KASAN_SHADOW_SCALE_SHIFT := 4
else ifeq ($(CONFIG_KASAN_GENERIC), y)
KASAN_SHADOW_SCALE_SHIFT := 3
endif

KBUILD_CFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)
KBUILD_CPPFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)
KBUILD_AFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)

core-y		+= arch/arm64/
libs-y		:= arch/arm64/lib/ $(libs-y)
libs-$(CONFIG_EFI_STUB) += $(objtree)/drivers/firmware/efi/libstub/lib.a

# Default target when executing plain make
boot		:= arch/arm64/boot
KBUILD_IMAGE	:= $(boot)/Image

# Don't compile Image in mixed build with "all" target
ifndef KBUILD_MIXED_TREE
all:	Image
endif


Image: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

Image.%: Image
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

zinstall install:
	$(Q)$(MAKE) $(build)=$(boot) $@

PHONY += vdso_install
vdso_install:
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso $@
	$(if $(CONFIG_COMPAT_VDSO), \
		$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso32 $@)

# We use MRPROPER_FILES and CLEAN_FILES now
archclean:
	$(Q)$(MAKE) $(clean)=$(boot)

ifeq ($(KBUILD_EXTMOD),)
prepare: vdso_prepare
vdso_prepare: prepare0
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso include/generated/vdso-offsets.h
	$(if $(CONFIG_COMPAT_VDSO),$(Q)$(MAKE) \
		$(build)=arch/arm64/kernel/vdso32  \
		include/generated/vdso32-offsets.h)
endif
