Here is the comprehensive engineering roadmap, complete project directory tree, and production-ready build system for **KirnOS**, standardized to use the **`.kn`** file extension for all Kirn source files.

---

# 1. Engineering Master Plan (Phases 0 to 6)

```
[Phase 0: Bootstrap & Toolchain]
               │
               ▼
[Phase 1: KirnCore & Memory Subsystems]
               │
               ▼
[Phase 2: Drivers (KDF) & KirnRing Syscalls]
               │
               ▼
[Phase 3: KirnFS CoW Storage Engine]
               │
               ▼
[Phase 4: Compatibility Subsystems (POSIX/Linux/WinNT)]
               │
               ▼
[Phase 5: KirnSurface Compositor & Realtime Audio]
               │
               ▼
[Phase 6: Desktop Environment & .kapp Bundle Packaging]
```

### Phase 0: Toolchain, Boot Protocol & Bare-Metal Setup
* **Goal**: Establish the cross-compilation pipeline targeting `x86_64-unknown-none-elf` (and later `aarch64-none-elf`) using the Kirn compiler (`knc`).
* **Boot Protocol**: Adopt the **Limine Boot Protocol** (UEFI & BIOS compatible, higher-half 64-bit kernel entry, provides physical memory maps and GOP framebuffer out of the box).
* **Deliverables**: Linker script (`linker.ld`), boot configuration, serial UART output (`COM1 / 0x3F8`) for early kernel panics.

### Phase 1: KirnCore Micro-Hybrid Kernel Core
* **Goal**: Basic execution environment and memory safety.
* **Tasks**:
  1. Initialize the GDT (Global Descriptor Table), IDT (Interrupt Descriptor Table), and TSS.
  2. Implement the **Physical Frame Allocator** (Buddy System / Bitmap allocator) in `memory/pmm.kn`.
  3. Implement **4-Level Paging (Virtual Memory Manager)** in `memory/vmm.kn` (Higher-half kernel mapping at `0xFFFFFFFF80000000`).
  4. Build kernel heap & slab allocators for fixed-size kernel objects (`memory/slab.kn`).
  5. Implement **Capability Handle Tables** (`security/capability.kn`) and the NT-style **Object Manager** (`core/object.kn`).

### Phase 2: Asynchronous I/O Engine & Driver Framework (KDF)
* **Goal**: Bring up the `io_uring`-inspired zero-copy `KirnRing` and user-mode driver framework.
* **Tasks**:
  1. Implement lock-free, atomic SQ/CQ ring buffers in `ipc/ring.kn`.
  2. Hardware device enumeration via PCI Express ECAM (`drivers/pci.kn`).
  3. Implement VirtIO drivers (VirtIO-Net, VirtIO-GPU, VirtIO-Block) for rapid hypervisor testing in QEMU.
  4. Implement native AHCI & NVMe controllers.

### Phase 3: KirnFS Storage Engine
* **Goal**: Fault-tolerant, self-healing, indexed storage.
* **Tasks**:
  1. BLAKE3 hardware-accelerated checksum verification engine.
  2. Copy-on-Write (CoW) B+ Tree block allocator (`fs/kirnfs/btree.kn`).
  3. Relational attribute database indexing for fast file metadata searches.
  4. Virtual File System (VFS) abstraction with Plan 9-style protocol mounting.

### Phase 4: Compatibility Subsystems & Standard Libraries
* **Goal**: Enable standard software (GCC/Clang, Python, Git) to compile and run.
* **Tasks**:
  1. Standard C Runtime binding (`libposix.kn` / `libc.kn`).
  2. Linux Syscall translation persona (translating Linux ABI calls to `KirnRing` messages).
  3. ELF64 userland dynamic loader and Address Space Layout Randomization (ASLR).

### Phase 5: Graphics & Audio Engines
* **Goal**: Real-time rendering and low-latency audio stack.
* **Tasks**:
  1. **KirnSurface**: Direct-scanout user-mode compositor utilizing hardware double-buffering.
  2. Vector UI rendering primitives and GPU shader pipelines.
  3. **KirnSound**: CoreAudio-style lock-free ring-buffer sound pipeline.

### Phase 6: Desktop Environment & Application Bundles
* **Goal**: End-user graphical system.
* **Tasks**:
  1. Implement `.kapp` directory bundle resolver and sandboxing verification.
  2. Native desktop shell (dock, notification center, unified spotlight-style search).
  3. Package manager (`kirn-pkg`) with transactional updates and instant rollbacks.

---

# 2. Project Directory Structure

