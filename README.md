# Cudy GS1024E Firmware Downgrade Patch (V1.1.6 to V1.0.3)

Reverse engineering analysis and binary patch procedure to bypass firmware version lockouts on the **Cudy GS1024E V1.0** switch.

## Built
**Pre-built Binary:** The patched firmware image is available for download above as [`GS1024E_Downgrade_IMG.img`](./GS1024E_Downgrade_IMG.img).

## Summary

The vendor firmware update validation on the Cudy GS1024E prevents downgrading from `V1.1.6` back to `V1.0.3`. Technical analysis revealed that the verification mechanism does not use cryptographic signatures; instead, it relies on a header field validation (`ARHT` header format) and a standard `zlib.crc32` check computed from offset `0x100`.

By patching the header field at offset `0x04` from `0x00` to `0x11`, the firmware is successfully accepted by the switch loader without corrupting the CRC checksum validation.

---

## Technical Analysis

### Header Comparison

**Firmware V1.0.3:**
```text
00000000: 4152 4854 0000 0000 0000 0100 0001 0000
00000010: 34ff 2300 5423 4039 0000 0000 0000 0000

**Firmware V1.1.6:**
```text
00000000: 4152 4854 1100 0000 0000 0100 0001 0000
00000010: 7cf4 2500 3ca2 e2a2 0000 0000 0000 0000
00000020: 312e 312e 3600 ...

## CRC Identification

The CRC32 stored at offset 0x14 matches zlib.crc32(data[0x100:]):
 - V1.0.3 Header CRC (Little Endian): 0x39402354 (54 23 40 39) -> Matches zlib.crc32(data[0x100:])
 - V1.1.6 Header CRC (Little Endian): 0xa2e2a23c (3c a2 e2 a2) -> Matches zlib.crc32(data[0x100:])

Because offset 0x04 lies outside the calculated CRC range (0x100 to EOF), modifying this byte bypasses the version check without triggering a CRC validation mismatch.

## Patching Procedure

If you have the official GS1024E_20241127V103.img file, you can reproduce the binary patch with the following steps:
**Create a working copy:**
 - cp GS1024E_20241127V103.img downgrade_fixed.img

**Patch the header field at offset 0x04:**
 - printf '\x11\x00\x00\x00' | dd of=downgrade_fixed.img bs=1 seek=4 conv=notrunc

**Verify the change:**
 - xxd -l 16 downgrade_fixed.img -> # Output should show: 00000000: 4152 4854 1100 0000 ...
 - cmp -l downgrade_fixed.img GS1024E_20241127V103.img -> # Output: 5 21 0 (Only byte offset 4 changed)

**Flash the patched image (downgrade_fixed.img) via the switch Web UI.**



## Disclaimer
**This information and patch are provided for educational and research purposes only. Flashing modified firmware carries inherent risks. Perform at your own discretion.**
