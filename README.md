# WARNING in btrfs_create_pending_block_groups (fs/btrfs/block-group.c:2929, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (fuzzing pipeline reproducer: confirmed=true, status=verified).

## Summary

While committing pending block groups, a warning fires in the block-group
creation path:

```
------------[ cut here ]------------
WARNING: fs/btrfs/block-group.c:2929 at btrfs_create_pending_block_groups+0x1044/0x1810 fs/btrfs/block-group.c:2929, CPU#3: syz.0.749/23716
RIP: 0010:btrfs_create_pending_block_groups+0x1044/0x1810 fs/btrfs/block-group.c:2929
```

The warning sits in `btrfs_create_pending_block_groups()`
(fs/btrfs/block-group.c:2929) at the `insert_block_group_item()` /
`BLOCK_GROUP_FLAG_CHUNK_ITEM_INSERTED` handling: an assertion-style check
about the block group's chunk-item insertion state fired while creating
the pending block group. The guest runs with `panic_on_warn=1`, which
escalates this warning to a kernel panic.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_BTRFS_FS=y, booted with
`oops=panic` and `panic_on_warn=1`. See `environment.txt` for the exact
QEMU command line and `kernel.config` for the configuration.

The crash depends on btrfs state built up by the preceding programs, so
the reproducer is an ordered program set (63 programs):

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

Verified twice: (1) by the fuzzing pipeline's reproducer machinery on a
fresh VM (reproducer pipeline outcome: confirmed=true, status=verified);
(2) by an external standalone chunked-replay run on a fresh VM — the
warning fired 248.5 seconds after guest boot and panic_on_warn escalated
it to a kernel panic (full console: `logs/external-verification/
external-verification-serial.log`). A C reproducer has not been included
yet; it will be added once syzkaller's C-repro stage passes for this bug.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (63-program ordered set + triggering testcase)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The warning is at the chunk-item-inserted flag handling in
  `btrfs_create_pending_block_groups()`; the crafted image + operation
  sequence drives a block group through the pending path in a state the
  assertion does not expect.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
