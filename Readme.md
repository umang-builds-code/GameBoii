# GBJoy - Game Boy (DMG) Emulator

GAMEBoiii is an educational Game Boy (DMG-01) emulator written in C++20, utilizing SDL2 for cross-platform video rendering, audio output, and input handling.

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Features](#features)
3. [System Requirements & Dependencies](#system-requirements--dependencies)
4. [Build Instructions](#build-instructions)
   - [Windows (MinGW64 / MSYS2)](#windows-mingw64--msys2)
   - [Linux (Debian / Ubuntu / Fedora)](#linux-debian--ubuntu--fedora)
5. [Usage & Controls](#usage--controls)
6. [Detailed Codebase Audit: Bugs & Errata](#detailed-codebase-audit-bugs--errata)
7. [Step-by-Step Improvement Roadmap](#step-by-step-improvement-roadmap)
8. [License & Acknowledgments](#license--acknowledgments)

---

## Architecture Overview

GBJoy follows the component-based hardware model of the original Nintendo Game Boy (DMG):

```
                        +----------------------+
                        |      Cartridge       |
                        | (ROM / MBC / ExtRAM) |
                        +----------+-----------+
                                   |
                                   v
+------------+          +----------+-----------+          +------------+
|   Joypad   +<-------->+      System Bus      +<-------->+   Screen   |
|  (Inputs)  |          | (Memory Map, IO, DMA)|          | (SDL2 Win) |
+------------+          +----+-------+-------+-+          +-----+------+
                             |       |       |                  ^
                             v       v       v                  |
                        +----+  +----+  +----+                  |
                        |CPU |  |PPU |  |APU +-> Audio Buffer   |
                        |LR35|  |DMG |  |Ch1-4                  |
                        +----+  +--+-+  +----+                  |
                                   |                            |
                                   +--- Pixel Video Buffer -----+
```

### Component Breakdown

| Module | Files | Responsibility |
|---|---|---|
| **Bus** | `include/bus.hpp`, `src/bus.cpp` | Central memory interconnect mapping 64 KB address space (ROM, VRAM, WRAM, Echo, OAM, IO, HRAM, IE). Manages DMA transfers and interrupt request routing (`IF`). |
| **CPU** | `include/cpu.hpp`, `src/cpu.cpp` | Sharp LR35902 8-bit core (hybrid Z80 / Intel 8080). Implements 256 standard opcodes and 256 CB-prefixed opcodes, register file (`AF`, `BC`, `DE`, `HL`, `SP`, `PC`), DIV/TIMA timers, and 5-stage interrupt dispatcher. |
| **PPU** | `include/ppu.hpp`, `src/ppu.cpp` | Picture Processing Unit. Emulates 4 PPU modes (Mode 2: OAM Search, Mode 3: Pixel Transfer, Mode 0: HBlank, Mode 1: VBlank). Handles background scroll, window overlay, and 8x8/8x16 sprite rendering. |
| **APU** | `include/apu.hpp`, `src/apu.cpp` | Audio Processing Unit. Emulates Channel 1 (Pulse + Sweep), Channel 2 (Pulse), Channel 3 (Waveform RAM), and Channel 4 (Noise LFSR), along with frame sequencer (512 Hz). |
| **Cartridge** | `include/cartridge.hpp`, `src/cartridge.cpp` | ROM loader, header parser, and Memory Bank Controller (MBC) switching logic (MBC1 and MBC3 supported). |
| **Joypad** | `include/joypad.hpp`, `src/joypad.cpp` | Direction and action button matrix multiplexer using register `0xFF00` (P1). |
| **Audio** | `include/audio.hpp`, `src/audio.cpp` | SDL2 audio stream output wrapper interfacing with host sound card at 44.1 kHz. |
| **Screen** | `include/screen.hpp`, `src/screen.cpp` | SDL2 window, hardware-accelerated renderer, and streaming texture display at customizable integer scaling. |
| **Main** | `src/main.cpp` | Application entry point, component wiring, master emulation loop, frame rate limiter (60 FPS), and audio synchronization. |

---

## Features

- **Accurate Instruction Decoding**: Complete implementation of standard and CB instruction sets.
- **Scanline Rendering Engine**: Pixel-accurate background tilemaps, hardware window overlay, and prioritized sprite layers.
- **Audio Synthesis**: Multi-channel APU with real-time waveform mixing for pulse, custom wave, and noise.
- **Bank Switching**: Support for ROM/RAM banking with MBC1 and MBC3.
- **Hardware DMA Transfers**: OAM DMA transfer support at `0xFF46`.
- **Keyboard Controls**: Fluid responsive controls mapped to standard PC keys.

---

## System Requirements & Dependencies

- **C++ Compiler**: C++20 compliant compiler (GCC 10+, Clang 11+, or MSVC 2019+).
- **Build System**: `mingw32-make` / `make` or `CMake` (version 3.16+).
- **Libraries**: SDL2 (version 2.0.14 or later, header and runtime binaries).

---

## Build Instructions

### Windows (MinGW64 / MSYS2)

The project includes pre-bundled SDL2 development libraries inside the `windows/` folder.

1. Ensure MinGW64 `g++` and `mingw32-make` (or `make`) are in your system `PATH`:
   ```powershell
   g++ --version
   mingw32-make --version
   ```

2. Navigate into the `windows` directory:
   ```powershell
   cd windows
   ```

3. Build the project using `mingw32-make`:
   ```powershell
   mingw32-make
   ```
   *This compiles all source files and automatically copies `SDL2.dll` next to `GBJoy.exe`.*

4. Clean build outputs:
   ```powershell
   mingw32-make clean
   ```

### Linux (Debian / Ubuntu / Fedora)

1. Install build tools and SDL2 development headers:
   - **Ubuntu/Debian**:
     ```bash
     sudo apt-get update
     sudo apt-get install cmake g++ libsdl2-dev make
     ```
   - **Fedora/RHEL**:
     ```bash
     sudo dnf install cmake gcc-c++ SDL2-devel make
     ```

2. Build using CMake:
   ```bash
   mkdir -p build && cd build
   cmake ..
   make -j$(nproc)
   ```

---

## Usage & Controls

Run the compiled executable from the terminal, supplying the relative or absolute path to a Game Boy ROM:

```bash
# Windows
.\windows\GBJoy.exe .\roms\tetris.gb

# Linux
./build/GBJoy ./roms/tetris.gb
```

Display help and keybindings:
```bash
.\windows\GBJoy.exe help
```

### Controls Mapping

| Game Boy Button | Keyboard Key | Description |
|---|---|---|
| **D-Pad Up** | `W` | Move Up |
| **D-Pad Down** | `S` | Move Down |
| **D-Pad Left** | `A` | Move Left |
| **D-Pad Right** | `D` | Move Right |
| **A** | `J` | Primary Action / Jump |
| **B** | `K` | Secondary Action / Attack / Cancel |
| **Start** | `1` | Start / Pause |
| **Select** | `2` | Select / Options |

---

## Detailed Codebase Audit: Bugs & Errata

An exhaustive audit of the source code identified several bugs, accuracy flaws, and safety hazards. These are documented below without code modifications, serving as the basis for safe, step-by-step refactoring.

### 1. Critical Functional Bugs

| ID | Location | Severity | Description & Impact |
|---|---|---|---|
| **BUG-01** | `src/bus.cpp:43-45` vs `src/bus.cpp:79-82` | **High** | **APU Register Reads Return Zero**: `Bus::write` forwards APU registers (`0xFF10 - 0xFF3F`) to `apu->write`, skipping `io_registers`. However, `Bus::read` never delegates `0xFF10 - 0xFF3F` to `apu->read(addr)`. Consequently, polling `NR52` (audio status) or other sound registers always reads `0x00`, breaking games that check audio channel readiness. |
| **BUG-02** | `src/bus.cpp:97-114` | **High** | **Joypad Line Register Clobbering**: In `key_down()` and `key_up()`, the code calls `write(0xFF00, output)`. In hardware, CPU writes bits 4 & 5 to select D-Pad or buttons, and bits 0-3 are calculated dynamically. Calling `write(0xFF00, ...)` mutates `io_registers[0]` and clears selection lines, corrupting the key matrix. |
| **BUG-03** | `src/ppu.cpp:279-294` | **Medium** | **Mode 1 False Trigger in PPU Scanline Loop**: At the end of Mode 0 (HBlank), `update_stat_mode(1)` is executed unconditionally before checking if `ly == 144`. On lines 0–143, STAT temporarily becomes Mode 1 and fires false VBlank STAT interrupts before switching to Mode 2. |
| **BUG-04** | `src/main.cpp:53-54` | **High** | **Unchecked `Cartridge::load()` Return Value**: `cart->load(...)` returns `false` on file missing or read failure, but `main.cpp` ignores this return value and continues initialization. This triggers an immediate out-of-bounds crash on invalid ROM paths. |
| **BUG-05** | `include/cpu.hpp:15`, `src/cpu.cpp:540` | **High** | **`noexcept` Exception Escalation**: `CPU::step()` is marked `noexcept`, but internal functions throw `std::runtime_error`. In C++, throwing across a `noexcept` boundary causes immediate `std::terminate()` abort, preventing `try/catch` handlers from reporting the error. |
| **BUG-06** | `src/screen.cpp:45` | **Medium** | **Unsafe Flat Texture Copy**: `memcpy(raw_pixels, ...)` ignores texture `pitch` returned by `SDL_LockTexture`. If `pitch` has GPU alignment padding (`pitch != width * 4`), scanlines become skewed or memory is corrupted. |
| **BUG-07** | `src/screen.cpp:31` & `src/audio.cpp:24` | **Medium** | **Multiple `SDL_Quit()` Calls**: Both `Screen` and `Audio` destructors call `SDL_Quit()`. Calling `SDL_Quit()` multiple times during tear-down can lead to access violations on application exit. |

### 2. Emulation Accuracy & Hardware Deviations

| ID | Location | Description & Impact |
|---|---|---|
| **ACC-01** | `src/cartridge.cpp` | **Missing MBC5 Mapper**: Types `0x19 - 0x1E` are unhandled. Modern and late-era Game Boy titles (*Pokémon Yellow*, *Zelda DX*, *Wario Land II/III*) crash or show corrupt graphics without MBC5 banking. |
| **ACC-02** | `src/cartridge.cpp` | **Volatile Save Data (No `.sav` Persistence)**: `eram` is stored exclusively in RAM. Closing the emulator wipes save games. Battery-backed cartridges must automatically load and save `.sav` files to disk. |
| **ACC-03** | `src/cpu.cpp:1888-1895` | **`ADD SP, e8` Half-Carry / Carry Calculation**: Signed vs unsigned promotion on immediate offset `e8` can calculate incorrect H and C flags on negative offsets. |
| **ACC-04** | `src/bus.cpp:49-51` | **IF Register High Bits Unmasked**: Reading `0xFF0F` on DMG should always return bits 5–7 as `1` (`0xE0 | (IF & 0x1F)`). Several game engines check this mask. |
| **ACC-05** | `src/cpu.cpp:982` | **STOP Instruction Unimplemented**: Opcode `0x10` does nothing instead of halting the system clock or preparing speed switches. |

### 3. Performance & Resource Bottlenecks

| ID | Location | Description & Impact |
|---|---|---|
| **PERF-01** | `include/ppu.hpp:21` | **92 KB Video Buffer Copy Every Frame**: `get_buffer()` returns `std::vector<std::uint32_t>` by value instead of `const std::vector<std::uint32_t>&`, creating ~5.5 MB/sec of needless heap allocations. |
| **PERF-02** | `src/main.cpp:79-80` | **Per-Sample `SDL_QueueAudio` Overhead**: Calling `SDL_QueueAudio()` individually 44,100 times per second incurs lock contention inside SDL. Batching audio into 512/1024 sample chunks reduces CPU utilization significantly. |
| **PERF-03** | `include/cpu.hpp:606-612` | **Opcode Dispatcher Virtual / Indirection Overhead**: 512 `std::function<void()>` objects and `std::string` allocations in lookup tables add unnecessary indirection in the hot CPU step loop. |

---

## Step-by-Step Improvement Roadmap

To ensure code stability and maintain existing working components, improvements will be conducted in strictly isolated, testable phases:

### Phase 1: Critical Bug Fixes (Zero Regression)
- [x] Connect `Bus::read` to `APU::read` for address range `0xFF10 - 0xFF3F`.
- [x] Correct Joypad register handling in `Bus::key_down` / `key_up` to eliminate register mutation.
- [x] Fix PPU Mode 0 -> Mode 1 state transition logic in `PPU::tick`.
- [x] Validate `cart->load()` return code in `main.cpp` with user-friendly error messages.
- [x] Make `Screen::update_screen` pitch-aware and safe via `SDL_UpdateTexture`.
- [x] Eliminate double `SDL_Quit` and remove `noexcept` violation in `CPU::step`.

### Phase 2: Memory & Battery Persistence
- [ ] Implement persistent battery-backed SRAM saving and loading (`.sav` file alongside ROM).
- [ ] Implement MBC5 mapper support for expanded game compatibility.

### Phase 3: Performance & Architecture Optimization
- [ ] Change `PPU::get_buffer()` to return a `const` reference.
- [ ] Implement an audio sample queue buffer to batch `SDL_QueueAudio` calls.
- [ ] Eliminate unnecessary heap allocations during CPU instruction lookup.

### Phase 4: Accuracy & Polish
- [ ] Fix IF register mask (`0xE0`) and timer overflow behaviors.
- [ ] Refine window line counter reset conditions.
- [ ] Add customizable scaling and fullscreen toggle hotkeys (e.g. `F11`, integer scale multipliers).

---

## License & Acknowledgments

- **License**: MIT License. See [LICENSE](LICENSE) for details.
- **Pan Docs**: Essential reference documentation by the Game Boy development community ([gbdev.io/pandocs](https://gbdev.io/pandocs/)).
- **SDL2**: Cross-platform audio, video, and event library ([libsdl.org](https://www.libsdl.org/)).
