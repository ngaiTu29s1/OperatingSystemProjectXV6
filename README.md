# Sleep & Wakeup Optimization — xv6-riscv

> **Đề tài 25110** · Tối ưu cơ chế sleep/wakeup trong xv6 kernel · RISC-V

## Tổng quan

Xv6 gốc sử dụng `wakeup()` duyệt **toàn bộ process table O(N)** mỗi lần cần đánh thức process. Project này thay thế bằng **hash-bucketed wait queue O(k)**, lấy cảm hứng từ Linux kernel `wait_queue_head_t`.

```
Trước (xv6 gốc):    wakeup(chan) → scan 64 process → acquire 64 locks
Sau (optimized):     wakeup(chan) → hash → scan ~4 entries → acquire 1 lock
```

## Kiến trúc

```
┌──────────────────────────────────────────────────┐
│             Wait Queue Hash Table (16 buckets)   │
│                                                   │
│  bucket[0]: [lock] → entry → entry → NULL        │
│  bucket[1]: [lock] → NULL                         │
│  ...                                              │
│  bucket[15]: [lock] → entry → NULL                │
└──────────────────────────────────────────────────┘
        ↑                          ↑
   sleep() O(1)              wakeup() O(k)
```

## Kỹ thuật áp dụng (từ Linux kernel)

| Kỹ thuật | Nguồn | Tác dụng |
|----------|-------|----------|
| Hash-bucketed wait queue | `linux/wait.h` | O(k) thay O(N) |
| `prepare_to_wait` pattern | `sched/wait.c` | Không lost wakeup |
| `wakeup_one()` + autoremove | `__wake_up_common()` | Tránh thundering herd |
| `wq_has_sleeper()` fast path | `fs/pipe.c` | Skip wakeup khi queue rỗng |
| Batch `copyin()` ngoài lock | `pipe_write()` | Giảm IPC latency |

> 📄 Chi tiết thiết kế: [docs/design.md](docs/design.md)

## Files thay đổi

| File | Thay đổi |
|------|----------|
| `kernel/waitqueue.h` | **NEW** — struct definitions |
| `kernel/waitqueue.c` | **NEW** — hash table + operations |
| `kernel/proc.h` | Thêm `wq_entry` vào `struct proc` |
| `kernel/proc.c` | Rewrite `sleep()`, `wakeup()`, thêm `wakeup_one()` |
| `kernel/pipe.c` | Fast path + batch copy + `wakeup_one()` |
| `kernel/defs.h` | Khai báo API mới |
| `user/bench_ipc.c` | **NEW** — IPC latency benchmark |
| `user/stress_wakeup.c` | **NEW** — Stress test 20+ process |

## Build & Run

```bash
# Build & boot
make clean
make TOOLPREFIX=riscv64-linux-gnu- qemu

# Trong xv6 shell
$ bench_ipc           # IPC latency benchmark
$ stress_wakeup       # Stress test

# Multi-core test
make CPUS=4 TOOLPREFIX=riscv64-linux-gnu- qemu
```

## Benchmark — Kỳ vọng lý thuyết

> Số liệu thực cần đo qua QEMU. Cải thiện thực tế phụ thuộc workload.

| Metric | Baseline | Optimized | Kỳ vọng |
|--------|----------|-----------|---------|
| `wakeup()` scan | O(64) | O(4) | ~16x lý thuyết |
| Spurious wakeups | Wake all | ≈ 0 | ~90%+ ↓ |
| Lock contention | 64 lock acquire | 1 lock acquire | ~16x lý thuyết |
| IPC latency | X ticks | < X ticks | Đo thực tế |

## Sprint Plan (8 tuần · 61 pts · 20 cards)

| Sprint | Tuần | Nội dung | Points |
|--------|------|----------|--------|
| 1 | 1–2 | Setup, đọc codebase, thiết kế struct | 10 |
| 2 | 3–4 | Implement wait queue, rewrite sleep/wakeup | 18 |
| 3 | 5–6 | wakeup_one, pipe optimization, benchmark | 20 |
| 4 | 7–8 | Fix bugs, báo cáo, demo | 13 |

<details>
<summary>📋 Danh sách 20 cards</summary>

| Card | Task | Sprint | Pts |
|------|------|--------|-----|
| S1-1 | Clone xv6, boot QEMU | 1 | 2 |
| S1-2 | Đọc `proc.c` — hiểu O(N) | 1 | 3 |
| S1-3 | Đọc Linux wait queue | 1 | 2 |
| S1-4 | Thiết kế `waitqueue.h` + lock ordering | 1 | 2 |
| S1-5 | Setup git repo + nhánh | 1 | 1 |
| S2-1 | Implement `waitqueue.c` | 2 | 5 |
| S2-2 | Rewrite `sleep()` | 2 | 4 |
| S2-3 | Rewrite `wakeup()` O(k) | 2 | 4 |
| S2-4 | Benchmark `bench_ipc.c` | 2 | 3 |
| S2-5 | Integration test: `usertests` | 2 | 2 |
| S3-1 | `wakeup_one()` | 3 | 3 |
| S3-2 | Pipe fast path + batch copy | 3 | 4 |
| S3-3 | Verify per-bucket lock | 3 | 2 |
| S3-4 | Verify subsystem (pipe, IDE, shell) | 3 | 3 |
| S3-5 | Benchmark so sánh trước/sau | 3 | 5 |
| S3-6 | Stress test 20+ process | 3 | 3 |
| S4-1 | Fix bugs | 4 | 3 |
| S4-2 | Benchmark report | 4 | 3 |
| S4-3 | Báo cáo kỹ thuật 4–6 trang | 4 | 3 |
| S4-4 | Demo script + 2 build | 4 | 2 |

</details>

## Tài liệu tham khảo

- [Linux `wait.h`](https://elixir.bootlin.com/linux/latest/source/include/linux/wait.h) · [Linux `sched/wait.c`](https://elixir.bootlin.com/linux/latest/source/kernel/sched/wait.c) · [Linux `fs/pipe.c`](https://elixir.bootlin.com/linux/latest/source/fs/pipe.c)
- [Columbia OS — Wait Queues Lecture](https://cs4118.github.io/www/2023-1/lect/10-run-wait-queues.html)
