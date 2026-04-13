# luadec-openwrt-tplink

This is an alpha version, decompiled lua may looks wrong but disassembly results looks great.

Overview
========

luadec-openwrt-tplink is a Lua decompiler for lua 5.1 , and experimental for lua 5.2 and 5.3.
However, this version is specifically patched for lua 5.1 according to the changes made by openwrt and some additions from TP-Link sources https://static.tp-link.com/resources/gpl/re220v2_gplcode.tar.gz


This project is based on viruscamp's luadec https://github.com/viruscamp/luadec and Hisham Muhammad's luadec which targeted lua 5.0.x and LuaDec51 by Zsolt Sz. Sztupak.

luadec-openwrt-tplink  is free software and uses the same license as the original LuaDec.

---

## Compatibility: TP-Link BE6700 (ARM32, Wi-Fi 7 series)

This fork includes additional patches on top of the original luadec-openwrt-tplink to support
decompiling LuCI Lua 5.1 bytecode from **TP-Link Archer BE6700 V1.6 (firmware 20251203)** and
likely other ARM32-based TP-Link Wi-Fi 7 devices (BE series).

### What's different on these devices

TP-Link's OpenWrt-derived firmware for the BE6700 and similar Wi-Fi 7 platforms introduces
two changes that break the original loader:

1. **`LUA_TINT` integer constant type** – TP-Link's build of Lua 5.1 adds a new bytecode
   constant tag `LUA_TINT` (integer literals stored as `lua_Integer` instead of `lua_Number`).
   Without handling this tag, `luaopswap` aborts with `"bad constant"`.

2. **`unsigned int` string/size fields** – OpenWrt replaces `size_t` with `unsigned int` for
   string length and header size fields. On x86-64 hosts `size_t` is 8 bytes while the ARM32
   firmware uses 4-byte fields, causing `"not enough memory"` or `"bad header"` errors.

### Patches applied (`luadec/lundump-5.1.c`)

| Patch | Location | Description |
|-------|----------|-------------|
| `unsigned int` string size | `LoadString()` | `size_t size` → `unsigned int size` |
| `unsigned int` header field | `luaU_header()` | `sizeof(size_t)` → `sizeof(unsigned int)` |
| `lua_Integer` size flag | `luaU_header()` | integral flag → `sizeof(int)` |
| `LoadInteger()` function | new function | reads `lua_Integer` from bytecode stream |
| `LUA_TINT` case | `LoadConstants()` | handles integer constant tag |

### Build fixes

| Patch | File | Description |
|-------|------|-------------|
| Linker order | `lua-5.1/src/Makefile` | Move `-llua` after object files to fix `undefined reference` |
| `LD_LIBRARY_PATH` | `luadec/Makefile` | Prefix `LUA=` and `LUAC=` with `LD_LIBRARY_PATH=$(LUASRC)` for non-installed shared lib |

### Tested firmware

| Device | Firmware version | Architecture | Result |
|--------|-----------------|--------------|--------|
| TP-Link Archer BE6700 V1.6 | 20251203 | ARM32 (Cortex-A55) | ✅ All 82 LuCI controllers decompiled |

### Opcode scrambling (luaopswap)

TP-Link BE6700 firmware **reorders the Lua 5.1 opcode table** (37 out of 38 opcodes are
shuffled). Before running `luadec` you must restore the original opcode order using `luaopswap`:

```bash
# Step 1: generate opcode mapping using the firmware's own luac via QEMU
export SYSROOT=/path/to/squashfs-root
cp /root/.agents/skills/luaopswap/allopcodes.lua /tmp/
qemu-arm -L $SYSROOT $SYSROOT/usr/bin/luac -o /tmp/allopcodes_tplink.luac /tmp/allopcodes.lua

# Step 2: generate swap table
./luadec/luaopswap -gs /tmp/allopcodes_tplink.luac > /tmp/tplink_opswap.txt

# Step 3: restore opcodes then decompile
./luadec/luaopswap -o /tmp/fixed.luac target.luac /tmp/tplink_opswap.txt
./luadec/luadec /tmp/fixed.luac
```

---

Compiling
---------
```
sudo apt install libncurses-dev libreadline-dev
git clone https://github.com/RE-Solver/luadec-openwrt-tplink
cd luadec-openwrt-tplink
cd lua-5.1
make linux
cd ../luadec
make LUAVER=5.1
```


Usage
-----
* disassemble lua source or binary  
    luadec -dis abc.lua  
    
* decompile lua binary file:  
  luadec abc.luac  



Use -h to get a complete list of usable parameters


Credits
-------

Originally forked from viruscamp's luadec https://github.com/viruscamp/luadec

ihipop's blog post https://blog.ihipop.info/2018/05/5110.html

Openwrt Project



