# WARNING in ext4_check_map_extents_env (fs/ext4/inode.c:517, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (fuzzing pipeline reproducer: confirmed=true, status=verified).

## Summary

A warning fires in the ext4 extents-mapping path when checking the
environment for map operations:

```
------------[ cut here ]------------
!inode_is_locked(inode) && !rwsem_is_locked(&inode->i_mapping->invalidate_lock)
WARNING: fs/ext4/inode.c:517 at ext4_check_map_extents_env+0x4b4/0x590 fs/ext4/inode.c:517, CPU#0: syz.8.7999/132148
RIP: 0010:ext4_check_map_extents_env+0x4b4/0x590 fs/ext4/inode.c:517
```

`ext4_check_map_extents_env()` (fs/ext4/inode.c:517) asserts that the
inode is either locked (`inode_is_locked()`) or the mapping's
invalidate_lock is held when extents are mapped; the crafted-image
workload reached the mapping path with neither lock held. The guest runs
with `panic_on_warn=1`, which escalates this warning to a kernel panic.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_EXT4_FS=y, booted with
`oops=panic` and `panic_on_warn=1`. See `environment.txt` for the exact
QEMU command line and `kernel.config` for the configuration.

The crash depends on ext4 state built up by the preceding programs, so
the reproducer is an ordered program set (104 programs):

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

Verified twice: (1) by the fuzzing pipeline's reproducer machinery on a
fresh VM (reproducer pipeline outcome: confirmed=true, status=verified);
(2) by an external standalone chunked-replay run on a fresh VM — the
warning fired 293.4 seconds after guest boot and panic_on_warn escalated
it to a kernel panic (full console: `logs/external-verification/
external-verification-serial.log`). A C reproducer has not been included
yet; it will be added once syzkaller's C-repro stage passes for this bug.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (104-program ordered set + triggering testcase)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The assertion is in `ext4_check_map_extents_env()` — a debug check that
  extents mapping happens under the inode lock or invalidate_lock; the
  repro workload (crafted ext4 image + file operations) reaches
  `ext4_map_blocks`-adjacent code outside those locks.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
