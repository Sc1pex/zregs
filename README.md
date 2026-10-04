# zregs

Type-safe memory-mapped register access for Zig, generated at compile time.

You describe each register's fields (bit offset, width, access mode, and
optionally an enum of allowed values). zregs then generates a struct you can
place over a peripheral's base address, with `read`, `write` and `modify`
methods that only expose the fields you are allowed to touch:

- `read()` returns only the readable (`RO`/`RW`) fields.
- `write(...)` requires every writable (`WO`/`RW`) field and zeroes the rest.
- `modify(...)` does a read-modify-write of only the fields you pass.

Reading a write-only field, writing a read-only one, overlapping fields, or a
misaligned register is a compile error.

Requires Zig 0.16.0 or newer.

## Installation

```sh
zig fetch --save git+https://github.com/Sc1pex/zregs
```

Then in `build.zig`:

```zig
const zregs = b.dependency("zregs", .{ .target = target });
exe.root_module.addImport("zregs", zregs.module("zregs"));
```

## Usage

```zig
const zr = @import("zregs");

// Each register lists its fields in ascending bit order.
// Gaps between fields become reserved bits.
const FR = zr.Register(32, struct {
    busy: zr.Field(3, 1, .RO),
    rxfe: zr.Field(4, 1, .RO),
    txff: zr.Field(5, 1, .RO),
});

const LCRH = zr.Register(32, struct {
    fen: zr.Field(4, 1, .RW),
    wlen: zr.FieldWithValues(5, 2, .RW, enum(u2) {
        bits5 = 0b00,
        bits6 = 0b01,
        bits7 = 0b10,
        bits8 = 0b11,
    }),
});

const DR = zr.Register(32, struct {
    data: zr.Field(0, 8, .RW),
});

// Registers are listed as { name, byte offset, register } in ascending
// offset order. Gaps between registers are padded automatically.
const Uart = zr.RegisterMap(&.{
    .{ "dr", 0x00, DR },
    .{ "fr", 0x18, FR },
    .{ "lcrh", 0x2C, LCRH },
});

const uart: *volatile Uart = @ptrFromInt(0xFE20_1000);

pub fn init() void {
    uart.lcrh.modify(.{ .wlen = .bits8, .fen = 1 });
}

pub fn putc(c: u8) void {
    while (uart.fr.read().txff != 0) {}
    uart.dr.write(.{ .data = c });
}
```

zregs does not insert memory barriers. Add whatever your platform needs
around peripheral accesses.

## Testing

```sh
zig build test
```

## License

[MIT](LICENSE)
