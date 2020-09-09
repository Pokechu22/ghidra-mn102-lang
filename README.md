A SLEIGH processor spec for [https://github.com/NationalSecurityAgency/ghidra](Ghidra) for the Matsushita (Panasonic) MN102 processor.  The MN102 is used by the Nintendo GameCube and Wii for the disc drive.

## Installation

This repository should be placed within the Ghidra install as the folder `Ghidra/Processors/MN102`, such that `Ghidra/Processors/MN102/data` is the path to the `data` folder.  Ghidra should automatically add it to the available processor list on its next start, and compile the files when it is first used.

## Manuals

Various processor manuals were once available on Panasonic's website.  Unfortunately, they no longer are; archive.org has partial listings of [hardware manuals](https://web.archive.org/web/20011020080306/http://www.semicon.panasonic.co.jp:80/cgi-bin/micom/manual_d/dwld_products.cgi?email=general&series=MN102H00&s_name=MN102H&mode=general&lang=english&type=hard) and [tool manuals](https://web.archive.org/web/20011020075750/http://www.semicon.panasonic.co.jp:80/cgi-bin/micom/manual_d/dwld_products.cgi?email=general&series=MN102H00&s_name=MN102H&mode=general&lang=english&type=tool), but not the manuals themselves.

The MN102**L** instruction manual is [readily available](http://hitmen.c02.at/files/docs/gc/12250-030e.pdf "MD5: 0219bb3c4efb30aa29351fce0e607b01e; SHA-512: 617b3b2025c1eac51796f685ac9f932ed48f85a3a1197e2108bc3dab9937b9ceb4c39cd10dcde24ec727643fa2dc458a37f4e43e28ca0d3dd95e1321a583d1d3"), but it is for the L variant; per [YAGCD](http://hitmen.c02.at/files/yagcd/yagcd/chap5.html#sec5.7.4) the GameCube uses MN102**H**60GFA.  According to [some product advertising (Pub.No.A000130E, DSA0038294)](https://www.datasheetarchive.com/pdf/download.php?id=7f495d85946b293cc511466b51257a8f2a7523 "MD5: 568235060e006347c75fde79849165cf; SHA-512: c9609630a5de0a56fb2dab7949157cdaa5a2bdf409eafc22d75b3f24344077ef181b7736c0ec093a3c2643788eca8a5a91a85ce3fe57b51afd3fcf6dfe0fce64"), the H version features a hardware multiplier; it actually has some other instructions as well.  Unfortunately, I couldn't find the MN102**H** instruction manual, but several versions of the LSI User's Manual are available, and they include a table of instructions at the end (along with other useful hardware information).  There are at least 3 relevant versions: a [Japanese-language MN10200 series manual (Pub.No.22399-031,  DSA00163540)](https://www.datasheetarchive.com/pdf/download.php?id=b71bc2f3557d4262221f932c55e9f628886b5d "MD5: 1d6e6f090323b08ba1ba6ea689b55f79; SHA-512: b1a98a4f4cfe32990e51c92dbdc38a198ff362dd95bf8feeaab9ff99d4c5458b0bc5b6251fc5350f8656c0d28d29928a9dc2d9be75f6dfc6f4bda55416bdf715"), the [English 1st edition, 1st printing MN102H60G/60K/F60G/F60K LSI User's Manual (Pub.No.22360-011E, DSA00163538)](https://www.datasheetarchive.com/pdf/download.php?id=774b216252be90158be73a949901c252f8878b "MD5: 8547ae70dc805a945e1ac66365700102; SHA-512: cc43ef1e8a174897c103efcd163571f9bb336605c79dbc70f175146121870e3563a753f4e03796f4d88a0a456d85de3114a1c9a0f742f702285934877f24fe03"), and the [English 1st edition, 3rd printing MN102H60G/60K/F60G/F60K LSI User's Manual (Pub.No.22360-013E, DSAH00424716)](https://www.datasheetarchive.com/pdf/download.php?id=c1e2428d529ca2ee89bc232235fd6b988d3ef9 "MD5: 199e73f4d9af63b1041433bd2b2c8492; SHA-512: aa40f59b0ab8c72c823ef6bd8505dc8caaf6052669420ae595281f7cff356534caf683ab170ef14ca6736c72530b0942d37db571137581c2beab0704b8d75f71").

## Notes
### `MOVB Dm,(An)`

The `MOVB Dm,(An)` instruction has incorrect information in the manual.  It claims that it's encoded as `10+Dm<<2+An`, which would result in `0x12` being `MOVB D0,(A2)`.  However, it should be `10+An<<2+Dm`, which would result in `0x12` being `MOVB D2,(A0)`.  Most other instructions have the first register unshifted and the second shifted by 2; for instance, `MOVB Dm,(d8,An)` is encoded as `F5 : 10+An<<2+Dm : d8` (for instance, `MOVB D2,(2,A0)` is `F5 12 02`).  This is the case in all manuals I've checked.

A relevant code snippet (located at around `82F50` in all versions of the GameCube drive firmware):

```asm
; void memset(byte * dst, byte value, uint2 count)
; byte *            A0:3           dst
; byte              D0:1           value
; uint2             D1:3           count
memset:
	ADD        -0x4,SP        ; d3 fc
	; Save the value of D2 (3 bytes)
	MOVX       D2,(0x0,SP)    ; f5 5e 00
	; Move value into D2
	MOV        D0,D2          ; 82
	; Clear D0, to use as a counter
	SUB        D0,D0          ; a0
	; This serves no real purpose since MOVB is used when writing...
	; It might be caused by value being an int according to the C spec
	; (but this version of memset does not return the passed dst pointer,
	; so I'm not sure if value is also kept unnecessarily large)
	EXTXBU     D2             ; be
	BRA        .test          ; ea 05
.loop
	; Write value to dst (*dst = value)
	; The manual would have this as MOVB D0,(A2)
	; which writes the counter to a register not used anywhere else here
	MOVB       D2,(A0)        ; 12
	; Increment counter
	ADD        0x1,D0         ; d4 01
	; Increment dst
	ADD        0x1,A0         ; d0 01
.test
	; Check if the counter is at count
	CMP        D1,D0          ; f3 94
	; If D0-D1 would carry (i.e. D0-D1 < 0, or D0 < D1),
	; then jump back to the loop.  Otherwise continue.
	; Note that this checks CF, so this is a 16-bit comparison.
	BCS        .loop          ; e4 f7
	; Restore contents of D2
	MOVX       (0x0,SP),D2    ; f5 7e 00
	ADD        0x4,SP         ; d3 04
	RTS                       ; fe
```

This is obviously supposed to be something like this:

```C
void memset(byte * dst, byte value, uint2 count) {
	uint2 counter = 0;
	while (counter < count) {
		*dst = value;
		count++;
		dst++;
	}
}
```

but would be this with the manual's `MOVB D0,(A2)`:

```C
void memset(byte * dst, byte value, uint2 count) {
	uint2 counter = 0;
	while (counter < count) {
		*whateverisinA2 = counter;
		count++;
		dst++;
	}
}
```

### `MOV (Di, An), Am` and `MOV Am, (Di, An)`

These instructions aren't listed in the 3rd printing of the MN102H60GFA manual, only in the 1st printing.  They have opcodes F1 00-F1 3F and F1 80-F1 BF. Similar `MOV (Di, An), Dm)` and `MOV Dm (Di, An)` instructions take up the rest of `F1`, and are listed in both printings. (These instructions are listed in the "Differences between 1st Edition 1st Printing and 1st Edition 3rd Printing" section, but it doesn't elaborate on why they were deleted).

The first form shows up in the GameCube drive code as `F1 00` (`MOV (D0, A0), A0`), in code related to the state machines (and thus to multi-dimensional arrays) where it seems plausible.  The second form does not show up at all.  If an implementation bug or something caused it to be removed, it probably does not affect the GameCube, since the relevant function is pretty frequently hit based on my understanding.

Interestingly, these instructions are actually present in the MN102L manual (but invisible; text can be copied out of it and pasted into notepad though).  They can be found on pages 35 (51 in the PDF) and 42 (58 in the PDF).

## Known issues

### Registers show up in decompilation

For some reason, registers show up in the decompilation, instead of using local variables.  For instance, the above memset example actually decompiles to this:

```C
void memset(byte value,uint2 count,byte *dst)
{
  D0 = 0;
  while( true ) {
    if ((ushort)D1 <= (ushort)D0) break;
    *dst = value;
    D0 = D0 + 1;
    dst = dst + 1;
  }
  return;
}
```

Without the current separate processor status variable hackery, it instead looks like this:

```C
void memset(byte value,uint2 count,byte *dst)
{
  ushort uVar1;
  ushort uVar2;
  ushort uVar3;
  ushort uVar4;
  ushort uVar5;

  D0 = 0;
  uVar1 = PSW & 0xff00 | 0x10;
  while( true ) {
    uVar2 = (ushort)(D0 < count) << 6;
    uVar3 = (ushort)SBORROW3(D0,count) << 7;
    uVar4 = (ushort)((ushort)D0 < (ushort)D1) << 2;
    uVar5 = (ushort)SBORROW2((ushort)D0,(ushort)D1) << 3;
    if (((byte)((uVar4 | uVar5) >> 2) & 1) != 1) break;
    *buf = value;
    D0 = D0 + 1;
    buf = buf + 1;
    uVar1 = uVar1 & 0xff00 | uVar2 | uVar3 | uVar4 | uVar5;
  }
  PSW = uVar1 & 0xff00 | uVar2 | uVar3 | uVar4 | uVar5 | (ushort)((int)register0x15 < 0) << 5 |
        (ushort)((undefined *)register0x15 == (undefined *)0x0) << 4 | (ushort)((short)SP < 0) << 1
        | (ushort)((short)SP == 0);
  return;
}
```

This doesn't make sense to me, since other processors don't have this problem (though, I think the 8051 one does, and I based mine off of it).

### Parameter ordering

The MN102 has both data registers (`D0`, `D1`, `D2`, `D3`) and address registers (`A0`, `A1`, `A2`, `SP`).  Parameters can be in either of them, with pointers usually going into address registers.  Thus, the original order of the parameters can't be determined, and things will often be a bit weird (I manually adjusted them in the original `memset` example).

I'm also not 100% sure what the actual calling convention is and whether `D0` is always the return register, or `A0` sometimes is.  The current specification is in `MN102.cspec`.
