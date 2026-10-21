# devmikey Cocos2d Dumper

High-performance native C++ memory layout dumper, dynamic ARM64 instruction reconciler, and Protobuf wire extractor designed for mobile games and applications built on Cocos2d-x and AArch64/x86_64 native libraries.

---

## Overview

When game updates ship, native C++ structs shift offsets, new fields appear, and virtual function tables change. Reversing thousands of structs by hand in a disassembler takes weeks.

**devmikey Cocos2d Dumper** automates the entire process in seconds:
- **DWARF AST Parsing**: Recursively reconstructs complete C++ class hierarchies, member types, array dimensions, and exact struct field offsets.
- **Dynamic ARM64 Instruction Scanning**: Inspects machine instructions (`ADRP`, `ADD`, `LDR`, `STR`) to identify new or shifted properties directly inside property deserializers.
- **RTTI Virtual Table Recovery**: Scans ELF dynamic relocation tables to pair classes with their exact runtime virtual method tables (`vtable`).
- **Protobuf Wire Schema Extraction**: Extracts and formats embedded Google Protobuf message definitions.
- **Android Floating Overlay App**: Run the entire dumping pipeline directly on your Android device with a lightweight, floating inspection tool.

---

## What Is Inside This Repository

```
devmikey-cocos2d-dumper/
├── Cargo.toml                     # Rust package configuration & dependency definitions
├── dumper.exe                     # Standalone pre-compiled Windows x64 binary
├── mikeycpp_Dumper.apk            # Pre-compiled Android floating overlay app
├── mikeycpp_Dumper_apk_project.zip # Android Studio / Python build project for the APK
├── run_dumper.bat                 # One-click Windows runner (runs dumper.exe directly)
├── run_rust.bat                   # One-click Windows Rust compiler & runner
├── run_dumper.sh                  # One-click Linux / macOS / Termux binary runner
├── run_rust.sh                    # One-click Linux / macOS / Termux Rust compiler & runner
├── src/
│   ├── main.rs                    # Standard Cargo entrypoint
│   └── main_final.rs              # Complete unified dumper & JNI engine source
└── .cargo/
    └── config.toml                # Native CPU build optimizations
```

---

## Quick Start (No Installation Required)

If you just want to dump a `.so` file on Windows without installing anything:

1. Double-click **`run_dumper.bat`** (or run `dumper.exe` in your terminal).
2. The dumper will ask for your target `.so` file.
3. Drag and drop your `.so` file into the terminal window and press **Enter**.
4. (Optional) Drag and drop an older DWARF reference `.so`, or press **Enter** to skip and run in Standalone Mode.
5. (Optional) Enter an output folder, or press **Enter** to use the default folder (`<so_directory>/dump_output`).

The dumper will process the binary and write clean headers and JSON offsets automatically.

---

## Running from Source (Rust & Cargo)

If you want to build or run directly from the Rust source code, use **`run_rust.bat`** (Windows) or **`run_rust.sh`** (Linux / macOS / Termux).

### First-Time Rust Setup

If you already have Rust installed, the runner detects it automatically.

If you installed Rust in a custom folder (for example, `D:\rust\cargo\bin`):
1. Run `run_rust.bat`.
2. When prompted, type or paste the path to your Cargo bin folder.
3. The script automatically saves this path to a local `cargo_env.bat` file.
4. From then on, you never have to type it again.

If you do not have Rust installed yet:
- **Windows**: Download and install from [https://rustup.rs](https://rustup.rs).
- **Linux / macOS**: Run `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
- **Android (Termux)**: Run `pkg install rust clang`

---

## Command Line Usage

You can also pass arguments directly in your terminal:

```bash
# Basic usage with positional arguments:
dumper <TARGET_SO> [DWARF_SO] [OUTPUT_DIR]

# Using explicit flags:
dumper -r libcocos2dcpp.so -d libcocos2dcpp_dwarf.so -o ./my_dump

# Standalone mode (no DWARF reference required):
dumper -r libcocos2dcpp.so -o ./my_dump
```

### Argument Details

- `-r, --release <FILE>`: Path to the target stripped release `.so` library.
- `-d, --dwarf <FILE>`: (Optional) Path to an unstripped reference `.so` containing DWARF debug sections (`.debug_info`). If omitted, the engine runs in Standalone Mode.
- `-o, --output <DIR>`: (Optional) Output directory. Defaults to `dump_output` in the target file's parent folder.

---

## How the Cargo.toml is Structured

Here is a breakdown of the dependencies used in `Cargo.toml` and why each was selected:

```toml
[dependencies]
# Parses DWARF debug sections (.debug_info, .debug_abbrev, .debug_str)
# with zero-copy memory slices for maximum speed.
gimli = "0.34.0"

# Memory-maps multi-gigabyte ELF shared libraries directly into memory.
# Avoids reading large files into RAM buffers, enabling near-instant loading.
memmap2 = "0.9"

# Cross-platform ELF and Mach-O header parser.
# Used to inspect section headers, DT_RELA relocations, and dynamic symbol tables.
object = "0.40.0"

# Multi-threaded work-stealing parallelism.
# Distributes AST resolution across all available CPU cores.
rayon = "1.12.0"

# Serialization framework for exporting clean offsets.json and schemas.
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# JNI bridge bindings.
# Allows the exact same Rust engine to compile into native Android shared libraries (.so)
# and communicate directly with the Android overlay app.
jni = "0.21.1"
```

---

## Output Files

When dumping completes, the output directory contains four clean, unpolluted output sets:

### 1. C Struct Headers (`classes/*.c`)
Individual C-style header files for every class, sorted strictly by memory offset:

```c
// Class: AdventurerRankDefinition | Size: 0x60 | VTable: N/A (Non-Virtual Class)
struct AdventurerRankDefinition {
    string title; // 0x0
    string subTitle; // 0x18
    string icon; // 0x30
    float requiredRank; // 0x48
    float minRankSteps; // 0x4C
    LootDefinition levelUpReward; // 0x50
    LootDefinition rankStepReward; // 0x58
};
```

### 2. JSON Offsets (`offsets.json`)
Machine-readable JSON schema containing every class, size, virtual table address, and member field with hex offsets.

### 3. Enumerations (`enums/Enums.prop`)
Clean key-value property file listing all C++ and Protobuf enumeration values:

```properties
[PlayerFlagType]
PlayerFlagTypeNone = 0
PlayerFlagTypeNormal = 1
PlayerFlagTypePitCrew = 2
PlayerFlagTypeVip = 3
```

### 4. Protobuf Wire Definitions (`hcr2_messages.proto`)
Pure Protobuf schema file containing all message structures and field numbers discovered in the binary.

---

## Android App (`mikeycpp_Dumper.apk`)

The pre-compiled APK included in this repository allows you to perform dumps directly on your mobile device:

1. Install `mikeycpp_Dumper.apk` on your Android device.
2. Grant overlay permission when prompted.
3. Select your target `.so` library from storage.
4. Tap **Start Dump**. The floating overlay will display real-time progress and save output directly to your device storage.

For developers who want to modify or rebuild the Android app, the complete build project is provided in `mikeycpp_Dumper_apk_project.zip`.

---

## License & Credits

Created by **devmikey**. Free for reverse engineering, research, and educational purposes.
