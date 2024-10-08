#
# aflgo - LLVM instrumentation
# ----------------------------
#
# Copyright 2015, 2016 Google LLC All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at:
#
#   http://www.apache.org/licenses/LICENSE-2.0
#

VERSION     = $(shell grep '^\#define VERSION ' ../afl-2.57b/config.h | cut -d '"' -f2)

LLVM_CONFIG ?= llvm-config

CFLAGS      ?= -O3 -funroll-loops
CFLAGS      += -Wall -D_FORTIFY_SOURCE=2 -g -Wno-pointer-sign -DVERSION=\"$(VERSION)\" 

CXXFLAGS    ?= -O3 -funroll-loops
CXXFLAGS    += -Wall -D_FORTIFY_SOURCE=2 -g -Wno-pointer-sign \
               -DVERSION=\"$(VERSION)\" -Wno-variadic-macros

ifdef AFLGO_TRACING
  CFLAGS    += -DAFLGO_TRACING=1
  CXXFLAGS  += -DAFLGO_TRACING=1
endif

# Mark nodelete to work around unload bug in upstream LLVM 5.0+
CLANG_CFL    = `$(LLVM_CONFIG) --cxxflags` -Wl,-znodelete -fno-rtti -fpic $(CXXFLAGS)
CLANG_LFL    = `$(LLVM_CONFIG) --ldflags` $(LDFLAGS)

# User teor2345 reports that this is required to make things work on MacOS X.

ifeq "$(shell uname)" "Darwin"
  CLANG_LFL += -Wl,-flat_namespace -Wl,-undefined,suppress
endif

# We were using llvm-config --bindir to get the location of clang, but
# this seems to be busted on some distros, so using the one in $PATH is
# probably better.

ifeq "$(origin CC)" "default"
  CC         = clang
  CXX        = clang++
endif

PROGS = aflgo-clang aflgo-pass.so aflgo-runtime.o aflgo-runtime-32.o aflgo-runtime-64.o

all: test_deps $(PROGS) test_build all_done

test_deps:
	@echo "[*] Checking for working 'llvm-config'..."
	@which $(LLVM_CONFIG) >/dev/null 2>&1 || ( echo "[-] Oops, can't find 'llvm-config'. Install clang or set \$$LLVM_CONFIG or \$$PATH beforehand."; echo "    (Sometimes, the binary will be named llvm-config-3.5 or something like that.)"; exit 1 )
	@echo "[*] Checking for working '$(CC)'..."
	@which $(CC) >/dev/null 2>&1 || ( echo "[-] Oops, can't find '$(CC)'. Make sure that it's in your \$$PATH (or set \$$CC and \$$CXX)."; exit 1 )
	@echo "[*] Checking for '../afl-2.57b/afl-showmap'..."
	@test -f ../afl-2.57b/afl-showmap || ( echo "[-] Oops, can't find '../afl-2.57b/afl-showmap'. Be sure to compile AFL first."; exit 1 )
	@echo "[+] All set and ready to build."

aflgo-clang: aflgo-clang.c | test_deps
	$(CC) $(CFLAGS) $< -o $@ $(LDFLAGS)
	ln -sf aflgo-clang    aflgo-clang++
	ln -sf aflgo-clang    afl-clang-fast
	ln -sf afl-clang-fast afl-clang-fast++

aflgo-pass.so: aflgo-pass.so.cc | test_deps
	$(CXX) $(CLANG_CFL) -shared $< -o $@ $(CLANG_LFL)

aflgo-runtime.o: aflgo-runtime.o.c | test_deps
	$(CC) $(CFLAGS) -fPIC -c $< -o $@

aflgo-runtime-32.o: aflgo-runtime.o.c | test_deps
	@printf "[*] Building 32-bit variant of aflgo runtime (-m32)... "
	@$(CC) $(CFLAGS) -m32 -fPIC -c $< -o $@ 2>/dev/null; if [ "$$?" = "0" ]; then echo "success!"; else echo "failed (that's fine)"; fi

aflgo-runtime-64.o: aflgo-runtime.o.c | test_deps
	@printf "[*] Building 64-bit variant of aflgo runtime (-m64)... "
	@$(CC) $(CFLAGS) -m64 -fPIC -c $< -o $@ 2>/dev/null; if [ "$$?" = "0" ]; then echo "success!"; else echo "failed (that's fine)"; fi

test_build: $(PROGS)
	@echo "[*] Testing the CC wrapper and instrumentation output..."
	unset AFL_USE_ASAN AFL_USE_MSAN AFL_INST_RATIO; AFL_QUIET=1 AFL_CC=$(CC) ./aflgo-clang $(CFLAGS) ./test-instr.c -o test-instr $(LDFLAGS)
# Use /dev/null to avoid problems with optimization messing up expected
# branches. See https://github.com/google/AFL/issues/30.
	../afl-2.57b/afl-showmap -m none -q -o .test-instr0 ./test-instr < /dev/null
	echo 1 | ../afl-2.57b/afl-showmap -m none -q -o .test-instr1 ./test-instr
	@rm -f test-instr
	@cmp -s .test-instr0 .test-instr1; DR="$$?"; rm -f .test-instr0 .test-instr1; if [ "$$DR" = "0" ]; then echo; echo "Oops, the instrumentation does not seem to be behaving correctly!"; echo; exit 1; fi
	@echo "[+] All right, the instrumentation seems to be working!"

all_done: test_build
	@echo "[+] All done! You can now use 'aflgo-clang' to compile programs."

.NOTPARALLEL: clean

clean:
	rm -f *.o *.so *~ a.out core core.[1-9][0-9]* test-instr .test-instr0 .test-instr1 
	rm -f $(PROGS) aflgo-clang++ afl-clang-fast afl-clang-fast++
