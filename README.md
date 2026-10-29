# CDFTL on FEMU

[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)
[![Platform](https://img.shields.io/badge/Platform-x86--64-brightgreen)](https://shields.io/)

A demand-based, two-level cached flash translation layer (**CDFTL**) implemented
inside the black-box SSD (`bbssd`) FTL of [FEMU](https://github.com/MoatLab/FEMU),
the QEMU-based NVMe SSD emulator. On top of the CDFTL algorithms, this project adds
a **batched translation-page update scheme for data garbage collection** that
removes redundant read-modify-write cycles on the translation flash.

The full design report with measurements and graphs is in
[`docs/CDFTL-report.pdf`](docs/CDFTL-report.pdf).

---

## Table of Contents

- [Background](#background)
- [What Is Implemented](#what-is-implemented)
- [Architecture](#architecture)
  - [Translation Flash](#translation-flash)
  - [GTD, CMT and CTP](#gtd-cmt-and-ctp)
  - [Translation Block Management](#translation-block-management)
- [Algorithms](#algorithms)
  - [Fetch-in](#fetch-in)
  - [Replace](#replace)
  - [Address Lookup and Update](#address-lookup-and-update)
  - [Data GC](#data-gc)
  - [Map GC](#map-gc)
- [Batched Mapping Update in Data GC](#batched-mapping-update-in-data-gc)
- [Evaluation](#evaluation)
- [Statistics](#statistics)
- [SSD Configuration](#ssd-configuration)
- [Build and Run](#build-and-run)
- [Repository Layout](#repository-layout)
- [References](#references)
- [License](#license)

---

## Background

DFTL keeps the page-level mapping table on flash and caches individual mapping
entries in RAM. It has two well-known weaknesses:

- **Eviction cost.** Evicting a single dirty mapping entry can consume a whole
  translation page on flash, because the caching granularity is one entry.
- **Locality.** Caching only individual entries exploits temporal locality but
  ignores the spatial locality of sequential workloads.

**CDFTL** addresses both by caching at two granularities at once:

| Cache | Granularity | Locality exploited |
|-------|-------------|--------------------|
| Cached Mapping Table (**CMT**) | one mapping entry (`dlpn -> dppn`) | temporal |
| Cached Translation Page (**CTP**) | one translation page (512 mappings) | spatial |

The two caches share one RAM budget. A CMT entry costs one space unit and a CTP
entry costs `N` units, so evicting `N` CMT entries makes room for one translation
page in CTP. The split between the two caches therefore adapts to the workload.

## What Is Implemented

- A dedicated **translation flash** emulated separately from the data flash.
- **GTD**, **CMT** and **CTP** with hash-table lookup and LRU replacement.
- CDFTL **Fetch-in**, **Replace** and **Garbage Collection** paths wired into the
  FEMU `bbssd` mapping functions.
- A separate **map GC** for translation blocks with a greedy victim policy.
- A **bucket-based batched update** of translation pages during data GC.
- **Statistics** (CMT/CTP hit ratio, cache miss ratio, translation-flash
  read/write counts, WAF) exposed through an NVMe Get Log Page command.

All FTL code lives in `hw/femu/bbssd/ftl.c` and `hw/femu/bbssd/ftl.h`. The
statistics hook is in `hw/femu/bbssd/bb.c`.

## Architecture

```
          DRAM                                   Flash
 +---------------------+          +----------------------------------+
 |  GTD  (3072 x tppn) |          |  Translation flash (tnand)       |
 |  CMT  (1024 x 8 B)  | <------> |  1 bank x 16 blk x 256 pg x 4 KB |
 |  CTP  (  32 x 4 KB) |          +----------------------------------+
 +---------------------+          |  Data flash (FEMU bbssd)         |
                                  +----------------------------------+
```

### Translation Flash

Translation pages are stored in a small NAND emulator that is independent from
the data flash. It uses the same page geometry as the data flash but does no
striping, and it has its own garbage collector.

| Parameter | Value |
|-----------|-------|
| Banks / planes | 1 / 1 |
| Blocks | 16 |
| Pages per block | 256 |
| Page size | 4 KB |
| Total size | 16 MB |
| Mappings per translation page | 512 (`4096 / sizeof(struct ppa)`) |

The whole space is managed through the `tnand` pointer and the following API.
Every call returns `NAND_SUCCESS` (0) or a negative error code.

```c
int tnand_init(int nbanks, int nblks, int npages);
int tnand_read(int bank, int blk, int page, void *data);
int tnand_write(int bank, int blk, int page, void *data);
int tnand_erase(int bank, int blk);

#define NAND_SUCCESS         0
#define NAND_ERR_INVALID    -1
#define NAND_ERR_OVERWRITE  -2
#define NAND_ERR_POSITION   -3
#define NAND_ERR_EMPTY      -4
```

### GTD, CMT and CTP

| Structure | Implementation | Size |
|-----------|----------------|------|
| **GTD** (Global Translation Directory) | array indexed by `tvpn`; each entry holds `tppn`, a location bit (flash or CTP) and a dirty bit | `MAX_TVPN = 3072` entries |
| **CMT** | hash table (`dlpn % 10` buckets) of singly-linked chains plus a doubly-linked LRU list | `MAX_CMTN = 1024` entries, 8 B each |
| **CTP** | hash table (`tvpn % 5` buckets) plus an LRU list; each entry owns a 4 KB `map_page` | `MAX_CTPN = 32` pages, 4 KB each |

The cache budget is `1024 * 8 + 32 * 4096` bytes. The cache is considered full
when the bytes currently occupied by CMT and CTP entries, plus one more CMT entry
and one more CTP entry, would exceed that budget (`is_cache_full()`).

### Translation Block Management

- **Free block list**: a FIFO queue (`Queuetype`) of erased translation blocks.
- **Active block list**: a min-heap (`Heaptype`) of blocks that hold valid pages.
- **Write cursor**: `tblk_curr` tracks the current block and next free page.
- A reverse map `tp2l[tppn] -> tvpn` and a per-block invalid-page counter
  (`blk_invalid[]`) support victim selection.
- **GC threshold**: map GC runs as soon as the free list drops below one block.

## Algorithms

### Fetch-in

`fetch_in(dlpn)` is called on a cache miss in the read path.

1. Check `is_cache_full()`.
2. If the cache is full, make room for one CMT entry and one CTP entry:
   - Evict one CMT entry by LRU.
   - If enough CMT entries whose translation page is also resident in CTP exist
     (at least one page's worth), flush those mappings into the cached page and
     evict them from CMT. Otherwise evict one CTP entry by LRU, writing it back
     to the translation flash if it is dirty.
3. Read the translation page that contains `dlpn` with `tnand_read()`, insert
   the requested mapping into CMT, and insert the whole page into CTP.

`ctp_fetch_in(dlpn)` is the same procedure without the CMT part. It is used by
the write path when the target mapping is not cached anywhere.

### Replace

`replace(dlpn, dppn)` updates a mapping after data is written to NAND. Four
cases are handled:

| CMT | CTP | Action |
|-----|-----|--------|
| hit | hit | Update the page in CTP, remove every CMT entry that belongs to the same translation page, move the CTP entry to MRU. |
| miss | hit | Update the page in CTP, set the GTD dirty bit, move to MRU. |
| hit | miss | Update the CMT entry, set its dirty bit, move to MRU. |
| miss | miss | `ctp_fetch_in()` the page into CTP only, then update it there. |

### Address Lookup and Update

- `get_maptbl_ent()` checks CMT, then CTP, and finally calls `fetch_in()` on a
  full miss. It returns the `dppn` from whichever level answered.
- `set_maptbl_ent()`, which originally wrote the in-DRAM mapping table in FEMU,
  now delegates to `replace()`.

### Data GC

CDFTL requires that GC does **not** evict cache entries. For each valid page
copied out of a victim block the original algorithm does:

1. If the mapping is in CMT, update it there.
2. Else if its translation page is in CTP, update it there.
3. Otherwise read the translation page from flash, patch it, and write it back
   immediately.

That linear path is kept as `set_maptbl_datagc()`. The default build uses the
batched path described in the next section instead.

### Map GC

`map_garbage_collection()` is triggered whenever a translation-page write would
run out of free translation blocks. Translation-page writes originate from:

- evicting a dirty CMT entry whose page is not resident in CTP,
- evicting a dirty CTP entry,
- translation-page updates during data GC.

Procedure:

1. Select the victim with the **greedy** policy (most invalid pages).
2. For every valid page in the victim:
   - if the page is cached in CTP and dirty, copy the CTP version;
   - otherwise read the page from the translation flash;
   - merge every dirty CMT mapping that belongs to the page;
   - clear the dirty bits and write the merged page to a fresh location.
3. Erase the victim block and return it to the free list.

## Batched Mapping Update in Data GC

### Problem

With the linear algorithm, every valid page whose mapping is absent from both
caches costs one `tnand_read()` and one `tnand_write()`. When many valid pages
in one victim block map into the **same** translation page, that page is read,
patched and rewritten over and over. Each rewrite also invalidates the previous
copy, which speeds up map GC.

### Approach

Instead of updating mappings one by one, the new mappings produced while
cleaning a block are collected in memory, grouped by `tvpn`, and applied to each
translation page exactly once after the whole block has been scanned.

```c
typedef struct _Bucket {
    uint64_t dlpn;
    uint64_t dppn;
    struct _Bucket *next;
} Bucket;

struct _TvpnBucket {
    Bucket   *gc_copy_list[MAX_TVPN + 1]; /* list head per tvpn              */
    uint64_t  exist[MAX_TVPN];            /* tvpn -> index in gc_copy_list   */
    uint64_t  bucket_size;                /* number of distinct tvpns        */
    uint64_t  pool_idx;                   /* number of valid pages so far    */
};

Bucket bucket_pool[256];                  /* one slot per page in a block    */
struct _TvpnBucket TvpnBucket;
```

- `bucket_pool` is a static pool with one slot per page of a block, so no
  allocation happens during GC.
- `exist[]` maps a `tvpn` to a 1-based slot in `gc_copy_list`, which keeps the
  per-`tvpn` lists compact regardless of which `tvpn`s appear.
- Each `Bucket` is 24 bytes, so the block-level pool is 6 KB.

### Flow

1. `clean_one_block()` resets `TvpnBucket` before scanning the victim block.
2. `gc_write_page()` copies each valid page and appends `(dlpn, new dppn)` to the
   list for its `tvpn` instead of touching the translation flash.
3. After the scan, `set_maptbl_batchgc()` walks each `tvpn` list once:
   - load the translation page (from flash, or a blank page if it never existed);
   - for every queued mapping, update CMT if present, else CTP if present, else
     patch the in-memory page;
   - if anything was patched in memory, write the page **once** with
     `tnand_write()` and update GTD and `tp2l`; otherwise leave it in cache.

The same mechanism can be applied per line (superblock) instead of per block.
That variant was measured for comparison but is not the default, because it
needs 32x more memory.

## Evaluation

Workload: `fio` sequential write of 3 GB followed by 5 loops of random write of
3 GB, unless stated otherwise. All graphs are in
[`docs/CDFTL-report.pdf`](docs/CDFTL-report.pdf).

### Translation-page writes during data GC

| Scheme | Batch unit | Buffer memory | Translation writes saved vs. linear |
|--------|-----------|---------------|-------------------------------------|
| Linear (original CDFTL) | none | 0 | baseline |
| **Batched (default)** | block (256 pages) | 256 x 24 B = 6 KB | about 18,000 fewer `tnand_write()` calls |
| Batched | line (4 ch x 8 LUN x 256 pages) | 8192 x 24 B = 192 KB | about 30,000 fewer `tnand_write()` calls |

Block-level batching captures most of the benefit for roughly 3% of the memory
of the line-level variant.

### Cache hit ratio vs. workload

CTP caching gives spatial locality on top of CMT's temporal locality, but a
random-write phase still lowers the combined cache hit ratio by about 30%
compared with a sequential-write phase.

### Write amplification vs. workload

Only writes update the mapping, so a 50:50 read/write mix shows a lower WAF than
a write-only workload of the same size.

## Statistics

The following counters are kept in `struct ssd` and printed through the FEMU
log on demand:

| Counter | Meaning |
|---------|---------|
| `cmt_hits`, `ctp_hits`, `cache_misses`, `total_requests` | hit/miss accounting in `get_maptbl_ent()` and `set_maptbl_ent()` |
| `tnand_reads_cnt`, `tnand_writes_cnt` | translation-flash traffic |
| `tnand_writes_gc` | translation-page writes caused by data GC |
| WAF | `(host writes + GC writes + translation writes) / host writes` |

They are returned by the NVMe **Get Log Page** command with log identifier
`0x17` (`bb_get_log()` in `hw/femu/bbssd/bb.c`). Inside the guest VM:

```bash
sudo nvme write-amplification-log /dev/nvme0n1
```

`write-amplification-log` is a small nvme-cli extension that issues that log
page; any tool that sends Get Log Page with LID `0x17` works. The values are
printed on the host side in the FEMU log, for example:

```
Cache Hit Rate Log:
CMT Hits=..., CTP Hits=..., Cache Misses=..., Total Requests=...,
CMT Hit Rate=..%, CTP Hit Rate=..%, Cache Miss Rate=..%
tnand_read count : ...
tnand_write count : ...
tnand_write_gc count : ...
```

## SSD Configuration

The FTL constants are sized for the following geometry:

| Parameter | Value |
|-----------|-------|
| Sector / page | 512 B / 4 KB (8 sectors) |
| Pages per block | 256 |
| Blocks per plane | 256 |
| LUNs per channel | 8 |
| Channels | 4 |
| Logical capacity | 6 GB (`N_LPNS = 1,572,864`) |
| Translation pages (`MAX_TVPN`) | 3072 |
| Translation space | 12 MB of mappings in a 16 MB translation flash |
| CMT / CTP | 1024 entries / 32 pages |

To reproduce this, set `nchs=4` and `ssd_size=6144` in
`femu-scripts/run-blackbox.sh`. For a different geometry, update `MAX_TVPN`,
`MAX_TPPN`, `MAX_CMTN` and `MAX_CTPN` in `hw/femu/bbssd/ftl.h` accordingly.

## Build and Run

The build follows the standard FEMU procedure.

```bash
git clone https://github.com/Jhyunil/SA-CDFTL.git
cd SA-CDFTL
mkdir -p build-femu && cd build-femu
cp ../femu-scripts/femu-copy-scripts.sh .
./femu-copy-scripts.sh .
sudo ./pkgdep.sh      # install dependencies
./femu-compile.sh     # configure + build qemu-system-x86_64
```

Place a guest image at `$HOME/images/u20s.qcow2` (or edit `OSIMGF` in the run
script), then start the emulator in black-box mode:

```bash
./run-blackbox.sh
```

The emulated NVMe device shows up inside the guest as `/dev/nvme0n1`. Run your
workload (for example with `fio`) and query the statistics as shown above. See
the [FEMU wiki](https://github.com/MoatLab/FEMU/wiki) for guest image
preparation and general usage.

## Repository Layout

| Path | Contents |
|------|----------|
| `hw/femu/bbssd/ftl.c` | translation flash, GTD/CMT/CTP, fetch-in, replace, data GC, map GC, batched GC update |
| `hw/femu/bbssd/ftl.h` | data structures, constants, prototypes |
| `hw/femu/bbssd/bb.c` | NVMe command hooks, statistics log page |
| `femu-scripts/` | build and run scripts |
| `docs/CDFTL-report.pdf` | design report with graphs and raw numbers |

Everything outside `hw/femu` is unmodified FEMU / QEMU.

## References

- Z. Qin, Y. Wang, D. Liu, and Z. Shao, *An Adaptive Demand-Based Caching
  Mechanism for NAND Flash Memory Storage Systems*, ACM Transactions on Design
  Automation of Electronic Systems, 2012.
- A. Gupta, Y. Kim, and B. Urgaonkar, *DFTL: A Flash Translation Layer Employing
  Demand-based Selective Caching of Page-level Address Mappings*, ASPLOS 2009.
- H. Li et al., *The CASE of FEMU: Cheap, Accurate, Scalable and Extensible Flash
  Emulator*, FAST 2018. https://github.com/MoatLab/FEMU

## License

This project inherits the license of FEMU and QEMU, GPL-2.0. See
[`LICENSE`](LICENSE) and [`COPYING`](COPYING).
