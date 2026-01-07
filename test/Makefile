# #######################################################################
#
#   BSD LICENSE
#
#   Copyright(c) 2007-2025 Intel Corporation. All rights reserved.
#   All rights reserved.
#
#   Redistribution and use in source and binary forms, with or without
#   modification, are permitted provided that the following conditions
#   are met:
#
#     * Redistributions of source code must retain the above copyright
#       notice, this list of conditions and the following disclaimer.
#     * Redistributions in binary form must reproduce the above copyright
#       notice, this list of conditions and the following disclaimer in
#       the documentation and/or other materials provided with the
#       distribution.
#     * Neither the name of Intel Corporation nor the names of its
#       contributors may be used to endorse or promote products derived
#       from this software without specific prior written permission.
#
#   THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
#   "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
#   LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
#   A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
#   OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
#   SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
#   LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
#   DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
#   THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
#   (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
#   OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#
# #######################################################################
BUILD_QAT ?= 1
BUILD_AOCL ?= 0

# Validate that BUILD_QAT and BUILD_AOCL are mutually exclusive, but skip for clean-like goals
ifneq ($(filter clean uninstall, $(MAKECMDGOALS)),)
# skip validation during clean/uninstall
else
ifeq ($(BUILD_QAT), 1)
ifeq ($(BUILD_AOCL), 1)
$(error BUILD_QAT and BUILD_AOCL cannot both be enabled. Use one of: BUILD_QAT=1 BUILD_AOCL=0 (QAT), BUILD_QAT=0 BUILD_AOCL=0 (pure ZSTD), or BUILD_QAT=0 BUILD_AOCL=1 (AOCL))
endif
endif
endif

ifeq ($(BUILD_QAT), 1)
	LIB     = ../src

	LDFLAGS = $(LIB)/libqatseqprod.a -I$(LIB)

	# Expose QAT-dependent code paths and CLI options (e.g. -m)
	CFLAGS  += -DBUILD_QAT

	ifneq ($(ICP_ROOT), )
		LDFLAGS += -lqat_s -lusdm_drv_s -lnuma -Wl,-rpath,$(ICP_ROOT)/build -L$(ICP_ROOT)/build
	else
		LDFLAGS += -lqat -lusdm -lnuma
	endif
endif

# AMD AOCL compression library support
ifeq ($(BUILD_AOCL), 1)
	CFLAGS += -DBUILD_AOCL

	# Default AOCL installation path
	AOCL_ROOT ?= ../../aocl-compression/install_path

	# Ensure -L (search path) precedes -l (library) for correct resolution
	LDFLAGS += -L$(AOCL_ROOT)/lib -laocl_compression
	CFLAGS += -I$(AOCL_ROOT)/include
endif

ifdef ZSTDLIB
CFLAGS += -I$(ZSTDLIB)
LDFLAGS += $(ZSTDLIB)/libzstd.a
else
ZSTDLIB := $(shell find /usr /lib /local -name 'libzstd.a' 2>/dev/null | head -n 1| xargs dirname)
LDFLAGS += $(ZSTDLIB)/libzstd.a
endif

default: test benchmark

all: test benchmark

# Optional arguments for running benchmark, e.g., ARGS="-m 1024"
ARGS ?=

check_zstdlib:
	@echo checking zstd static libaray '$(ZSTDLIB)/libzstd.a'
	@if [ ! -e $(ZSTDLIB)/libzstd.a ]; then \
	  echo  libzstd.a is not found, please install libzstd or specify the path manually; \
	  exit 1; \
	fi

.PHONY: check_aocl

check_aocl:
ifeq ($(BUILD_AOCL), 1)
	@echo "checking AOCL library at '$(AOCL_ROOT)/lib' (cwd: $(CURDIR))"
	@if [ ! -d $(AOCL_ROOT) ]; then \
	  echo AOCL_ROOT not found: $(AOCL_ROOT); \
	  echo Please set AOCL_ROOT to your AOCL install path; \
	  exit 1; \
	fi
	@if [ ! -e $(AOCL_ROOT)/lib/libaocl_compression.so ] && [ ! -e $(AOCL_ROOT)/lib/libaocl_compression.a ]; then \
	  echo libaocl_compression not found in $(AOCL_ROOT)/lib; \
	  echo Ensure AOCL is installed and library path is correct; \
	  exit 1; \
	fi
else
	@true
endif

test: test.c check_zstdlib check_aocl
ifeq ($(BUILD_QAT), 1)
	$(Q)$(MAKE) -C $(LIB)
endif
	$(CC) $< $(CFLAGS) $(LDFLAGS) -o $@

benchmark: benchmark.c check_zstdlib check_aocl
ifeq ($(BUILD_QAT), 1)
	$(Q)$(MAKE) -C $(LIB)
endif
	$(CC) $(CFLAGS) $< $(LDFLAGS) -o $@ -lpthread

clean:
	$(RM) test benchmark
.PHONY: clean check_zstdlib



# Run the compiled benchmark with optional ARGS (e.g., -m 1024)
.PHONY: run-benchmark
run-benchmark: benchmark
	./benchmark $(ARGS)