```text
kirnos/
├── Makefile                        # Master orchestration Makefile
├── build.kn                        # Native Kirn build-system definition
├── limine.conf                     # Limine bootloader configuration
├── linker.ld                       # x86_64 ELF higher-half linker script
├── toolchain/
│   ├── knc                         # Kirn compiler binary / wrapper
│   └── qemu-runner.sh              # Fast testing script
├── boot/
│   └── x86_64/
│       ├── entry.S                 # Low-level boot trampoline (Assembly)
│       └── limine_requests.kn      # Limine protocol structure bindings
├── kernel/
│   ├── main.kn                     # Kernel entry point (kmain)
│   ├── arch/
│   │   └── x86_64/
│   │       ├── gdt.kn              # Global Descriptor Table
│   │       ├── idt.kn              # Interrupt Descriptor Table & ISRs
│   │       ├── cpu.kn              # CPUID, MSRs, CR0/CR3/CR4 controls
│   │       ├── serial.kn           # Early 16550 UART driver
│   │       └── trap.S              # Low-level interrupt handlers
│   ├── core/
│   │   ├── object.kn               # Object Manager & Handle Tables
│   │   ├── process.kn              # Process control blocks (PCB)
│   │   ├── thread.kn               # Thread state and fibers
│   │   ├── scheduler.kn            # Work-stealing scheduler
│   │   └── panic.kn                # Kernel panic & stack trace printer
│   ├── memory/
│   │   ├── pmm.kn                  # Physical Memory Manager (Bitmap/Buddy)
│   │   ├── vmm.kn                  # 4-level / 5-level Paging Engine
│   │   ├── slab.kn                 # Object cache & slab allocator
│   │   └── heap.kn                 # General-purpose kernel heap
│   ├── ipc/
│   │   ├── ring.kn                 # Zero-copy KirnRing (io_uring style)
│   │   ├── channel.kn              # Synchronous & async capability channels
│   │   └── futex.kn                # Fast userspace locking primitives
│   └── security/
│       ├── capability.kn           # seL4/Capsicum capability verifier
│       └── token.kn                # Security identifiers & ACLs
├── drivers/
│   ├── bus/
│   │   └── pci.kn                  # PCIe enumeration & MSI-X configuration
│   ├── storage/
│   │   ├── nvme.kn                 # Native NVMe driver
│   │   └── virtio_blk.kn           # VirtIO block storage driver
│   ├── gpu/
│   │   ├── virtio_gpu.kn           # VirtIO 2D/3D graphics driver
│   │   └── fb.kn                   # Linear GOP framebuffer driver
│   └── net/
│       └── virtio_net.kn           # High-speed VirtIO network driver
├── fs/
│   ├── vfs.kn                      # Virtual File System router
│   ├── devfs.kn                    # Plan 9-style device filesystem
│   └── kirnfs/
│       ├── super.kn                # Superblock & volume descriptor
│       ├── btree.kn                # CoW B+ Tree indexing
│       ├── checksum.kn             # BLAKE3 sector verification
│       └── query.kn                # Database-like metadata query engine
├── libs/
│   ├── libkn/                      # Kirn Standard Library (Core & Alloc)
│   │   ├── sys.kn
│   │   ├── string.kn
│   │   ├── sync.kn
│   │   └── math.kn
│   ├── libposix/                   # POSIX compatibility shim
│   │   └── unistd.kn
│   └── libsurface/                 # UI & Vector Graphics client library
│       ├── window.kn
│       └── widget.kn
├── servers/
│   ├── compositor/
│   │   └── main.kn                 # KirnSurface display compositor daemon
│   └── audio/
│       └── main.kn                 # Low-latency CoreAudio-style sound server
└── apps/
    ├── terminal/
    │   ├── Manifest.kn
    │   └── src/main.kn
    └── desktop/
        ├── Manifest.kn
        └── src/main.kn
```

---

# 3. Build Scripts & Boot Infrastructure

### 3.1. Kernel Linker Script (`linker.ld`)
Configures the kernel to be placed in the higher half at `0xFFFFFFFF80000000`:

```ld
/* KirnOS Higher-Half x86_64 Linker Script */
OUTPUT_FORMAT(elf64-x86-64)
OUTPUT_ARCH(i386:x86-64)

ENTRY(_kstart)

PHDRS
{
    limine_requests PT_LOAD FLAGS(4);           /* Read-only */
    text            PT_LOAD FLAGS(5);           /* Read + Execute */
    rodata          PT_LOAD FLAGS(4);           /* Read-only */
    data            PT_LOAD FLAGS(6);           /* Read + Write */
}

SECTIONS
{
    /* Virtual base: Top 2GB of address space */
    . = 0xFFFFFFFF80000000;

    .limine_requests : {
        KEEP(*(.limine_requests))
    } :limine_requests

    . = ALIGN(4096);

    .text : {
        *(.text .text.*)
    } :text

    . = ALIGN(4096);

    .rodata : {
        *(.rodata .rodata.*)
    } :rodata

    . = ALIGN(4096);

    .data : {
        *(.data .data.*)
    } :data

    .bss : {
        *(COMMON)
        *(.bss .bss.*)
    } :data

    /DISCARD/ : {
        *(.eh_frame)
        *(.note .note.*)
    }
}
```

