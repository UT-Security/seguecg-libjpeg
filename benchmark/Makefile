NOTPARALLEL:

.DEFAULT_GOAL := build

SHELL := /bin/bash

MKFILE_PATH := $(abspath $(lastword $(MAKEFILE_LIST)))
MKFILE_DIR := $(dir $(MKFILE_PATH))
REPO_PATH=$(MKFILE_DIR)/..
ROOT_PATH=$(REPO_PATH)/..
PARALLEL_COUNT=$(shell nproc)

CC=clang
CXX=clang++ -std=c++17
WASI_SDK_PATH=$(ROOT_PATH)/wasi-sdk
WASI_CC=$(WASI_SDK_PATH)/bin/clang --sysroot $(WASI_SDK_PATH)/share/wasi-sysroot/
WASI_CXX=$(WASI_SDK_PATH)/bin/clang++ --sysroot $(WASI_SDK_PATH)/share/wasi-sysroot/ -std=c++17
WASI_AR=$(WASI_SDK_PATH)/bin/ar
WASI_LD=$(WASI_SDK_PATH)/bin/wasm-ld
WASI_CFLAGS=-I $(MKFILE_DIR)/wasmcompat
WASI_LDFLAGS=-Wl,--export-all -Wl,--stack-first -Wl,-z,stack-size=1048576 -Wl,--no-entry -Wl,--import-memory -Wl,--import-table
WASI_SA_LDFLAGS=-Wl,--export-all -Wl,--stack-first -Wl,-z,stack-size=1048576

RLBOX_WASM2C_PATH=$(ROOT_PATH)/rlbox_wasm2c_sandbox

WASM2C_SRC_PATH=$(ROOT_PATH)/seguecg-wasm2c
WASM2C_RUNTIME=$(WASM2C_SRC_PATH)/wasm2c/wasm-rt-impl.c $(RLBOX_WASM2C_PATH)/src/wasm2c_rt_mem.c $(RLBOX_WASM2C_PATH)/src/wasm2c_rt_minwasi.c
WASM2C_SA_RUNTIME=$(WASM2C_SRC_PATH)/wasm2c/wasm-rt-impl.c $(RLBOX_WASM2C_PATH)/src/wasm2c_rt_minwasi.c

ifdef DEBUG
BUILD_TYPE_DIR=debug
BUILD_TYPE_CONFIG=Debug
BUILD_TYPE_FLAGS=-Og -g
BUILD_BINARY_FLAGS=-fsanitize=address
else
BUILD_TYPE_DIR=release
BUILD_TYPE_CONFIG=Release
BUILD_TYPE_FLAGS=-O3
BUILD_BINARY_FLAGS=
endif

LIBJPEG_INCLUDES=-I$(REPO_PATH)/ -I$(REPO_PATH)/build_$(BUILD_TYPE_DIR)
LIBJPEG_NOSIMD_INCLUDES=-I$(REPO_PATH)/ -I$(REPO_PATH)/build_nosimd_$(BUILD_TYPE_DIR)
LIBJPEG_NOSIMD_WASM_INCLUDES=-I$(REPO_PATH)/ -I$(REPO_PATH)/build_nosimd_wasm_$(BUILD_TYPE_DIR)
LIBJPEG_NOSIMD_WASM_SA_INCLUDES=-I$(REPO_PATH)/ -I$(REPO_PATH)/build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)
LIBJPEG_LINK=$(REPO_PATH)/build_$(BUILD_TYPE_DIR)/libjpeg.a
LIBJPEG_NOSIMD_LINK=$(REPO_PATH)/build_nosimd_$(BUILD_TYPE_DIR)/libjpeg.a
LIBJPEG_NOSIMD_WASM_LINK=-Wl,--whole-archive $(REPO_PATH)/build_nosimd_wasm_$(BUILD_TYPE_DIR)/libjpeg.a -Wl,--no-whole-archive
LIBJPEG_NOSIMD_WASM_SA_LINK=-Wl,--whole-archive $(REPO_PATH)/build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/libjpeg.a -Wl,--no-whole-archive
RLBOX_INCLUDES=-I $(ROOT_PATH)/rlbox/code/include -I $(RLBOX_WASM2C_PATH)/include -DRLBOX_WASM2C_MODULE_NAME=libjpeg
RLBOX_LINK=-lrt
WASM2C_BUILD_INCLUDES=-I $(WASM2C_SRC_PATH)/wasm2c -I$(REPO_PATH)/build_nosimd_wasm_$(BUILD_TYPE_DIR)/ -DWASM_RT_SKIP_SIGNAL_RECOVERY=1
WASM2C_SA_BUILD_INCLUDES=-I $(WASM2C_SRC_PATH)/wasm2c -I$(REPO_PATH)/build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/ -DWASM_RT_SKIP_SIGNAL_RECOVERY=1

################Deps#####################

