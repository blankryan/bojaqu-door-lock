# Flash Partition Layout (16 MB — `partitions.csv`)

Per [ADR-0006](../adr/0006-ota-update-and-flash-partitioning.md), both `dev` and `prod` presets share `CONFIG_PARTITION_TABLE_OFFSET=0x10000` (giving 64 KB bootloader space for Secure Boot v2 + Flash Encryption) and the exact 16 MB (`0x1000000`) flash layout defined in [partitions.csv](../../partitions.csv):

| Name | Type | SubType | Offset | Size (Hex) | Size (Human) | Flags | Purpose |
|---|---|---|---|---|---|---|---|
| *(bootloader)* | — | — | `0x000000` | `0x010000` | 64 KB | — | 2nd-stage bootloader (fits SBv2 + Flash Enc + Anti-Rollback) |
| *(part table)* | — | — | `0x010000` | `0x001000` | 4 KB | — | Partition table binary + MD5 |
| `otadata` | `data` | `ota` | `0x011000` | `0x002000` | 8 KB | — | Active OTA slot selector & pending-verify state |
| `phy_init` | `data` | `phy` | `0x013000` | `0x001000` | 4 KB | — | RF PHY calibration data |
| `nvs_keys` | `data` | `nvs_keys` | `0x014000` | `0x001000` | 4 KB | `encrypted` | AES-XTS keys for NVS encryption (`prod`) |
| `nvs` | `data` | `nvs` | `0x015000` | `0x020000` | 128 KB | *(AES-XTS via `nvs_keys`)* | Wi-Fi creds, DESFire credential store, master div key, runtime config |
| `fctry` | `data` | `nvs` | `0x035000` | `0x00B000` | 44 KB | *(AES-XTS via `nvs_keys`)* | Read-only factory identity: per-device X.509 mTLS cert/key, serial, BLE PoP |
| `ota_0` | `app` | `ota_0` | `0x040000` | `0x500000` | **5 MB** | *(Flash-Enc in `prod`)* | Primary OTA firmware slot (64 KB aligned) |
| `ota_1` | `app` | `ota_1` | `0x540000` | `0x500000` | **5 MB** | *(Flash-Enc in `prod`)* | Secondary OTA firmware slot (64 KB aligned) |
| `coredump` | `data` | `coredump` | `0xA40000` | `0x010000` | 64 KB | — | Crash dump ELF/summary for field diagnostics |
| `assets` | `data` | `spiffs` | `0xA50000` | `0x200000` | 2 MB | — | Read-only voice prompts & chime audio files |
| `storage` | `data` | `spiffs` | `0xC50000` | `0x3B0000` | 3,776 KB (~3.69 MB) | — | Offline audit event ring buffer & snapshot staging |
| **Total** | | | **End: `0x1000000`** | **`0x1000000`** | **16,384 KB (16 MB)** | | Exact 16 MB boundary |

> [!WARNING]
> Do **not** add the `encrypted` flag to `nvs` or `fctry` in `partitions.csv`! ESP-IDF NVS encryption uses software AES-XTS keyed from the flash-encrypted `nvs_keys` partition; marking `data, nvs` with `encrypted` causes ESP-IDF build/runtime errors.
