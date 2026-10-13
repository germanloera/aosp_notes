## 1. Not enough space on device for partition product with size 

### Exception

```bash
10-12 19:45:44.233 1456225 1456225 I lpmake  : builder.cpp:1025 [liblp] Partition system will resize from 0 bytes to 1329430528 bytes
10-12 19:45:44.233 1456225 1456225 I lpmake  : builder.cpp:1025 [liblp] Partition system_dlkm will resize from 0 bytes to 8134656 bytes
10-12 19:45:44.233 1456225 1456225 I lpmake  : builder.cpp:1025 [liblp] Partition system_ext will resize from 0 bytes to 412540928 bytes
10-12 19:45:44.233 1456225 1456225 E lpmake  : builder.cpp:607 [liblp] Partition product is part of group emulator_dynamic_partitions which does not have enough space free (458391552 requested, 1750106112 used out of 1887436800)
Not enough space on device for partition product with size 458391552



FAILED: ninja: 
4h32m36.98s Build Failure: 145445 done, 1 failed, 6965 remaining - 8.89/s
 1 steps failed: exit=1
warning: ninja: see ./out/siso_output for full command line and output
 or ./out/siso.INFO
use ./out/siso_failed_commands.sh to re-run failed commands
19:45:52 ninja failed with: exit status 1
```

### Fix

The issue is there is not enough space for emulator partitions, this has happened only when creating an emulator
the file is located in android-17.0.0_r1/device/generic/goldfish/64bitonly/product/, 

the filename is the target in lunch command

Example
 * Command: lunch sdk_phone64_x86_64-aosp_current-en
 * filename: sdk_phone64_x86_64.mk

You need to replace  the following line 

```make
BOARD_EMULATOR_DYNAMIC_PARTITIONS_SIZE ?= $(shell expr 1800 \* 1048576 )
```

With

```make
BOARD_EMULATOR_DYNAMIC_PARTITIONS_SIZE ?= $(shell expr 2000 \* 1048576 )
```

Replace the number 1800 with a larger number and rerun the m command
it takes less time to compile

### Fixed file 

```make

#
# Copyright (C) 2009 The Android Open Source Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# android-17.0.0_r1/device/generic/goldfish/64bitonly/product/



PRODUCT_USE_DYNAMIC_PARTITIONS := true


PARTITION_SIZE_INCREMENTAL_MULTIPLIER ?= $(shell expr 4) #
PARTITION_SIZE_INCREMENTAL ?= $(shell expr 200 \* $(PARTITION_SIZE_INCREMENTAL_MULTIPLIER)) # Each increase to the multiplier adds 200 bytes to the partition
PARTITION_SIZE_INCREMENTAL_VALUE ?= $(shell expr 1800 + $(PARTITION_SIZE_INCREMENTAL))



BOARD_EMULATOR_DYNAMIC_PARTITIONS_SIZE ?= $(shell expr  $(PARTITION_SIZE_INCREMENTAL_VALUE) \* 1048576 )
BOARD_SUPER_PARTITION_SIZE := $(shell expr $(BOARD_EMULATOR_DYNAMIC_PARTITIONS_SIZE) + 8388608 )  # +8M

$(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit_only.mk)

# Enable mainline checking for this excat product name
ifeq (sdk_phone64_x86_64,$(TARGET_PRODUCT))
PRODUCT_ENFORCE_ARTIFACT_PATH_REQUIREMENTS := relaxed
endif

PRODUCT_SDK_ADDON_SYS_IMG_SOURCE_PROP := \
    device/generic/goldfish/64bitonly/product/phone_source.prop_template

$(call inherit-product, device/generic/goldfish/board/emu64x/details.mk)
$(call inherit-product, device/generic/goldfish/product/phone.mk)

PRODUCT_BRAND := Android
PRODUCT_NAME := sdk_phone64_x86_64
PRODUCT_DEVICE := emu64x
PRODUCT_MODEL := Android SDK built for x86_64



```