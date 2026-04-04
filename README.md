# Sleep & Wakeup Optimization — xv6-riscv

> **Đề tài 25110** · Kernel-level wait queue optimization · RISC-V

---

## Mục lục

1. [Tổng quan vấn đề](#1-tổng-quan-vấn-đề)
2. [Kiến trúc giải pháp](#2-kiến-trúc-giải-pháp)
3. [Kỹ thuật tối ưu áp dụng](#3-kỹ-thuật-tối-ưu-áp-dụng)
4. [Thiết kế chi tiết](#4-thiết-kế-chi-tiết)
5. [Sprint Plan](#5-sprint-plan)
6. [Benchmark & Verification](#6-benchmark--verification)
7. [Hướng dẫn Build & Demo](#7-hướng-dẫn-build--demo)
8. [Tài liệu tham khảo](#8-tài-liệu-tham-khảo)

---

## 1. Tổng quan vấn đề

### 1.1. Xv6 gốc: `wakeup()` scan O(N)

```c
// kernel/proc.c — xv6 gốc
void wakeup(void *chan) {
    struct proc *p;
    for(p = proc; p < &proc[NPROC]; p++) {   // ← Duyệt TẤT CẢ 64 process
        acquire(&p->lock);
        if(p->state == SLEEPING && p->chan == chan)
            p->state = RUNNABLE;
        release(&p->lock);
    }
}
```

| Vấn đề | Mô tả | Ảnh hưởng |
|---------|--------|-----------|
| **O(N) scan** | `wakeup()` duyệt toàn bộ `proc[NPROC]` mỗi lần | CPU waste, tăng latency |
| **Thundering herd** | Wake **tất cả** process trên cùng channel | Spurious wakeup, lock contention |
| **Lock hold time lớn** | `pipewrite()` giữ lock suốt vòng lặp, gọi `copyin()` trong lock | Block reader, tăng IPC latency |
| **Không có fast path** | Luôn acquire N lock dù không ai sleeping | Waste cycles khi queue rỗng |

### 1.2. Mục tiêu

| Yêu cầu đề bài | Mục tiêu cụ thể | Metric |
|-----------------|------------------|--------|
| Better performance | `wakeup()` O(k) thay O(N) | k ≪ N |
| Efficient blocking | Wait queue + `prepare_to_wait` pattern | Không lost wakeup |
| Redesign sleep queues | Hash-bucketed wait queue, embedded entry | O(1) enqueue |
| Reduce wakeups | `wakeup_one()` — chỉ wake 1 process | Spurious rate ↓ |
| IPC latency reduction | Fast path check + giảm lock hold time | Pipe throughput ↑ |

---

## 2. Kiến trúc giải pháp

### 2.1. Tổng quan kiến trúc

```
┌─────────────────────────────────────────────────────────┐
│                    KERNEL SPACE                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │          Wait Queue Hash Table (NWQUEUE=17)     │    │
│  │                                                  │    │
│  │  bucket[0]: [lock] → entry → entry → NULL       │    │
│  │  bucket[1]: [lock] → NULL                        │    │
│  │  bucket[2]: [lock] → entry → NULL                │    │
│  │  ...                                             │    │
│  │  bucket[16]: [lock] → entry → entry → NULL       │    │
│  └─────────────────────────────────────────────────┘    │
│           ↑                       ↑                     │
│      sleep() enqueue         wakeup() lookup            │
│      O(1) prepend            O(1) hash + O(k) scan      │
│                                                         │
│  ┌─────────┐   ┌────────────┐   ┌──────────────────┐   │
│  │ sleep() │   │ wakeup()   │   │ wakeup_one()     │   │
│  │ 4-step  │   │ O(k) scan  │   │ wake 1, stop     │   │
│  │ pattern │   │ per-bucket │   │ anti-thundering   │   │
│  └─────────┘   └────────────┘   └──────────────────┘   │
│                                                         │
│  ┌─────────────────┐   ┌───────────────────────────┐   │
│  │ wq_has_sleeper() │   │ Pipe batch copy           │   │
│  │ fast path check  │   │ copyin() ngoài lock       │   │
│  └─────────────────┘   └───────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 2.2. Files thay đổi

| File | Thay đổi | Loại |
|------|----------|------|
| `kernel/waitqueue.h` | Định nghĩa struct + API | **NEW** |
| `kernel/waitqueue.c` | Implement hash table + operations | **NEW** |
| `kernel/proc.h` | Thêm `wq_entry` vào `struct proc` | MODIFY |
| `kernel/proc.c` | Rewrite `sleep()`, `wakeup()`, thêm `wakeup_one()` | MODIFY |
| `kernel/pipe.c` | Fast path + batch copy + `wakeup_one()` | MODIFY |
| `kernel/defs.h` | Khai báo API mới | MODIFY |
| `user/bench_ipc.c` | IPC latency benchmark | **NEW** |
| `user/stress_wakeup.c` | Stress test 20+ process | **NEW** |

---

## 3. Kỹ thuật tối ưu áp dụng

Các kỹ thuật được học từ Linux kernel và adapt cho xv6:

| # | Kỹ thuật | Nguồn Linux | Áp dụng trong Xv6 | Sprint |
|---|----------|-------------|-------------------|--------|
| 1 | Per-object wait queue, O(1) enqueue | `include/linux/wait.h` | Hash-bucketed `wq_bucket` với embedded `wq_entry` | S2 |
| 2 | `prepare_to_wait` pattern 4 bước | `kernel/sched/wait.c` | Enqueue → set state → schedule → cleanup trong `sleep()` | S2 |
| 3 | `WQ_FLAG_EXCLUSIVE` / selective wakeup | `__wake_up_common()` | `wakeup_one()` — chỉ wake 1 process | S3 |
| 4 | `wq_has_sleeper()` fast path | `fs/pipe.c` | Skip wakeup khi queue rỗng, tránh acquire lock thừa | S3 |
| 5 | Giảm lock hold time | `pipe_write()`/`pipe_read()` | Batch `copyin()` ra ngoài lock trong `pipewrite()` | S3 |

> **Kỹ thuật bỏ qua:** Doubly-linked list (`list_head`) — xv6 chỉ có tối đa 64 process, singly-linked đủ nhanh, thêm `prev` pointer chỉ tăng complexity mà không cải thiện đáng kể.

---

## 4. Thiết kế chi tiết

### 4.1. Data Structures — `kernel/waitqueue.h`

```c
#ifndef _WAITQUEUE_H_
#define _WAITQUEUE_H_

#define NWQUEUE 16    // số bucket — power-of-2 cho phép dùng bitmask thay modulo

struct wq_entry {
    struct proc     *proc;      // process đang ngủ
    void            *chan;      // channel đang chờ
    struct wq_entry *next;      // linked list trong bucket
};

struct wq_bucket {
    struct spinlock  lock;      // per-bucket lock — fine-grained
    struct wq_entry *head;      // head of linked list
};

struct waitqueue_table {
    struct wq_bucket buckets[NWQUEUE];
};

#endif
```

**Thêm vào `struct proc` (kernel/proc.h):**

```c
struct proc {
    // ... existing fields ...
    struct wq_entry wq_entry;   // embedded — 1 entry/proc, không cần kalloc()
};
```

> **Lý do embedded:** Mỗi process chỉ sleep trên 1 channel tại một thời điểm → 1 entry/proc là đủ. Tránh `kalloc()` (4KB/page quá lớn cho 24 bytes), không memory leak.

### 4.2. Lock Ordering — BẮT BUỘC

```
caller's lock (lk)  →  bucket lock  →  p->lock
        ①                   ②              ③
```

Thực tế là **3 tầng lock**, vì caller (pipe, log, ...) giữ `lk` khi gọi `sleep(chan, lk)`:
- `sleep()`: caller giữ `lk` → acquire bucket lock → acquire `p->lock` → release `lk` (bên trong)
- `wakeup()`: caller giữ `lk` → acquire bucket lock → acquire `p->lock`

> **Deadlock rule:** Không bao giờ acquire bucket lock khi đang giữ `p->lock`. Không bao giờ acquire `lk` khi đang giữ bucket lock (trừ khi đã release bucket lock trước).

### 4.3. Core API — `kernel/waitqueue.c`

```c
extern struct waitqueue_table wt;  // global wait queue table

// Hash function — bitmask thay modulo vì NWQUEUE là power-of-2
int wq_hash(void *chan) {
    return ((uint64)chan >> 3) & (NWQUEUE - 1);  // >> 3: bỏ 3 bit alignment
}

// Init tất cả bucket locks
void wq_init(void) {
    for(int i = 0; i < NWQUEUE; i++) {
        initlock(&wt.buckets[i].lock, "wq_bucket");
        wt.buckets[i].head = 0;
    }
}

// O(1) prepend — gọi khi ĐÃ giữ bucket lock
void wq_add_locked(struct wq_bucket *b, void *chan, struct proc *p) {
    p->wq_entry.proc = p;
    p->wq_entry.chan = chan;
    p->wq_entry.next = b->head;
    b->head = &p->wq_entry;
}

// Dequeue — gọi bên trong wakeup() khi set RUNNABLE (autoremove pattern)
// Caller PHẢI giữ bucket lock
static void wq_dequeue_locked(struct wq_bucket *b, struct wq_entry *entry) {
    struct wq_entry **pp = &b->head;
    while(*pp) {
        if(*pp == entry) {
            *pp = entry->next;
            entry->next = 0;
            return;
        }
        pp = &(*pp)->next;
    }
}

// Fast path — skip wakeup nếu bucket rỗng (Kỹ thuật #4)
// Lưu ý: kiểm tra bucket, không lọc theo chan → có thể false positive
// (bucket có entry nhưng không phải chan này). Không ảnh hưởng correctness
// vì wakeup_one() vẫn lọc đúng chan — chỉ tốn 1 lần acquire lock thừa.
static inline int wq_has_sleeper(void *chan) {
    int idx = wq_hash(chan);
    return __atomic_load_n(&wt.buckets[idx].head, __ATOMIC_ACQUIRE) != 0;
}
```

### 4.4. Rewritten `sleep()` — `prepare_to_wait` pattern

```c
void sleep(void *chan, struct spinlock *lk) {
    struct proc *p = myproc();
    int idx = wq_hash(chan);
    struct wq_bucket *b = &wt.buckets[idx];

    // Bước 1: prepare_to_wait — enqueue
    acquire(&b->lock);              // bucket lock TRƯỚC (lock ordering ✓)
    release(lk);                    // safe: wakeup cũng cần bucket lock
    wq_add_locked(b, chan, p);      // O(1) prepend

    // Bước 2: set state + schedule
    acquire(&p->lock);              // p->lock SAU (lock ordering ✓)
    p->chan = chan;
    p->state = SLEEPING;
    release(&b->lock);              // sched() chỉ cho giữ p->lock
    sched();                        // yield CPU

    // Bước 3: finish_wait — cleanup
    // wakeup() đã dequeue entry khỏi queue (autoremove pattern)
    // → không cần gọi wq_remove() ở đây
    p->chan = 0;
    release(&p->lock);
    acquire(lk);                    // re-acquire caller's lock
}
```

**Tại sao không lost wakeup:**
- Bước 1: giữ bucket lock → `wakeup()` cũng cần bucket lock → phải chờ
- Bước 2: giữ `p->lock` → `wakeup()` cần `p->lock` để set RUNNABLE → phải chờ
- Khi `sched()` release `p->lock` → `wakeup()` thấy `SLEEPING` → set `RUNNABLE` + dequeue ✓

**Tại sao autoremove (dequeue trong wakeup) thay vì self-remove (dequeue trong sleep):**
- Self-remove: entry tồn tại trong queue sau khi process đã RUNNABLE → `wq_has_sleeper()` thấy false positive → gọi `wakeup_one()` thừa → spurious wakeup
- Autoremove: entry bị xóa ngay khi set RUNNABLE → queue sạch → fast path chính xác hơn

### 4.5. Rewritten `wakeup()` — O(k) + autoremove

```c
void wakeup(void *chan) {
    int idx = wq_hash(chan);
    struct wq_bucket *b = &wt.buckets[idx];

    acquire(&b->lock);
    struct wq_entry *e = b->head;
    while(e) {
        struct wq_entry *next = e->next;    // lưu next trước (dequeue sẽ sửa next)
        if(e->chan == chan) {
            acquire(&e->proc->lock);        // lock ordering ✓
            if(e->proc->state == SLEEPING) {
                e->proc->state = RUNNABLE;
                wq_dequeue_locked(b, e);    // ← autoremove: xóa ngay khỏi queue
            }
            release(&e->proc->lock);
        }
        e = next;
    }
    release(&b->lock);
}
```

### 4.6. `wakeup_one()` — Anti-thundering herd (Kỹ thuật #3)

```c
void wakeup_one(void *chan) {
    int idx = wq_hash(chan);
    struct wq_bucket *b = &wt.buckets[idx];

    acquire(&b->lock);
    struct wq_entry *e = b->head;
    while(e) {
        if(e->chan == chan) {
            acquire(&e->proc->lock);
            if(e->proc->state == SLEEPING) {
                e->proc->state = RUNNABLE;
                wq_dequeue_locked(b, e);    // ← autoremove
                release(&e->proc->lock);
                break;                      // chỉ wake 1, dừng ngay
            }
            release(&e->proc->lock);
        }
        e = e->next;
    }
    release(&b->lock);
}
```

> **Yêu cầu:** Caller phải có retry loop. Xv6 pipe read/write đã có `while(condition)` → an toàn.

### 4.7. Pipe optimization — `kernel/pipe.c` (Kỹ thuật #4 + #5)

```c
int pipewrite(struct pipe *pi, uint64 addr, int n) {
    int total = 0;
    struct proc *pr = myproc();
    char buf[PIPESIZE];     // buffer tạm trên kernel stack (512 bytes — OK)

    while(total < n) {
        // Batch copy: lấy data từ user space TRƯỚC khi acquire lock (Kỹ thuật #5)
        // Mỗi vòng copy tối đa PIPESIZE bytes, loop lại nếu n > PIPESIZE
        int chunk = n - total;
        if(chunk > PIPESIZE) chunk = PIPESIZE;
        if(copyin(pr->pagetable, buf, addr + total, chunk) == -1)
            break;

        acquire(&pi->lock);
        int i = 0;
        while(i < chunk) {
            if(pi->readopen == 0 || killed(pr)) {
                release(&pi->lock);
                return total > 0 ? total : -1;
            }
            if(pi->nwrite == pi->nread + PIPESIZE) {
                // Fast path check trước khi wakeup (Kỹ thuật #4)
                if(wq_has_sleeper(&pi->nread))
                    wakeup_one(&pi->nread);     // Kỹ thuật #3
                sleep(&pi->nwrite, &pi->lock);
            } else {
                pi->data[pi->nwrite++ % PIPESIZE] = buf[i++];
            }
        }
        if(wq_has_sleeper(&pi->nread))
            wakeup_one(&pi->nread);
        release(&pi->lock);
        total += i;
    }
    return total;
}

int piperead(struct pipe *pi, uint64 addr, int n) {
    int i;
    struct proc *pr = myproc();
    char ch;

    acquire(&pi->lock);
    while(pi->nread == pi->nwrite && pi->writeopen) {
        if(killed(pr)) {
            release(&pi->lock);
            return -1;
        }
        sleep(&pi->nread, &pi->lock);
    }
    for(i = 0; i < n; i++) {
        if(pi->nread == pi->nwrite)
            break;
        ch = pi->data[pi->nread % PIPESIZE];
        if(copyout(pr->pagetable, addr + i, &ch, 1) == -1) {
            if(i == 0) i = -1;
            break;
        }
        pi->nread++;
    }
    if(wq_has_sleeper(&pi->nwrite))
        wakeup_one(&pi->nwrite);
    release(&pi->lock);
    return i;
}
```

---

## 5. Sprint Plan

### Sprint 1 — Setup & Đọc codebase (Tuần 1–2) · 10 pts

| Card | Task | Points |
|------|------|--------|
| S1-1 | Clone xv6-riscv, boot trên QEMU | 2 |
| S1-2 | Đọc `proc.c`: `sleep()`, `wakeup()`, `scheduler()` — hiểu O(N) | 3 |
| S1-3 | Đọc Linux wait queue, tóm tắt thundering herd problem | 2 |
| S1-4 | Thiết kế `waitqueue.h` — thống nhất struct + lock ordering | 2 |
| S1-5 | Setup git repo: nhánh `kernel/waitqueue` + `tool/benchmark` | 1 |

**Definition of Done:** Xv6 boot được, cả 2 hiểu vấn đề, struct đã thống nhất.

<details>
<summary>Chi tiết Card S1-1</summary>

```
Goal: xv6 boot thành công, thấy shell $ trong QEMU

Việc cần làm:
- sudo dnf install qemu-system-riscv gcc-riscv64-linux-gnu
- git clone https://github.com/mit-pdos/xv6-riscv
- make TOOLPREFIX=riscv64-linux-gnu- qemu

Done when: Boot được xv6, thấy $ prompt
```
</details>

<details>
<summary>Chi tiết Card S1-2</summary>

```
Goal: Hiểu rõ vấn đề O(N) scan của wakeup() hiện tại

Việc cần làm:
- Đọc kernel/proc.c: hàm sleep(), wakeup(), scheduler()
- Đọc kernel/proc.h: struct proc, enum procstate
- Đọc kernel/spinlock.c: cách dùng spinlock
- Đọc xv6 book Chapter 7 (Scheduling)
- Vẽ sơ đồ flow: process RUNNABLE → SLEEPING → RUNNABLE

Done when: Giải thích được tại sao wakeup() hiện tại là O(N)
```
</details>

<details>
<summary>Chi tiết Card S1-3</summary>

```
Goal: Hiểu cấu trúc wait queue của Linux để làm reference

Việc cần làm:
- Đọc Linux wait.h, sched/wait.c
- Tóm tắt: wait_queue_head_t, prepare_to_wait, finish_wait
- Hiểu: thundering herd, WQ_FLAG_EXCLUSIVE
- Tóm tắt: wq_has_sleeper() fast path trong fs/pipe.c

Done when: Viết được 1 trang giải thích Linux waitqueue
```
</details>

<details>
<summary>Chi tiết Card S1-4</summary>

```
Goal: Định nghĩa struct dùng chung trước khi code

Struct cần thống nhất: xem mục 4.1 (waitqueue.h)
Lock ordering: xem mục 4.2
Lý do embedded wq_entry: xem mục 4.1

Done when: Cả 2 ký duyệt struct + lock ordering, commit waitqueue.h
```
</details>

---

### Sprint 2 — Implement wait queue core (Tuần 3–4) · 18 pts

| Card | Task | Points |
|------|------|--------|
| S2-1 | Implement `waitqueue.c`: hash table, `wq_init/add/remove` | 5 |
| S2-2 | Rewrite `sleep()` với wait queue — `prepare_to_wait` pattern | 4 |
| S2-3 | Rewrite `wakeup()` — O(k) scan per-bucket | 4 |
| S2-4 | Viết `bench_ipc.c`: pipe latency benchmark cơ bản | 3 |
| S2-5 | Integration test: `usertests` pass, không deadlock | 2 |

**Definition of Done:** Wait queue hoạt động, `sleep()`/`wakeup()` dùng bucket, `usertests` PASS.

<details>
<summary>Chi tiết Card S2-1</summary>

```
File: kernel/waitqueue.h, kernel/waitqueue.c (tạo mới)

Implement:
- wq_init(): khởi tạo tất cả bucket + lock
- wq_hash(chan): (uint64)chan % NWQUEUE
- wq_add_locked(bucket, chan, proc): O(1) prepend
- wq_remove(wt, chan, proc): xóa entry khỏi bucket
- wq_has_sleeper(chan): inline fast path check
- Thêm #include "waitqueue.h" vào defs.h

⚠️ KHÔNG dùng kalloc() — dùng embedded wq_entry trong struct proc

Done when: Compile được, test bằng GDB thấy add/remove đúng bucket
```
</details>

<details>
<summary>Chi tiết Card S2-2</summary>

```
File: kernel/proc.c
Xem thiết kế: mục 4.4 (Rewritten sleep)

⚠️ Critical: tuân thủ lock ordering bucket → p->lock
⚠️ Critical: sched() yêu cầu chỉ giữ đúng 1 lock (p->lock)

Done when: sleep() compile được, không deadlock, không lost wakeup
```
</details>

<details>
<summary>Chi tiết Card S2-3</summary>

```
File: kernel/proc.c
Xem thiết kế: mục 4.5 (Rewritten wakeup)

Complexity: O(1) hash + O(k) scan — k = số entry trong bucket
KHÔNG xóa entry trong wakeup — process tự xóa trong sleep()

Done when: wakeup() hoạt động đúng, lock ordering nhất quán
```
</details>

<details>
<summary>Chi tiết Card S2-4</summary>

```
File: user/bench_ipc.c (tạo mới)

Code:
  int main() {
      int fds[2]; pipe(fds);
      int pid = fork();
      if(pid == 0) {
          // child: đọc
          char buf[1];
          for(int i = 0; i < 10000; i++) read(fds[0], buf, 1);
          exit(0);
      }
      // parent: ghi + đo
      uint64 start = uptime();
      for(int i = 0; i < 10000; i++) write(fds[1], "x", 1);
      uint64 end = uptime();
      wait(0);
      printf("IPC: %d ticks / 10000 ops\n", end - start);
      exit(0);
  }

Thêm vào Makefile: $U/_bench_ipc

Done when: Chạy được trong xv6, in ra số ticks
```
</details>

---

### Sprint 3 — Optimize + Verify + Benchmark (Tuần 5–6) · 20 pts

| Card | Task | Points |
|------|------|--------|
| S3-1 | Implement `wakeup_one()` — anti-thundering herd | 3 |
| S3-2 | Pipe optimization: `wq_has_sleeper()` fast path + batch copy | 4 |
| S3-3 | Verify per-bucket lock: 2 channel khác nhau không block | 2 |
| S3-4 | Verify toàn bộ subsystem: pipe, IDE, wait/exit, shell | 3 |
| S3-5 | Benchmark so sánh trước/sau: 5, 10, 20, 50 process | 5 |
| S3-6 | Stress test: 20+ process sleep/wakeup, CPUS=1 và CPUS=4 | 3 |

**Definition of Done:** `wakeup_one()` hoạt động, pipe optimized, có bảng benchmark trước/sau.

<details>
<summary>Chi tiết Card S3-1</summary>

```
File: kernel/proc.c, kernel/defs.h
Xem thiết kế: mục 4.6

Thay wakeup() bằng wakeup_one() ở:
- pipewrite(): wakeup(&pi->nread) → wakeup_one(&pi->nread)
- piperead(): wakeup(&pi->nwrite) → wakeup_one(&pi->nwrite)

⚠️ Giữ nguyên wakeup() ở: exit() → wakeup(parent), reparent() → wakeup(initproc)
   (Những chỗ này cần wake tất cả)

Done when: Pipe dùng wakeup_one(), không wake thừa, có retry loop
```
</details>

<details>
<summary>Chi tiết Card S3-2</summary>

```
File: kernel/pipe.c
Xem thiết kế: mục 4.7

2 optimization:
1. wq_has_sleeper() fast path: skip wakeup khi queue rỗng
2. Batch copyin(): copy data vào buffer tạm TRƯỚC acquire lock

⚠️ Buffer trên stack: char buf[512] = 512 bytes trên kernel stack (4KB) — OK

Done when: pipewrite tối ưu, lock hold time giảm
```
</details>

<details>
<summary>Chi tiết Card S3-5</summary>

```
File: user/bench_compare.c hoặc script

Việc cần làm:
- Chạy bench_ipc trên xv6 GỐC → ghi baseline
- Chạy bench_ipc trên xv6 OPTIMIZED → ghi kết quả
- Test với 5, 10, 20, 50 process đồng thời
- Tính % cải thiện

Output:
| Số process | Baseline (ticks) | Optimized (ticks) | Cải thiện % |
|------------|------------------|-------------------|-------------|
| 5          | ???              | ???               | ???         |
| 10         | ???              | ???               | ???         |
| 20         | ???              | ???               | ???         |
| 50         | ???              | ???               | ???         |

Done when: Có bảng kết quả đo thực tế từ QEMU
```
</details>

<details>
<summary>Chi tiết Card S3-6</summary>

```
File: user/stress_wakeup.c (tạo mới)

Viết program: fork 20 process, mỗi process sleep/wakeup 100 lần
Kiểm tra: không panic, không deadlock, không lost wakeup
Chạy 5 lần liên tiếp

⚠️ BẮT BUỘC test multi-core: make CPUS=4 qemu
   (nhiều bug lock chỉ xuất hiện khi multi-CPU)

Done when: 5 lần chạy không crash, cả CPUS=1 và CPUS=4
```
</details>

---

### Sprint 4 — Polish & Demo (Tuần 7–8) · 13 pts

| Card | Task | Points |
|------|------|--------|
| S4-1 | Fix bugs từ stress test Sprint 3 | 3 |
| S4-2 | Hoàn thiện benchmark report với số liệu + biểu đồ | 3 |
| S4-3 | Viết báo cáo kỹ thuật (4–6 trang) | 3 |
| S4-4 | Chuẩn bị demo script: 2 build trước/sau, chạy 5 phút | 2 |
| S4-5 | Code review & cleanup: no warnings, comment đầy đủ | 2 |

**Definition of Done:** Bug-free, demo chạy trơn tru, có báo cáo hoàn chỉnh.

<details>
<summary>Chi tiết Card S4-3: Cấu trúc báo cáo</summary>

```
1. Vấn đề: tại sao xv6 sleep/wakeup kém hiệu quả
2. Giải pháp: thiết kế wait queue, 5 kỹ thuật tối ưu
3. Implementation: các thay đổi trong kernel (có code diff)
4. Kết quả: bảng so sánh benchmark + phân tích
5. Kết luận: bài học, hạn chế, hướng phát triển

Done when: Báo cáo 4–6 trang, đủ 5 phần
```
</details>

<details>
<summary>Chi tiết Card S4-4: Kịch bản demo 5 phút</summary>

```
1. Boot xv6 gốc     → chạy bench_ipc → ghi số (30s)
2. Giải thích O(N) problem (1 phút)
3. Boot xv6 optimized → chạy bench_ipc → ghi số (30s)
4. So sánh 2 số trực tiếp (30s)
5. Chạy stress_wakeup: 20 process, không crash (1 phút)
6. Show code diff: sleep(), wakeup(), pipe.c (1.5 phút)
```
</details>

---

## 6. Benchmark & Verification

### 6.1. Test Suite

| Test | Mục đích | Command |
|------|----------|---------|
| `usertests` | Regression — không break kernel | `$ usertests` |
| `bench_ipc` | Đo IPC pipe latency | `$ bench_ipc` |
| `stress_wakeup` | Race condition, deadlock | `$ stress_wakeup` |
| Pipe pipeline | Functional correctness | `echo "hello" \| cat` |
| Multi-core | Lock ordering bugs | `make CPUS=4 qemu` |

### 6.2. Kỳ vọng kết quả (lý thuyết)

> **Lưu ý:** Các con số dưới đây là **kỳ vọng lý thuyết**, không phải số đo thực. Trong thực tế xv6 với workload nhẹ trên QEMU, cải thiện sẽ thấp hơn vì hầu hết bucket rỗng và xv6 gốc chỉ acquire lock khi `p->state == SLEEPING`. Số liệu thực cần đo qua benchmark.

| Metric | Baseline (xv6 gốc) | Optimized | Kỳ vọng lý thuyết |
|--------|--------------------:|----------:|-------------------:|
| `wakeup()` complexity | O(N) = O(64) | O(k), k = 64/16 ≈ 4 | ~16x lý thuyết |
| Spurious wakeups (pipe) | Wake tất cả | ≈ 0 (wakeup_one + autoremove) | ~90%+ ↓ |
| IPC latency (1-byte ops) | Baseline X ticks | < X ticks | Đo thực tế |
| IPC throughput (512-byte ops) | Baseline Y ticks | < Y ticks | Đo thực tế |
| Lock contention | Global scan, N lock acquire | Per-bucket, 1 lock | ~16x lý thuyết |

### 6.3. Benchmark bổ sung — Throughput test

Ngoài `bench_ipc` đo latency (write 1 byte), nên thêm test throughput (write 512 bytes/lần) để thấy profile khác nhau:

```c
// Trong bench_ipc.c — thêm mode throughput
char bigbuf[512];
for(int i = 0; i < 1000; i++) {
    write(fds[1], bigbuf, 512);   // throughput test
}
```

---

## 7. Hướng dẫn Build & Demo

### 7.1. Build xv6 gốc (baseline)

```bash
git checkout main
make clean
make TOOLPREFIX=riscv64-linux-gnu- qemu
# Trong xv6 shell:
$ bench_ipc
```

### 7.2. Build xv6 optimized

```bash
git checkout kernel/waitqueue
make clean
make TOOLPREFIX=riscv64-linux-gnu- qemu
# Trong xv6 shell:
$ bench_ipc
$ stress_wakeup
```

### 7.3. Multi-core test

```bash
make CPUS=4 TOOLPREFIX=riscv64-linux-gnu- qemu
```

---

## 8. Tài liệu tham khảo

| Tài liệu | Link |
|-----------|------|
| Linux `include/linux/wait.h` | [Source](https://elixir.bootlin.com/linux/latest/source/include/linux/wait.h) |
| Linux `kernel/sched/wait.c` | [Source](https://elixir.bootlin.com/linux/latest/source/kernel/sched/wait.c) |
| Linux `__wake_up_common()` | [Source](https://elixir.bootlin.com/linux/latest/source/kernel/sched/wait.c) |
| Linux `fs/pipe.c` | [Source](https://elixir.bootlin.com/linux/latest/source/fs/pipe.c) |
| Columbia OS Wait Queues | [Lecture](https://cs4118.github.io/www/2023-1/lect/10-run-wait-queues.html) |

---

## Tóm tắt Sprint

| Sprint | Tuần | Points | Cards |
|--------|------|--------|-------|
| 1 — Setup & Codebase | 1–2 | 10 pts | S1-1 → S1-5 |
| 2 — Wait Queue Core | 3–4 | 18 pts | S2-1 → S2-5 |
| 3 — Optimize & Verify | 5–6 | 20 pts | S3-1 → S3-6 |
| 4 — Polish & Demo | 7–8 | 13 pts | S4-1 → S4-5 |
| **Tổng** | **8 tuần** | **61 pts** | **20 cards** |

---

## Danh sách Cards (20 cards)

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