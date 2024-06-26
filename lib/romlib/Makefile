#
# Copyright (c) 2018-2024, Arm Limited and Contributors. All rights reserved.
#
# SPDX-License-Identifier: BSD-3-Clause
#

toolchains := aarch64

include ../../make_helpers/toolchain.mk

ROMLIB_GEN  = ./romlib_generator.py
BUILD_DIR   = $(BUILD_PLAT)/romlib
LIB_DIR     = $(BUILD_PLAT)/lib
WRAPPER_DIR = $(BUILD_PLAT)/libwrapper
LIBS        = -lmbedtls -lfdt -lc
INC         = $(INCLUDES:-I%=-I../../%)
PPFLAGS     = $(INC) $(DEFINES) -P -x assembler-with-cpp -D__LINKER__ -MD -MP -MT $(BUILD_DIR)/romlib.ld
OBJS        = $(BUILD_DIR)/jmptbl.o $(BUILD_DIR)/init.o
MAPFILE     = $(BUILD_PLAT)/romlib/romlib.map

ifneq ($(PLAT_DIR),)
  WRAPPER_SOURCES   = $(shell $(ROMLIB_GEN) genwrappers -b $(WRAPPER_DIR) --list ../../$(PLAT_DIR)/jmptbl.i)
  WRAPPER_OBJS      = $(WRAPPER_SOURCES:.s=.o)
endif

V ?= 0
ifeq ($(V),0)
  Q := @
else
  Q :=
endif

LDFLAGS := -Wl,--gc-sections -nostdlib

ifeq ($(DEBUG),1)
   LDFLAGS += -Wl,-Map=$(MAPFILE)
endif

ifeq (${ARM_ARCH_MINOR},0)
	ASFLAGS = -march=armv8-a
else
	ASFLAGS = -march=armv8.${ARM_ARCH_MINOR}-a
endif

.PHONY: all clean distclean

all: $(BUILD_DIR)/romlib.bin $(LIB_DIR)/libwrappers.a

%.o: %.s
	@echo "  AS      $@"
	$(Q)$(aarch64-as) -c $(ASFLAGS) -o $@ $<

$(BUILD_DIR)/%.o: %.s
	@echo "  AS      $@"
	$(Q)$(aarch64-as) -c $(ASFLAGS) -o $@ $<

$(BUILD_DIR)/romlib.ld: romlib.ld.S
	@echo "  PP      $@"
	$(Q)$(aarch64-cpp) -E $(PPFLAGS) -o $@ romlib.ld.S

$(BUILD_DIR)/romlib.elf: $(OBJS) $(BUILD_DIR)/romlib.ld
	@echo "  LD      $@"
	$(Q)$(aarch64-ld) -T $(BUILD_DIR)/romlib.ld -L$(LIB_DIR) $(LDFLAGS) -o $@ $(OBJS) $(LIBS)

$(BUILD_DIR)/romlib.bin: $(BUILD_DIR)/romlib.elf
	@echo "  BIN     $@"
	$(Q)$(aarch64-oc) -O binary $(BUILD_DIR)/romlib.elf $@

$(WRAPPER_DIR)/jmpvar.s: $(BUILD_DIR)/romlib.elf
	@echo "  VAR     $@"
	$(Q)$(ROMLIB_GEN) genvar --output $@ $<

$(LIB_DIR)/libwrappers.a: $(WRAPPER_DIR)/jmpvar.o $(WRAPPER_OBJS)
	@echo "  AR      $@"
	$(Q)$(aarch64-ar) -rc $@ $(WRAPPER_DIR)/jmpvar.o $(WRAPPER_OBJS)

$(BUILD_DIR)/jmptbl.i: ../../$(PLAT_DIR)/jmptbl.i
	@echo "  PRE     $@"
	$(Q)$(ROMLIB_GEN) pre --output $@ --deps $(BUILD_DIR)/jmptbl.d $<

$(BUILD_DIR)/wrappers.stamp: $(BUILD_DIR)/jmptbl.i
	@echo "  WRP     $<"
	$(Q)$(ROMLIB_GEN) genwrappers --bti=$(ENABLE_BTI) -b $(WRAPPER_DIR) $<
	@touch $@

$(WRAPPER_SOURCES): $(BUILD_DIR)/wrappers.stamp

$(WRAPPER_OBJS): $(WRAPPER_SOURCES) $(BUILD_DIR)/wrappers.stamp

$(BUILD_DIR)/jmptbl.s: $(BUILD_DIR)/jmptbl.i
	@echo "  TBL     $@"
	$(Q)$(ROMLIB_GEN) gentbl --output $@ --bti=$(ENABLE_BTI) $<

clean:
	@rm -f $(BUILD_DIR)/*

-include $(BUILD_DIR)/romlib.d
-include $(BUILD_DIR)/jmptbl.d
