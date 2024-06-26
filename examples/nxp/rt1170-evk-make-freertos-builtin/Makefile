CFLAGS  = -W -Wall -Wextra -Werror -Wundef -Wshadow -Wdouble-promotion
CFLAGS += -Wformat-truncation -fno-common -Wconversion
CFLAGS += -g3 -Os -ffunction-sections -fdata-sections
CFLAGS += -I. -Icmsis_core/CMSIS/Core/Include -Icmsis_mcu/devices/MIMXRT1176
CFLAGS += -Icmsis_mcu/devices/MIMXRT1176/drivers -DCPU_MIMXRT1176DVMAA_cm7
CFLAGS += -mcpu=cortex-m7 -mthumb -mfloat-abi=hard -mfpu=fpv5-d16 $(CFLAGS_EXTRA)
CFLAGS += -Wno-conversion -Wno-unused-parameter # due to NXP FSL code
LDSCRIPT = link.ld
LDFLAGS ?= -T$(LDSCRIPT) -nostdlib -nostartfiles --specs nano.specs -lc -lgcc -Wl,--gc-sections -Wl,-Map=$@.map 

SOURCES = main.c syscalls.c sysinit.c
SOURCES += cmsis_mcu/devices/MIMXRT1176/gcc/startup_MIMXRT1176_cm7.S # NXP startup file. Compiler-dependent!
SOURCES += cmsis_mcu/devices/MIMXRT1176/drivers/fsl_clock.c cmsis_mcu/devices/MIMXRT1176/drivers/fsl_anatop_ai.c cmsis_mcu/devices/MIMXRT1176/drivers/fsl_pmu.c cmsis_mcu/devices/MIMXRT1176/drivers/fsl_dcdc.c cmsis_mcu/devices/MIMXRT1176/drivers/fsl_common_arm.c # NXP support files
CFLAGS += -D__ATOLLIC__ -D__STARTUP_CLEAR_BSS	# Make startup code work as expected

# FreeRTOS. RT1170 has a Cortex-M7 r1p2 core, FreeRTOS recommends using CM4F port for non-r0p1 CM7 micros
SOURCES += FreeRTOS-Kernel/portable/MemMang/heap_4.c
SOURCES += FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c
CFLAGS += -IFreeRTOS-Kernel/include 
CFLAGS += -IFreeRTOS-Kernel/portable/GCC/ARM_CM4F -Wno-conversion

# Mongoose options are defined in mongoose_config.h
SOURCES += mongoose.c net.c packed_fs.c

# Example specific build options. See README.md
CFLAGS += -DHTTP_URL=\"http://0.0.0.0/\" -DHTTPS_URL=\"https://0.0.0.0/\"

ifeq ($(OS),Windows_NT)
  RM = cmd /C del /Q /F /S
else
  RM = rm -rf
endif

all build example update: SOURCES += flash_image.c
all build example: firmware.bin

ram: LDSCRIPT = link_ram.ld
ram: firmware.bin

firmware.bin: firmware.elf
	arm-none-eabi-objcopy -O binary $< $@

firmware.elf: FreeRTOS-Kernel cmsis_core cmsis_mcu $(SOURCES) hal.h link_ram.ld link.ld Makefile mongoose_config.h
	arm-none-eabi-gcc $(SOURCES) $(wildcard FreeRTOS-Kernel/*.c) $(CFLAGS) $(LDFLAGS) -o $@
	arm-none-eabi-size $@

flash: firmware.bin
# flash

cmsis_core:     # ARM CMSIS core headers
	git clone -q --depth 1 -b 5.9.0 https://github.com/ARM-software/CMSIS_5 $@
cmsis_mcu:
	curl -sL https://mcuxpresso.nxp.com/cmsis_pack/repo/NXP.MIMXRT1176_DFP.18.0.0.pack -o $@.zip
	mkdir $@ && cd $@ && unzip -q ../$@.zip
FreeRTOS-Kernel:  # FreeRTOS sources
	git clone --depth 1 -b V10.5.0 https://github.com/FreeRTOS/FreeRTOS-Kernel $@

# Automated remote test. Requires env variable VCON_API_KEY set. See https://vcon.io/automated-firmware-tests/
DEVICE_URL ?= https://dash.vcon.io/api/v3/devices/15
update: firmware.bin
	curl --fail-with-body -su :$(VCON_API_KEY) $(DEVICE_URL)/ota --data-binary @$<

update updateram: CFLAGS += -DUART_DEBUG=LPUART2

test: update
	curl --fail-with-body -su :$(VCON_API_KEY) $(DEVICE_URL)/tx?t=5 | tee /tmp/output.txt
	grep 'READY, IP:' /tmp/output.txt       # Check for network init


updateram: ram
	curl --fail-with-body -su :$(VCON_API_KEY) $(DEVICE_URL)/ota --data-binary @firmware.bin
	curl -su :$(VCON_API_KEY) $(DEVICE_URL)/rpc/swd.exec -d '{"req":"init"}'
	curl -su :$(VCON_API_KEY) $(DEVICE_URL)/rpc/swd.exec -d '{"req":"wm,e000edf0,a05f0003 wm,e000edfc,1"}'
	PC=`curl -su :$(VCON_API_KEY) $(DEVICE_URL)/rpc/swd.exec -d '{"req":"rm,4"}' | jq -r .resp[5:]` && \
	SP=`curl -su :$(VCON_API_KEY) $(DEVICE_URL)/rpc/swd.exec -d '{"req":"rm,0"}' | jq -r .resp[5:]` && \
	REQ="wm,e000ed08,0 wr,d,$$SP wr,f,$$PC" && \
	curl -su :$(VCON_API_KEY) $(DEVICE_URL)/rpc/swd.exec -d '{"req":"'"$$REQ"'"}'
	curl -su :$(VCON_API_KEY) $(DEVICE_URL)/rpc/swd.exec -d '{"req":"wm,e000edf0,a05f0001"}'

testram: updateram
	curl --fail-with-body -su :$(VCON_API_KEY) $(DEVICE_URL)/tx?t=5 | tee /tmp/output.txt
	grep 'READY, IP:' /tmp/output.txt       # Check for network init


clean:
	$(RM) firmware.* *.su cmsis_core cmsis_mcu FreeRTOS-Kernel *.zip