test_out.jpeg: test.jpeg ../build_$(BUILD_TYPE_DIR)/libjpeg.a
	$(CC) -O0 -g generate_test_out.c -o ./generate_test_out $(LIBJPEG_INCLUDES) $(LIBJPEG_LINK)
	./generate_test_out test.jpeg test_out.jpeg

test_bytes.h: test.jpeg test_out.jpeg
	echo "unsigned char inputData[] = {" > $@
	hexdump -ve '1/1 "0x%.2x, "' test.jpeg >> $@
	echo "0x00 };" >> $@
	echo "unsigned char outputData[] = {" >> $@
	hexdump -ve '1/1 "0x%.2x, "' test_out.jpeg >> $@
	echo "0x00 };" >> $@

$(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR)/wasm2c:
	cmake -S $(WASM2C_SRC_PATH) -B $(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR) -DCMAKE_BUILD_TYPE=$(BUILD_TYPE_CONFIG)
	cd $(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR) && make -j$(PARALLEL_COUNT)

#####################################

../build_$(BUILD_TYPE_DIR)/libjpeg.a:
	cmake -S ../ -B ../build_$(BUILD_TYPE_DIR) -DCMAKE_BUILD_TYPE=$(BUILD_TYPE_CONFIG) -DWITH_SIMD=1 -DREQUIRE_SIMD=1
	cd ../build_$(BUILD_TYPE_DIR) && make -j$(PARALLEL_COUNT)

../build_nosimd_$(BUILD_TYPE_DIR)/libjpeg.a:
	cmake -S ../ -B ../build_nosimd_$(BUILD_TYPE_DIR) -DCMAKE_BUILD_TYPE=$(BUILD_TYPE_CONFIG) -DWITH_SIMD=0
	cd ../build_nosimd_$(BUILD_TYPE_DIR) && make -j$(PARALLEL_COUNT)

../build_nosimd_wasm_$(BUILD_TYPE_DIR)/libjpeg.wasm.c: $(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR)/wasm2c
	CC="$(WASI_CC)" CFLAGS="$(WASI_CFLAGS)" LDFLAGS="$(WASI_LDFLAGS)" \
		CXX="$(WASI_CXX)" CXXFLAGS="$(WASI_CFLAGS)" LDFLAGS="$(WASI_LDFLAGS)" \
		AR="$(WASI_AR)" LD="$(WASI_LD)" \
		cmake -S ../ -B ../build_nosimd_wasm_$(BUILD_TYPE_DIR) -DCMAKE_BUILD_TYPE=$(BUILD_TYPE_CONFIG) -DENABLE_SHARED=0 -DWITH_SIMD=0
	cd ../build_nosimd_wasm_$(BUILD_TYPE_DIR) && make -j$(PARALLEL_COUNT)
	$(WASI_CC) $(BUILD_TYPE_FLAGS) $(WASI_CFLAGS) $(WASI_LDFLAGS) -o ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/libjpeg.wasm $(RLBOX_WASM2C_PATH)/c_src/wasm2c_sandbox_wrapper.c $(LIBJPEG_NOSIMD_WASM_LINK)
	$(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR)/wasm2c ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/libjpeg.wasm -o ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/libjpeg.wasm.c

../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/libjpeg.wasm.c: $(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR)/wasm2c image_change_quality.c
	CC="$(WASI_CC)" CFLAGS="$(WASI_CFLAGS)" LDFLAGS="$(WASI_SA_LDFLAGS)" \
		CXX="$(WASI_CXX)" CXXFLAGS="$(WASI_CFLAGS)" LDFLAGS="$(WASI_SA_LDFLAGS)" \
		AR="$(WASI_AR)" LD="$(WASI_LD)" \
		cmake -S ../ -B ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR) -DCMAKE_BUILD_TYPE=$(BUILD_TYPE_CONFIG) -DENABLE_SHARED=0 -DWITH_SIMD=0
	cd ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR) && make -j$(PARALLEL_COUNT)
	$(WASI_CC) $(BUILD_TYPE_FLAGS) $(WASI_CFLAGS) $(WASI_SA_LDFLAGS) -o ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/libjpeg.wasm image_change_quality.c $(LIBJPEG_NOSIMD_WASM_SA_INCLUDES) $(LIBJPEG_NOSIMD_WASM_SA_LINK)
	$(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR)/wasm2c ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/libjpeg.wasm -o ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/libjpeg.wasm.c

#####################################

../build_$(BUILD_TYPE_DIR)/image_change_quality: image_change_quality.c ../build_$(BUILD_TYPE_DIR)/libjpeg.a
	$(CC) $(BUILD_TYPE_FLAGS) $(BUILD_BINARY_FLAGS) -o $@ $< $(LIBJPEG_INCLUDES) $(LIBJPEG_LINK)

../build_nosimd_$(BUILD_TYPE_DIR)/image_change_quality: image_change_quality.c ../build_nosimd_$(BUILD_TYPE_DIR)/libjpeg.a
	$(CC) $(BUILD_TYPE_FLAGS) $(BUILD_BINARY_FLAGS) -o $@ $< $(LIBJPEG_NOSIMD_INCLUDES) $(LIBJPEG_NOSIMD_LINK)

