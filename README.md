# luadec-openwrt-tplink

Lua 5.1 decompiler patched for OpenWrt firmware, with additional support for
**TP-Link BE6700 / Wi-Fi 7 series** (ARM32) devices.

> Decompiled Lua may look wrong in some edge cases, but disassembly results are reliable.

This project is based on [viruscamp/luadec](https://github.com/viruscamp/luadec),
originally patched by [RE-Solver](https://github.com/RE-Solver/luadec-openwrt-tplink)
for OpenWrt + TP-Link sources.

---

## Table of Contents

- [Build](#build)
- [Usage: single file](#usage-single-file)
- [Usage: TP-Link BE6700 (opcode-scrambled firmware)](#usage-tp-link-be6700-opcode-scrambled-firmware)
  - [Pre-built opswap tables](#pre-built-opswap-tables)
  - [Batch decompile all LuCI controllers](#batch-decompile-all-luci-controllers)
  - [Generate opswap table for a different firmware version](#generate-opswap-table-for-a-different-firmware-version)
- [Why BE6700 needs extra steps](#why-be6700-needs-extra-steps)
- [Patches applied](#patches-applied)
- [Tested firmware](#tested-firmware)
- [Credits](#credits)

---

## Build

```bash
sudo apt install libncurses-dev libreadline-dev qemu-user

git clone https://github.com/0xh3y3/luadec-openwrt-tplink
cd luadec-openwrt-tplink

# Build Lua 5.1 runtime (required by luadec/luaopswap)
cd lua-5.1
make linux
cd ..

# Build luadec and luaopswap
cd luadec
make LUAVER=5.1
cd ..
```

After a successful build you will have:
- `luadec/luadec` — the decompiler
- `luadec/luaopswap` — opcode swap tool (needed for TP-Link scrambled firmware)

---

## Usage: single file

```bash
export LD_LIBRARY_PATH=$PWD/lua-5.1/src

# Decompile a Lua 5.1 bytecode file
./luadec/luadec target.luac

# Disassemble (more reliable when decompile output looks wrong)
./luadec/luadec -dis target.luac

# All options
./luadec/luadec -h
```

> **Note:** `LD_LIBRARY_PATH` must point to `lua-5.1/src` because `luadec` links
> against the locally-built `liblua.so` rather than a system-installed one.

---

## Usage: TP-Link BE6700 (opcode-scrambled firmware)

TP-Link BE6700 firmware **shuffles the Lua 5.1 opcode table** (37 of 38 opcodes are
reordered). Running `luadec` directly on these files produces `"bad code"` errors.
You must first restore opcodes with `luaopswap`.

### Pre-built opswap tables

Ready-made swap tables are included in `opswap-tables/`. No QEMU needed.

```
opswap-tables/
└── BE6700_V1.6_20251203.txt          ← swap table (text, human-readable)
└── BE6700_V1.6_20251203_allopcodes.luac  ← reference bytecode used to derive the table
```

**Decompile a single file:**

```bash
export LD_LIBRARY_PATH=$PWD/lua-5.1/src
OPSWAP=opswap-tables/BE6700_V1.6_20251203.txt

# Step 1: restore scrambled opcodes
./luadec/luaopswap -o /tmp/fixed.luac /path/to/target.lua $OPSWAP

# Step 2: decompile the restored bytecode
./luadec/luadec /tmp/fixed.luac
```

### Batch decompile all LuCI controllers

LuCI controller bytecode lives at
`<squashfs-root>/usr/lib/lua/luci/controller/**/*.lua`
(the `.lua` extension is misleading — the files are precompiled Lua 5.1 bytecode).

```bash
export LD_LIBRARY_PATH=$PWD/lua-5.1/src
OPSWAP=$PWD/opswap-tables/BE6700_V1.6_20251203.txt
SYSROOT=/path/to/squashfs-root           # adjust to your extraction path
OUTPUT=/path/to/decompiled_lua           # output directory
mkdir -p "$OUTPUT"

for f in "$SYSROOT"/usr/lib/lua/luci/controller/*.lua \
          "$SYSROOT"/usr/lib/lua/luci/controller/**/*.lua; do
    [[ -f "$f" ]] || continue
    out=$(basename "$f" .lua)_decompiled.lua
    ./luadec/luaopswap -o /tmp/fixed_"$out" "$f" "$OPSWAP" 2>/dev/null
    ./luadec/luadec /tmp/fixed_"$out" > "$OUTPUT/src_${out}" 2>/dev/null \
        && echo "OK: $f" || echo "FAIL: $f"
done
```

Expected output: `OK:` lines for all 82 controllers on BE6700 V1.6 fw20251203.

### Generate opswap table for a different firmware version

If you have a firmware version not listed in `opswap-tables/`, you need to derive a new
table using the firmware's own `luac` binary. This requires `qemu-user`.

```bash
# Prerequisites: firmware extracted to $SYSROOT, qemu-arm installed
export SYSROOT=/path/to/squashfs-root
export LD_LIBRARY_PATH=$PWD/lua-5.1/src

# Step 1: compile allopcodes.lua with the firmware's luac (runs under QEMU)
qemu-arm -L "$SYSROOT" "$SYSROOT/usr/bin/luac" \
    -o /tmp/allopcodes_fw.luac \
    bin/allopcodes-5.1.lua

# Step 2: derive the opcode swap table
./luadec/luaopswap -gs /tmp/allopcodes_fw.luac > opswap-tables/<device>_<fw_version>.txt

# Step 3: verify the table looks reasonable (should list ~37 swapped opcodes)
cat opswap-tables/<device>_<fw_version>.txt

# Step 4: use it (same as above)
./luadec/luaopswap -o /tmp/fixed.luac target.lua opswap-tables/<device>_<fw_version>.txt
./luadec/luadec /tmp/fixed.luac
```

---

## Why BE6700 needs extra steps

TP-Link's OpenWrt-derived firmware for the BE6700 and similar Wi-Fi 7 platforms introduces
changes that break `luaopswap` / `luadec` on a standard x86-64 host:

### 1. Opcode table scrambling

The order of all 38 Lua 5.1 opcodes is shuffled at compile time.
Attempting to load such bytecode without opcode restoration gives:

```
luadec: bad code in precompiled chunk
```

`luaopswap -gs` reconstructs the original→scrambled mapping by comparing opcode
sequences in a reference `.luac` compiled by both the standard and the firmware `luac`.

### 2. `unsigned int` instead of `size_t`

OpenWrt's Lua 5.1 build replaces `size_t` (8 bytes on x86-64) with `unsigned int`
(4 bytes) for string lengths and header size fields. Without the fix, the host reads
8 bytes for a string length field and gets an astronomical number:

```
luaopswap: not enough memory   (reading garbage 8-byte "length")
luaopswap: bad header          (sizeof(size_t) mismatch in header)
```

### 3. `LUA_TINT` integer constant type

TP-Link's build adds a new bytecode constant tag `LUA_TINT` for integer literals,
stored as `lua_Integer` rather than `lua_Number`. Without a handler:

```
luaopswap: bad constant
```

---

## Patches applied

### `luadec/lundump-5.1.c`

| Location | Change | Reason |
|----------|--------|--------|
| `LoadString()` | `size_t size` → `unsigned int size` | ARM32 encodes 4-byte lengths; x86-64 `size_t` reads 8 bytes causing overflow |
| `luaU_header()` | `sizeof(size_t)` → `sizeof(unsigned int)` | Header byte must match ARM32 value (4), not x86-64 (8) |
| `luaU_header()` | integral flag → `sizeof(int)` | TP-Link stores `lua_Integer` size here instead of float type flag |
| new function | `LoadInteger()` | Reads `lua_Integer` from bytecode stream |
| `LoadConstants()` | add `case LUA_TINT:` | Handles integer constant tag added by TP-Link |

### Build files

| File | Change | Reason |
|------|--------|--------|
| `lua-5.1/src/Makefile` | Move `-llua` after object files | Linker requires `-l` flags after the objects that reference them |
| `luadec/Makefile` | Prefix `LUA=` / `LUAC=` with `LD_LIBRARY_PATH=$(LUASRC)` | Non-system `.so` is not found by the linker without the hint |

---

## Tested firmware

| Device | Firmware | Architecture | Controllers decompiled |
|--------|----------|--------------|----------------------|
| TP-Link Archer BE6700 V1.6 | 20251203 | ARM32 (Cortex-A55) | ✅ 82 / 82 |

---

## Credits

- [viruscamp/luadec](https://github.com/viruscamp/luadec) — original luadec
- [RE-Solver/luadec-openwrt-tplink](https://github.com/RE-Solver/luadec-openwrt-tplink) — OpenWrt + TP-Link patches
- [ihipop's blog post](https://blog.ihipop.info/2018/05/5110.html)
- OpenWrt Project
- TP-Link GPL sources: https://static.tp-link.com/resources/gpl/re220v2_gplcode.tar.gz