---

### 3.2. Limine Bootloader Configuration (`limine.conf`)

```ini
TIMEOUT=3
DEFAULT_ENTRY=1

:KirnOS (Micro-Hybrid Kernel)
    PROTOCOL=limine
    KERNEL_PATH=boot:///boot/kirncore.elf
    RESOLUTION=1920x1080x32
    COMMENT=Booting KirnOS with High-Performance Ring Syscalls
```

---

### 3.3. Assembly Bootstrap (`boot/x86_64/entry.S`)

```assembly
/* Early Trampoline: Prepares the CPU and jumps to Kirn kmain */
.section .limine_requests, "a", @progbits
.align 8
.global limine_base_revision
limine_base_revision:
    .quad 2

.section .text
.global _kstart
.type _kstart, @function

_kstart:
    /* Disable interrupts immediately */
    cli

    /* Reset flags register */
    pushq $0
    popfq

    /* Set up a clean initial stack pointer */
    leaq initial_boot_stack_top(%rip), %rsp

    /* Zero the frame pointer for clean debug backtraces */
    xorq %rbp, %rbp

    /* Call the Kirn kernel entry point */
    call kmain

    /* Halt CPU indefinitely if kmain returns */
.global _halt_loop
_halt_loop:
    hlt
    jmp _halt_loop

.section .bss
.align 16
initial_boot_stack_bottom:
    .skip 65536     /* 64 KiB early stack */
initial_boot_stack_top:
```

---

### 3.4. Kernel Entry Point in Kirn (`kernel/main.kn`)

```kirn
module kernel.main;

import kernel.arch.x86_64.serial;
import kernel.arch.x86_64.gdt;
import kernel.arch.x86_64.idt;
import kernel.memory.pmm;
import kernel.memory.vmm;
import kernel.ipc.ring;

pub struct LimineFramebufferRequest {
    pub id: [u64; 4],
    pub revision: u64,
    pub response: *const u64,
}

// Ensure the struct is emitted into the special linker section
@section(".limine_requests")
pub static FRAMEBUFFER_REQ: LimineFramebufferRequest = LimineFramebufferRequest {
    id: [0xc7b35ea743604e10, 0xa49b6b3e3f722f29, 0, 0],
    revision: 0,
    response: null,
};

@export
pub fn kmain() -> noreturn {
    // 1. Initialize serial port for early debug output
    serial::init_uart(serial::COM1_PORT, 115200);
    serial::write_str(serial::COM1_PORT, "[KirnCore] Kernel initialization sequence started...\n");

    // 2. Initialize Core CPU Tables
    gdt::init();
    serial::write_str(serial::COM1_PORT, "[KirnCore] GDT loaded.\n");

    idt::init();
    serial::write_str(serial::COM1_PORT, "[KirnCore] IDT and ISR vectors installed.\n");

    // 3. Initialize Physical and Virtual Memory
    pmm::init();
    serial::write_str(serial::COM1_PORT, "[KirnCore] Physical Frame Allocator active.\n");

    vmm::init();
    serial::write_str(serial::COM1_PORT, "[KirnCore] Virtual 4-Level Paging operational.\n");

    // 4. Initialize KirnRing Syscall Engine
    ring::init_global_subsystem();
    serial::write_str(serial::COM1_PORT, "[KirnCore] Lockless Syscall Ring initialized.\n");

    serial::write_str(serial::COM1_PORT, "[KirnCore] System online. Spawning init process.\n");

    // Enable CPU Interrupts
    @asm volatile ("sti");

    // Kernel idle loop
    while true {
        @asm volatile ("hlt");
    }
}
```

---

### 3.5. Native Build Script in Kirn (`build.kn`)

```kirn
module build;

import std.process;
import std.fs;

pub fn main() -> Result[(), Error] {
    let builder = Builder::init();

    builder.set_target("x86_64-unknown-none-elf");
    builder.set_optimization(Optimization::ReleaseSafe);

    // Compile assembly files
    builder.assemble("boot/x86_64/entry.S", "build/entry.o")?;

    // Compile Kirn kernel files (.kn)
    let kernel_sources = [
        "kernel/main.kn",
        "kernel/arch/x86_64/gdt.kn",
        "kernel/arch/x86_64/idt.kn",
        "kernel/arch/x86_64/serial.kn",
        "kernel/memory/pmm.kn",
        "kernel/memory/vmm.kn",
        "kernel/ipc/ring.kn",
    ];

    for src in kernel_sources {
        builder.compile_kn(src)?;
    }

    // Link higher-half kernel ELF
    builder.link(LinkConfig {
        linker_script: "linker.ld",
        output: "build/iso/boot/kirncore.elf",
        flags: ["--no-relax", "-nostdlib", "-z", "max-page-size=0x1000"],
    })?;

    // Create bootable ISO image
    builder.generate_iso("build/kirnos.iso", "build/iso")?;
    println("Build completed: build/kirnos.iso");

    return Result::Ok(());
}
```

