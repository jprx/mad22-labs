CC := gcc
AS := as
LD := ld

TARGET := egghunter prefetch

DEPS := ../inc/mad_aslr.h Makefile

OBJECTS := main.o
BUILD_OBJECTS := $(patsubst %,build/%,$(OBJECTS))

ASFLAGS :=
CFLAGS := -I../inc -g -O0 -fno-stack-protector

.PHONY: all clean

all: $(TARGET)

clean:
	rm -rf build $(TARGET)

egghunter: build/egghunter.o build/main.egghunter.o
	@echo " LD    $<"
	@mkdir -p build
	@$(CC) -o $@ -DMAD_EGGHUNTER build/main.egghunter.o build/egghunter.o

build/egghunter.o: egghunter.c $(DEPS) $(BUILD_OBJECTS) build/main.egghunter.o
	@echo " CC    $<"
	@mkdir -p build
	@$(CC) -c $< $(CFLAGS) -DMAD_EGGHUNTER -o $@

prefetch: build/prefetch.o build/main.o
	@echo " LD    $<"
	@mkdir -p build
	@$(CC) -o $@ $(BUILD_OBJECTS) build/prefetch.o

build/prefetch.o: prefetch.c $(DEPS) $(BUILD_OBJECTS)
	@echo " CC    $<"
	@mkdir -p build
	@$(CC) -c $< $(CFLAGS) -o $@

build/main.o: main.c Makefile ../inc
	@echo " CC    $<"
	@mkdir -p build
	@$(CC) -c $< $(CFLAGS) -o $@

build/main.egghunter.o: main.c Makefile ../inc
	@echo " CC    $<"
	@mkdir -p build
	@$(CC) -c $< $(CFLAGS) -DMAD_EGGHUNTER -o $@