../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/image_change_quality_wasm2c: image_change_quality.c  $(REPO_PATH)/benchmark/wasm-rt-static-runner.c ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/libjpeg.wasm.c
	$(WASM2C_SRC_PATH)/build_$(BUILD_TYPE_DIR)/wasm2c ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/libjpeg.wasm -o ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/image_change_quality_wasm2c.wasm.c
	$(CC) $(BUILD_TYPE_FLAGS) $(BUILD_BINARY_FLAGS) -o $@ $(WASM2C_SA_RUNTIME) $(REPO_PATH)/benchmark/wasm-rt-static-runner.c ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/image_change_quality_wasm2c.wasm.c $(WASM2C_SA_BUILD_INCLUDES) $(RLBOX_INCLUDES)

#####################################

../build_$(BUILD_TYPE_DIR)/image_change_quality_rlbox_noop: image_change_quality_rlbox.cpp ../build_$(BUILD_TYPE_DIR)/libjpeg.a
	$(CXX) $(BUILD_TYPE_FLAGS) $(BUILD_BINARY_FLAGS) -o $@ $< $(LIBJPEG_INCLUDES) $(RLBOX_INCLUDES) $(LIBJPEG_LINK) $(RLBOX_LINK)

../build_nosimd_$(BUILD_TYPE_DIR)/image_change_quality_rlbox_noop: image_change_quality_rlbox.cpp ../build_nosimd_$(BUILD_TYPE_DIR)/libjpeg.a
	$(CXX) $(BUILD_TYPE_FLAGS) $(BUILD_BINARY_FLAGS) -o $@ $< $(LIBJPEG_NOSIMD_INCLUDES) $(RLBOX_INCLUDES) $(LIBJPEG_NOSIMD_LINK) $(RLBOX_LINK)

../build_nosimd_wasm_$(BUILD_TYPE_DIR)/image_change_quality_rlbox_wasm2c: image_change_quality_rlbox.cpp ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/libjpeg.wasm.c
	rm -rf ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/objs
	mkdir -p ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/objs
	cd ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/objs && \
		$(CC) $(BUILD_TYPE_FLAGS) $(BUILD_BINARY_FLAGS) $(WASM2C_RUNTIME) $(RLBOX_INCLUDES) $(REPO_PATH)/build_nosimd_wasm_$(BUILD_TYPE_DIR)/libjpeg.wasm.c -c $(WASM2C_BUILD_INCLUDES)
	$(CXX) $(BUILD_TYPE_FLAGS) $(BUILD_BINARY_FLAGS) ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/objs/*.o -o $@ $< -DRLBOX_WASM2C $(LIBJPEG_NOSIMD_WASM_SA_INCLUDES) $(WASM2C_BUILD_INCLUDES) $(RLBOX_INCLUDES) $(RLBOX_LINK)

#####################################

# ../build_$(BUILD_TYPE_DIR)/image_change_quality ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/image_change_quality_wasm2c
build: test_bytes.h
build: ../build_$(BUILD_TYPE_DIR)/image_change_quality_rlbox_noop
build: ../build_nosimd_$(BUILD_TYPE_DIR)/image_change_quality_rlbox_noop
build: ../build_nosimd_wasm_$(BUILD_TYPE_DIR)/image_change_quality_rlbox_wasm2c
build: ../build_$(BUILD_TYPE_DIR)/image_change_quality
build: ../build_nosimd_$(BUILD_TYPE_DIR)/image_change_quality
build: ../build_nosimd_wasm_sa_$(BUILD_TYPE_DIR)/image_change_quality_wasm2c

test_rlbox:
	echo "JPEG Stock"
	cd $(REPO_PATH)/build_$(BUILD_TYPE_DIR) && ./image_change_quality_rlbox_noop
	sleep 1
	echo "JPEG Stock NoSIMD"
	cd $(REPO_PATH)/build_nosimd_$(BUILD_TYPE_DIR) && ./image_change_quality_rlbox_noop
	sleep 1
	echo "JPEG Wasm"
	cd $(REPO_PATH)/build_nosimd_wasm_$(BUILD_TYPE_DIR) && ./image_change_quality_rlbox_wasm2c

test:
	echo "JPEG Stock"
	cd $(REPO_PATH)/build_$(BUILD_TYPE_DIR) && ./image_change_quality
	sleep 1
	echo "JPEG Stock NoSIMD"
	cd $(REPO_PATH)/build_nosimd_$(BUILD_TYPE_DIR) && ./image_change_quality
	sleep 1
	echo "JPEG Wasm"
	cd $(REPO_PATH)/build_nosimd_wasm_sa_$(BUILD_TYPE_DIR) && ./image_change_quality_wasm2c


clean:
	rm -f test_out.jpeg
	rm -f test_bytes.h
	rm -rf ../build_*/
	rm -rf $(WASM2C_SRC_PATH)/build_*