---

### 3.6. Master Orchestration `Makefile` & QEMU Runner

```makefile
# KirnOS Master Makefile
KNC         ?= knc
AS          := x86_64-elf-as
LD          := x86_64-elf-ld
QEMU        := qemu-system-x86_64

BUILD_DIR   := build
ISO_DIR     := $(BUILD_DIR)/iso
KERNEL_ELF  := $(ISO_DIR)/boot/kirncore.elf
IMAGE_ISO   := $(BUILD_DIR)/kirnos.iso

KN_FLAGS    := --target x86_64-none-elf --emit-obj -O2
LDFLAGS     := -n -T linker.ld -nostdlib -z max-page-size=0x1000

KN_SRCS     := $(shell find kernel -type f -name "*.kn")
KN_OBJS     := $(patsubst %.kn, $(BUILD_DIR)/%.o, $(KN_SRCS))

ASM_SRCS    := boot/x86_64/entry.S
ASM_OBJS    := $(BUILD_DIR)/boot/entry.o

.PHONY: all clean run iso

all: $(IMAGE_ISO)

# Assemble entry code
$(BUILD_DIR)/boot/entry.o: boot/x86_64/entry.S
	@mkdir -p $(dir $@)
	$(AS) $< -o $@

# Compile Kirn (.kn) source files
$(BUILD_DIR)/%.o: %.kn
	@mkdir -p $(dir $@)
	$(KNC) $(KN_FLAGS) -c $< -o $@

# Link kernel binary
$(KERNEL_ELF): $(ASM_OBJS) $(KN_OBJS)
	@mkdir -p $(ISO_DIR)/boot
	$(LD) $(LDFLAGS) $(ASM_OBJS) $(KN_OBJS) -o $@

# Build bootable hybrid ISO with Limine
$(IMAGE_ISO): $(KERNEL_ELF)
	@mkdir -p $(ISO_DIR)/boot/limine
	cp limine.conf $(ISO_DIR)/boot/limine/limine.conf
	
	# Fetch Limine binaries if not present
	@if [ ! -f $(BUILD_DIR)/limine/limine-bios.sys ]; then \
		git clone https://github.com/limine-bootloader/limine.git --branch=v7.x-binary --depth=1 $(BUILD_DIR)/limine; \
	fi

	cp $(BUILD_DIR)/limine/limine-bios.sys $(ISO_DIR)/boot/limine/
	cp $(BUILD_DIR)/limine/limine-bios-cd.bin $(ISO_DIR)/boot/limine/
	cp $(BUILD_DIR)/limine/limine-uefi-cd.bin $(ISO_DIR)/boot/limine/

	xorriso -as mkisofs -b boot/limine/limine-bios-cd.bin \
		-no-emul-boot -boot-load-size 4 -boot-info-table \
		--efi-boot boot/limine/limine-uefi-cd.bin \
		-efi-boot-part --efi-boot-image --protective-msdos-label \
		$(ISO_DIR) -o $(IMAGE_ISO)

	$(BUILD_DIR)/limine/limine bios-install $(IMAGE_ISO)
	@echo ">> KirnOS ISO successfully generated at $(IMAGE_ISO)"

# Launch inside QEMU with serial output directed to stdout
run: $(IMAGE_ISO)
	$(QEMU) \
		-M q35 \
		-m 2G \
		-smp 4 \
		-cdrom $(IMAGE_ISO) \
		-serial stdio \
		-vga virtio \
		-device isa-debug-exit,iobase=0xf4,iosize=0x04 \
		-no-reboot

clean:
	rm -rf $(BUILD_DIR)
```

---

## 4. Verification & Testing

To begin testing and booting KirnOS:

1. **Build the image**:
   ```bash
   make
   ```
2. **Execute under QEMU**:
   ```bash
   make run
   ```
3. **Expected Output in Terminal**:
   ```text
   [KirnCore] Kernel initialization sequence started...
   [KirnCore] GDT loaded.
   [KirnCore] IDT and ISR vectors installed.
   [KirnCore] Physical Frame Allocator active.
   [KirnCore] Virtual 4-Level Paging operational.
   [KirnCore] Lockless Syscall Ring initialized.
   [KirnCore] System online. Spawning init process.
   ```

This provides a complete, modern boot harness and development foundation for compiling, linking, and booting **KirnOS** using the **`.kn`** language toolchain.
