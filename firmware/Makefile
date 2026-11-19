# Makefile adaptado do lab-07 para a estrutura do monitor-bpm
PROGNAME = monitor_bpm

CC = arm-none-eabi-gcc
LD = arm-none-eabi-gcc
CP = arm-none-eabi-objcopy
RM = rm -rf

OBJDIR = build
DEPDIR = deps

INCLUDES = -Icmsis -Idrivers -Idevices -Iapp
CFLAGS   = -g -mcpu=cortex-m4 -mthumb -O0 -Wall $(INCLUDES)
DEPFLAGS = -MMD -MF $(DEPDIR)/$*.d
LFLAGS   = -nostdlib -Tlinker/stm32f411-flash.ld

# Todos os .c das pastas do firmware
SRCS = $(wildcard startup/*.c drivers/*.c devices/*.c app/*.c)

OBJS = $(patsubst %.c,$(OBJDIR)/%.o,$(SRCS))
$(shell mkdir -p $(sort $(dir $(OBJS))))

DEPS = $(patsubst %.c,$(DEPDIR)/%.d,$(SRCS))
$(shell mkdir -p $(sort $(dir $(DEPS))))

all: $(PROGNAME).elf

$(PROGNAME).elf: $(OBJS)
	$(LD) $(LFLAGS) $^ -o $@
	$(CP) -O binary $@ $(PROGNAME).bin

$(OBJDIR)/%.o: %.c $(DEPS)
	$(CC) -c $(CFLAGS) $(DEPFLAGS) $< -o $@

$(DEPS):

-include $(DEPS)

clean:
	$(RM) $(OBJDIR) $(DEPDIR) $(PROGNAME).elf $(PROGNAME).bin
