# game2048: losing corrupts the display because set_vram_update(0) leaves a stale pointer high byte

## Summary

After losing a game in `samples\game2048`, the display becomes corrupted: title-screen graphics reappear, with garbled tiles, text, and colored fragments across the screen instead of a stable board and loss message.

**RCA: confirmed in the locally rebuilt ROM.** The transpiler emits an incomplete 16-bit argument for `set_vram_update(0)`. It clears `A` but leaves `X` holding the high byte of the preceding ROM-table pointer. VRAM updates therefore remain enabled with an invalid update-stream pointer. Subsequent NMIs interpret unrelated ROM bytes as PPU update commands.

**Fix applied:** scalar byte-sized addresses are now zero-extended for `set_vram_update`, while existing word values and array pointers retain their high byte. The sample explicitly uses a `ushort` zero address in both its loss and victory handlers.

## Reproduction

1. Build and launch the sample:

   ```powershell
   dotnet build samples\game2048\game2048.csproj
   dotnet run --project samples\game2048\game2048.csproj
   ```

2. Press Start and play until the board is full with no legal merges.
3. Observe the display when the loss notification appears.

**Expected:** The board remains visible, the vertical `YOU LOST  PRESS START` message appears, and the screen stays stable while waiting for Start.

**Actual:** The screen shows title-screen content mixed with corrupted graphics. The supplied MesenCE screenshot shows the large outlined `2048` title and instructions underneath the corruption.

Attach the original screenshot to this issue.

## Investigation environment

- Repository checkout: `27bddcf551a04d1338c80afdb23187a339860b0b`.
- Windows; .NET SDK `11.0.100`; sample target `net10.0`.
- Sample rebuilt with the locally installed `dotnes` package `0.3.3-beta`; emulator supplied by `dotnes.mesen` `2.2.1`.
- Mapper 0, 32 KiB PRG ROM, 8 KiB CHR ROM, horizontal mirroring.
- Investigated ROM: `samples\game2048\bin\Debug\net10.0\game2048.nes`.
- ROM SHA-256: `0D858B007D9350E7DB20CE375CE9C8B8C8FAE142645334F2AFB5F3B3DE7E8E49`.

The addresses below belong to this ROM and should be rediscovered for other builds.

## Confirmed failure chain

