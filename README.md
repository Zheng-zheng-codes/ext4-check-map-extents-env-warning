# WARNING in ext4_check_map_extents_env (fs/ext4/inode.c:517, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (fuzzing pipeline reproducer: confirmed=true, status=verified).

## Summary

A warning fires in the ext4 extents-mapping path when the uprobe
instruction-reading path reaches `ext4_map_blocks()` without the expected
locks:

```
------------[ cut here ]------------
!inode_is_locked(inode) && !rwsem_is_locked(&inode->i_mapping->invalidate_lock)
WARNING: fs/ext4/inode.c:517 at ext4_check_map_extents_env+0x4b4/0x590 fs/ext4/inode.c:517, CPU#0: syz.8.7999/132148
RIP: 0010:ext4_check_map_extents_env+0x4b4/0x590 fs/ext4/inode.c:517
```

With `CONFIG_EXT4_DEBUG=y`, `ext4_check_map_extents_env()` asserts that
the inode lock or the mapping's `invalidate_lock` is held when extents
are mapped. Neither condition is satisfied on the observed path. The
guest runs with `panic_on_warn=1`, which escalates this warning to a
kernel panic.

## Call trace and analysis

The warning is reached through the uprobe instruction-reading path
during `mmap()`:

```
mmap
  -> uprobe_mmap
    -> install_breakpoint
      -> prepare_uprobe
        -> copy_insn
          -> __copy_insn
            -> read_mapping_page
              -> read_cache_page
                -> filemap_read_folio
                  -> ext4_read_folio
                    -> ext4_mpage_readpages
                      -> ext4_map_blocks
                        -> ext4_check_map_extents_env
                          -> WARN_ON_ONCE
```

The uprobe subsystem attempts to retrieve instruction bytes from a
file-backed mapping. This initiates a page-cache read through ext4 and
eventually reaches `ext4_map_blocks()`.

With `CONFIG_EXT4_DEBUG=y`, `ext4_check_map_extents_env()` checks
whether the inode lock or the mapping's `invalidate_lock` is held.
Neither condition is satisfied on the observed path, causing
`WARN_ON_ONCE()` to trigger.

Two original syzkaller crash reports show the same execution path.

The warning demonstrates that the debug assertion is reached without
the expected locks. However, the available logs do not establish whether
the file-read path contains a harmful synchronization defect or whether
the assertion is too restrictive for this case.

Because `panic_on_warn=1` is enabled, the warning results in a kernel
panic.

Full crash reports: `logs/report-0.txt` and `logs/report-1.txt`.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_EXT4_FS=y,
CONFIG_EXT4_DEBUG=y, CONFIG_PROVE_LOCKING=y, booted with `oops=panic` and
`panic_on_warn=1`. See `environment.txt` for the exact QEMU command line
and `kernel.config` for the configuration.

The crash depends on ext4 state built up by the preceding programs. The
reproducer is a multi-program syzkaller execution log (105 programs):

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

The programs are replayed repeatedly with 8 parallel workers (`-repeat=0`);
the execution order is not guaranteed to be strictly serial.

Verified by the fuzzing pipeline's reproducer machinery on a fresh VM
(reproducer pipeline outcome: confirmed=true, status=verified). A C
reproducer has not been included yet; it will be added once syzkaller's
C-repro stage passes for this bug.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (105-program multi-program execution log)
- `reproducer/repro.prog` — pipeline-extracted isolated triggering testcase; has not reproduced this warning independently
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The assertion is in `ext4_check_map_extents_env()` — a debug check that
  extents mapping happens under the inode lock or invalidate_lock; the
  repro workload reaches `ext4_map_blocks()` via the uprobe
  instruction-reading path, which does not hold either lock.
- Could you advise whether the uprobe-triggered file-read path requires
  additional locking here, or whether this debug assertion should account
  for this particular execution path?
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
