# Storage — Lenovo ThinkPad P73

## Summary

Total raw storage: **~5.5 TB** across 3 drives (2x NVMe + 1x SATA SSD).

| Drive | Model | Size | Filesystem | Mount | Used | Available |
|-------|-------|------|------------|-------|------|-----------|
| nvme0n1p2 | Crucial CT1000P1SSD8 | 932 GB | ext4 | `/` (root) | 92 GB (11%) | 778 GB |
| nvme1n1p3 | Samsung MZVLB1T0HBLR | 953 GB | ntfs | `/media/Windows` | 675 GB (71%) | 279 GB |
| sda2 | Samsung SSD 870 QVO 4TB | 3.7 TB | ntfs | `/media/Arxv` | 2.4 TB (65%) | 1.4 TB |

## Drive Details

### NVMe 0 — Linux Root (Crucial 1TB)
- **Device:** `/dev/nvme0n1` (931.5 GB)
- **Partitions:** 16 MB reserved + 931.5 GB ext4
- **Mounted at:** `/`
- **Swap:** 64 GB swapfile on this drive (`/swapfile`)
- **Plenty of headroom** — only 11% used

### NVMe 1 — Windows + EFI (Samsung 1TB)
- **Device:** `/dev/nvme1n1` (953.9 GB)
- **Partitions:**
  - 260 MB EFI (`/boot/efi`)
  - 16 MB reserved (MSR)
  - 952.5 GB NTFS — Windows installation (`/media/Windows`)
  - 1.1 GB NTFS recovery (`/media/B8804C7F804C465A`, 88% full)
- **71% used** — mostly Windows data

### SATA — Archive (Samsung 870 QVO 4TB)
- **Device:** `/dev/sda` (3.64 TB)
- **Partitions:** 16 MB reserved + 3.6 TB NTFS
- **Mounted at:** `/media/Arxv`
- **65% used** (2.4 TB) — 1.4 TB still available

## Memory

- **RAM:** 64 GB DDR4 (62 Gi usable)
- **Swap:** 64 GB swapfile on root drive (currently unused)