The [loss handler](https://github.com/dartasen/dotnes/blob/27bddcf551a04d1338c80afdb23187a339860b0b/samples/game2048/Program.cs#L610-L622) does:

```csharp
set_vram_update(UPD_LOST);
ppu_wait_nmi();
set_vram_update(0);
while (true)
{
    ppu_wait_nmi();
    if ((pad_poll(0) & PAD.START) != 0) break;
}
```

The loss-message buffer is a valid 25-byte vertical update stream: header, 21 tile bytes, and an EOF marker. The problem occurs when disabling updates after displaying that message.

The emitted loss-handler code is:

```asm
$9579: A9 B5     LDA #$B5
$957B: A2 9F     LDX #$9F
$957D: 20 76 83  JSR $8376   ; set_vram_update($9FB5), the loss-message stream
$9580: 20 F0 82  JSR $82F0   ; ppu_wait_nmi()
$9583: A9 00     LDA #$00
$9585: 20 76 83  JSR $8376   ; intended set_vram_update($0000), but X is still $9F
$9588: 20 F0 82  JSR $82F0   ; next NMI processes the unintended stream
```

The [runtime routine](https://github.com/dartasen/dotnes/blob/27bddcf551a04d1338c80afdb23187a339860b0b/src/dotnes.tasks/ObjectModel/BuiltInSubroutines.cs#L1548-L1560) correctly expects the full pointer in `A:X`:

```asm
$8376: 85 04     STA $04     ; NAME_UPD_ADR low
$8378: 86 05     STX $05     ; NAME_UPD_ADR high
$837A: 05 05     ORA $05
$837C: 85 06     STA $06     ; NAME_UPD_ENABLE = low | high
$837E: 60        RTS
```

Consequently, the supposed disable call writes:

```text
NAME_UPD_ADR    ($04-$05) = $9F00    expected $0000
NAME_UPD_ENABLE ($06)    = $9F      expected $00
```

`ppu_wait_nmi()` preserves the caller's `X` in this sequence. The subsequent wait loop keeps requesting VRAM processing. [UpdVRAM](https://github.com/dartasen/dotnes/blob/27bddcf551a04d1338c80afdb23187a339860b0b/src/dotnes.tasks/ObjectModel/BuiltInSubroutines.cs#L2245-L2255) sees the nonzero enable flag and calls the update-stream parser using `$9F00`, which is not the selected message buffer.

## Runtime confirmation and control experiment

Two headless MesenCE runs used the same rebuilt ROM and deterministic setup:

1. Press Start through scripted controller input.
2. At frame 180, replace the 16 board-state bytes at `$0325-$0334` with alternating tile ranks `1` and `2`, arranged as a checkerboard with no legal merges. This forces the normal loss-detection branch without playing a complete game.
3. Let the unmodified loss handler execute and capture the pointer, enable flag, palette, nametable RAM, and final screen.
4. Repeat with one debugger-only intervention: set `cpu.x = 0` immediately before instruction `$9583`. No ROM bytes or repository source were changed.

| Observation | Original execution | Control: clear X before disable |
|---|---|---|
| Loss-message pointer | `$9FB5` | `$9FB5` |
| Pointer after supposed disable | `$9F00` | `$0000` |
| Enable flag after supposed disable | `$9F` | `$00` |
| Nametable bytes changed between message completion and frame 300 | 156 of 2048 | 0 of 2048 |
| Final screen | Reproduces the reported title-screen/garbled-graphics appearance | Existing board graphics and loss message remain stable |

Both runs had identical nametable RAM immediately after the message update, before the next NMI processed the invalid pointer. Palette snapshots also remained identical. This isolates the stale high byte as the cause of the reproduced corruption, rather than a bad loss-message asset or a palette initialization problem.

**Scope of confirmation:** This was an accelerated loss-path reproduction, not a complete natural-play playthrough. The original symptom was reported during normal play. The win path and Release configuration still need runtime regression checks.

## Code-generation analysis

- [`WriteLdc(byte)`](https://github.com/dartasen/dotnes/blob/27bddcf551a04d1338c80afdb23187a339860b0b/src/dotnes.tasks/Utilities/IL2NESWriter.LocalVariables.cs#L273-L338) emits the small constant into `A` without defining `X`.
- The [default NESLib call path](https://github.com/dartasen/dotnes/blob/27bddcf551a04d1338c80afdb23187a339860b0b/src/dotnes.tasks/Utilities/IL2NESWriter.ILDispatch.cs#L3352-L3427) handles byte-array pointers but does not widen a byte-sized zero into the complete word required by `set_vram_update(ushort)`.
- [`ReflectionCache`](https://github.com/dartasen/dotnes/blob/27bddcf551a04d1338c80afdb23187a339860b0b/src/dotnes.tasks/Utilities/ReflectionCache.cs#L42-L46) selects the `byte[]` overload by name. That is relevant when designing an overload-aware fix; it is not sufficient to assume the cached method describes a raw-address call.
- At the investigated baseline, the [win handler](https://github.com/dartasen/dotnes/blob/27bddcf551a04d1338c80afdb23187a339860b0b/samples/game2048/Program.cs#L623-L635) used the same `set_vram_update(0)` pattern; the fix covers both terminal paths.

## Fix details

The default call lowering now clears `X` for scalar `set_vram_update` arguments known to fit in one byte, unless a word value is already loaded in `A:X`. It excludes byte-array pointer arguments, so ROM/RAM update buffers remain valid. The runtime routine was not changed: address `$0100` must stay enabled, even though its low byte is zero.

The regression test selects an array buffer first to seed a nonzero high byte, then verifies a literal zero, a runtime byte zero, and a word zero all disable updates. It also checks `$0123`, `$0100`, and array-pointer handling. `samples\game2048\Program.cs` now passes a named `ushort` zero value from both loss and win handlers.

## Validation

- Rebuilt the game2048 sample in Debug and Release with zero warnings or errors.
- Rebuilt the sample test assemblies and updated the game2048 ROM snapshot to reflect the sample source change; other sample snapshots were unchanged.
- The full test suite passed: 68 analyzer tests and 3,093 dotnes tests. The repository's package-clean target was temporarily skipped because a locked file prevented deleting `samples\packages`; the target was restored afterward.
- Headless MesenCE runs forced full boards into both the loss and victory paths. In each path, the post-message update pointer became `$0000`, updates were disabled after the next NMI, and nametable RAM remained unchanged afterward.
