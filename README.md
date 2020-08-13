TODO

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