# Walkthrough: VirtIO Queue Notification Counter & Tracing

This document provides a comprehensive walkthrough of the fixes, implementation details, and verification results for Tasks 1 through 4.

---

## 1. Problem Diagnosis & Root Causes

During initial testing, the trace log showed `notify_count: 0` for all trace events, and the guest kernel failed to boot.

1. **Counter Stuck at 0 (`notify_count: 0`)**:
   - Modern virtio devices (such as `virtio-blk-pci`) utilize KVM `ioeventfd` by default (`vq->host_notifier_enabled = true`).
   - When the guest driver kicks a virtqueue, the write to the doorbell is handled directly by KVM, signaling the eventfd.
   - QEMU's event loop receives the notification in `virtio_queue_host_notifier_read()` and invokes `virtio_queue_notify_vq(vq)`.
   - In `virtio_queue_notify_vq(vq)`, `vq->notify_count` was passed to the trace event without being incremented (`vq->notify_count++` was only present in the legacy MMIO fallback path `virtio_queue_notify()`).
   - Furthermore, incrementing in both places without checking `vq->host_notifier_enabled` would cause double-counting when guest kicks fell back through `virtio_queue_notify()`.

2. **Guest Kernel Boot Failure (`error -8` / SeaBIOS Boot Loops)**:
   - In the guest initramfs (`initramfs-virtio-trace.cpio.gz`), `/init` had an invalid shebang line (`#\!/bin/sh` with a backslash), causing the Linux kernel's `binfmt_script` to reject it with `-ENOEXEC` (`-8`).
   - Inside `/home/student/Desktop/nishchay/virtio-trace-root/bin/`, `busybox` was a circular symlink pointing to `/bin/busybox`.
   - `/dev/zero` did not exist because `devtmpfs` had not been mounted.

---

## 2. Changes Made

### A. QEMU Notification Counter & Trace Call Fixes
- [hw/virtio/virtio.c](file:///home/student/Desktop/nishchay/qemu/qemu/hw/virtio/virtio.c#L2512-L2554):
  - Updated `virtio_queue_notify_vq()` to increment `vq->notify_count++` before emitting `trace_virtio_queue_notify(...)` and running `vq->handle_output()`.
  - Updated `virtio_queue_notify()` to only increment `vq->notify_count++` and trace when `host_notifier_enabled` is false (direct MMIO/PIO path), preventing double increments.
  - Implemented `virtio_queue_get_notify_count(VirtQueue *vq)` getter.
- [include/hw/virtio/virtio.h](file:///home/student/Desktop/nishchay/qemu/qemu/include/hw/virtio/virtio.h#L504):
  - Exported `uint64_t virtio_queue_get_notify_count(VirtQueue *vq);`.
- [hw/virtio/trace-events](file:///home/student/Desktop/nishchay/qemu/qemu/hw/virtio/trace-events#L84):
  - Preserved the trace event signature:
    `virtio_queue_notify(void *vdev, int n, void *vq, uint64_t notify_count) "vdev %p n %d vq %p notify_count: %" PRIu64`

### B. Guest Workload & Initramfs Fixes
- Copied the static host `/bin/busybox` binary to `virtio-trace-root/bin/busybox`.
- Fixed `/init` shebang to `#!/bin/sh`.
- Added `mount -t devtmpfs devtmpfs /dev` so `/dev/zero`, `/dev/null`, and `/dev/vda` are automatically populated.
- Repacked `initramfs-virtio-trace.cpio.gz`.
- Created a repeatable runner script: [`run_virtio_blk_workload.sh`](file:///home/student/Desktop/nishchay/run_virtio_blk_workload.sh).

---

## 3. Verification & Results

### Task 1 & 2: Trace Event Registration
Verified using the newly built binary:
```bash
$ ./build/qemu-system-x86_64 -trace help | grep virtio_queue_notify
virtio_queue_notify
```

### Task 3 & 4: Executing QEMU & Repeatable Workload
Ran `/home/student/Desktop/nishchay/run_virtio_blk_workload.sh`:
- Guest boots Linux kernel `vmlinuz-host` with `initramfs-virtio-trace.cpio.gz`.
- Detects `/dev/vda` (VirtIO block device, major 253, minor 0).
- Executes 5 passes of direct I/O writes using `dd if=/dev/zero of=/dev/vda bs=1M count=8 oflag=direct`.
- Powers down cleanly via `reboot: Power down`.

Guest console output:
```text
=== Devices in /dev ===
crw-rw-rw-    1 0        0           1,   3 Sep  8 07:46 /dev/null
brw-------    1 0        0         253,   0 Sep  8 07:46 /dev/vda
crw-rw-rw-    1 0        0           1,   5 Sep  8 07:46 /dev/zero
virtio workload start
pass 1 done
pass 2 done
pass 3 done
pass 4 done
pass 5 done
virtio workload done
[    2.993649] reboot: Power down
```

### Monotonic Notification Counter Verification (`trace.log`)
Trace output from `qemu/trace.log`:
```text
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 1
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 1
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 2
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 3
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 4
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 5
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 6
...
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 43
virtio_queue_notify vdev 0x61de5a1d22a0 n 0 vq 0x61de5a211120 notify_count: 44
Total VirtQueue notifications recorded: 45
```
*(Note: SeaBIOS performs 1 notification during initial probing before handoff; the Linux kernel then resets the device, resetting `notify_count` to 0, after which it monotonically increments 1 through 44 during driver init and disk I/O workload).*

---

## 4. How to Reproduce Anytime

To run the workload and inspect the trace logs again, execute:
```bash
/home/student/Desktop/nishchay/run_virtio_blk_workload.sh
```
