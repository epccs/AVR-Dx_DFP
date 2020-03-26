# Atmel Packs (AVR-Dx_DFP)

downloaded from http://packs.download.atmel.com/


# Usage

avr-gcc -mmcu=atmega4809 -B ../ATmega_DFP/gcc/dev/atmega4809/ -I ../ATmega_DFP/include/


# Atmel toolchain

http://distribute.atmel.no/tools/opensource/Atmel-AVR-GNU-Toolchain/

https://www.microchip.com/mplab/avr-support/avr-and-sam-downloads-archive

# Example Makefile

```
# used to kick the tires
TARGET = BlinkLED
LIBDIR = ../lib
OBJECTS = main.o \
	$(LIBDIR)/twi0_mc.o

# Chip and project-specific global definitions
MCU   =  atmega4809
F_CPU = 2000000UL/6
# BAUD  =  38400UL
# remove -DBAUD=$(BAUD) since it is used in iom4809.h as register e.g., USART0.BAUD
CPPFLAGS = -DF_CPU=$(F_CPU)

# Cross-compilation wtih Atmel toolchain 3.6.2
CC = ~/Samba/avr8-3.6.2/avr8-gnu-toolchain-linux_x86_64/bin/avr-gcc
OBJCOPY = ~/Samba/avr8-3.6.2/avr8-gnu-toolchain-linux_x86_64/bin/avr-objcopy
OBJDUMP = ~/Samba/avr8-3.6.2/avr8-gnu-toolchain-linux_x86_64/bin/avr-objdump
SIZE = ~/Samba/avr8-3.6.2/avr8-gnu-toolchain-linux_x86_64/bin/avr-size


# UPDI is the programing interface, it is half-duplex UART based
# Most USB to serial bridges show as /dev/ttyUSB0, 
# Uno's serial bridge (an ATmega16U2) shows as /dev/ttyACM0  (a modem,?)
# Pi Zero on chip hardware serial shows as /dev/ttyAMA0 (hardware UART on a Linux system)
detect_PORT := $(shell sh -c 'ls /dev/ttyAMA0 2>/dev/null || echo not')
ifeq ($(detect_PORT),/dev/ttyAMA0)
	UPDI_PORT = /dev/ttyAMA0
endif
detect_PORT := $(shell sh -c 'ls /dev/ttyUSB0 2>/dev/null || echo not')
ifeq ($(detect_PORT),/dev/ttyUSB0)
	UPDI_PORT = /dev/ttyUSB0
endif

# Compiler/linker options
CFLAGS = -Os -g -std=gnu99 -Wall
# CFLAGS += -funsigned-char -funsigned-bitfields 
# CFLAGS += -fpack-struct -fshort-enums 
CFLAGS += -ffunction-sections -fdata-sections 

# atmega4809 is not in the avr-gcc packaged for my OS
# -I see https://gcc.gnu.org/onlinedocs/gcc-8.2.0/gcc/Directory-Options.html#Directory-Options
# -B see same as above
TARGET_ARCH = -mmcu=$(MCU) \
-B $(LIBDIR)/ATmega_DFP/gcc/dev/atmega4809/ \
-I $(LIBDIR)/ATmega_DFP/include/ \
## if someday it is in mainline use
##TARGET_ARCH = -mmcu=$(MCU)

LDFLAGS = -Wl,-Map,$(TARGET).map -Wl,--gc-sections 

.PHONY: help

# some help for the make impaired
# https://marmelab.com/blog/2016/02/29/auto-documented-makefile.html
help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

all: $(TARGET).hex $(TARGET).lst ## build the image and its related files

$(TARGET): $(TARGET).hex

$(TARGET).hex: $(TARGET).elf
	$(OBJCOPY) -j .text -j .data -O ihex $< $@

# # This MCU has a built in uploader as part of UPDI, so no bootloader is needed but avrdude is not set up to upload to this interface so...
# sudo apt install python3-pip
# pip3 install serial
# # place pyupdi next to your project repository
# git clone https://github.com/mraardvark/pyupdi
# # fuses, what, what?
updi: $(TARGET).hex ## upload with pyupid 
	shell sh -c 'which python3 2>/dev/null || echo install python3'
	shell sh -c 'ls ../../pyupdi/pyupdi.py 2>/dev/null || echo clone pyupdi'
	python3 ../../pyupdi/pyupdi.py -v -d $(MCU) -c $(UPDI_PORT) -b 115200 -e -f $(TARGET).hex

$(TARGET).elf: $(OBJECTS)
	$(CC) $(LDFLAGS) $(TARGET_ARCH) $^ -o $@
#	$(SIZE) -C --mcu=$(MCU) $@
# avr-size -C is not in mainline, it uses pramaters from a config file that needs added for each new chip
# but the pramaters that allow showing % size are now in the elf, 
# avr-objdump -s -j .note.gnu.avr.deviceinfo Uart0_hello.elf
# https://www.eit.lth.se/fileadmin/eit/courses/edi021/Avr-libc-2.0.0/mem_sections.html#sec_dot_note
	$(SIZE) $@
	rm -f $(TARGET).o $(OBJECTS)

clean: ## remove the image and its related files
	rm -f $(TARGET).hex $(TARGET).map $(TARGET).elf $(TARGET).lst
 
%.lst: %.elf
	$(OBJDUMP) -h -S $< > $@
```
