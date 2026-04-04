# Thiết kế chi tiết — Sleep & Wakeup Optimization

> Tài liệu kỹ thuật nội bộ. Xem [README.md](../README.md) để tổng quan project.

---

## 1. Vấn đề hiện tại

### Xv6 gốc: `wakeup()` scan O(N)

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

---

## 2. Kỹ thuật tối ưu áp dụng

Học từ Linux kernel, adapt cho xv6:

| # | Kỹ thuật | Nguồn Linux | Áp dụng trong Xv6 |
|---|----------|-------------|-------------------|
| 1 | Per-object wait queue, O(1) enqueue | `include/linux/wait.h` | Hash-bucketed `wq_bucket` với embedded `wq_entry` |
| 2 | `prepare_to_wait` pattern 4 bước | `kernel/sched/wait.c` | Enqueue → set state → schedule → cleanup trong `sleep()` |
| 3 | `WQ_FLAG_EXCLUSIVE` / selective wakeup | `__wake_up_common()` | `wakeup_one()` — chỉ wake 1 process |
| 4 | `wq_has_sleeper()` fast path | `fs/pipe.c` | Skip wakeup khi bucket rỗng, tránh acquire lock thừa |
| 5 | Giảm lock hold time | `pipe_write()`/`pipe_read()` | Batch `copyin()` ra ngoài lock trong `pipewrite()` |

> **Bỏ qua:** Doubly-linked list (`list_head`) — xv6 chỉ có tối đa 64 process, singly-linked đủ nhanh.

---

## 3. Data Structures — `kernel/waitqueue.h`

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

---

## 4. Lock Ordering — BẮT BUỘC

```
caller's lock (lk)  →  bucket lock  →  p->lock
        ①                   ②              ③
```

Thực tế là **3 tầng lock**, vì caller (pipe, log, ...) giữ `lk` khi gọi `sleep(chan, lk)`:
- `sleep()`: caller giữ `lk` → acquire bucket lock → acquire `p->lock` → release `lk` (bên trong)
- `wakeup()`: caller giữ `lk` → acquire bucket lock → acquire `p->lock`

> **Deadlock rule:** Không bao giờ acquire bucket lock khi đang giữ `p->lock`. Không bao giờ acquire `lk` khi đang giữ bucket lock (trừ khi đã release bucket lock trước).

---

## 5. Core API — `kernel/waitqueue.c`

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

---

## 6. Rewritten `sleep()` — `prepare_to_wait` pattern

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

---

## 7. Rewritten `wakeup()` — O(k) + autoremove

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

---

## 8. `wakeup_one()` — Anti-thundering herd

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

---

## 9. Pipe optimization — `kernel/pipe.c`

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
                if(wq_has_sleeper(&pi->nread))
                    wakeup_one(&pi->nread);
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

## 10. Tài liệu tham khảo

| Tài liệu | Link |
|-----------|------|
| Linux `include/linux/wait.h` | [Source](https://elixir.bootlin.com/linux/latest/source/include/linux/wait.h) |
| Linux `kernel/sched/wait.c` | [Source](https://elixir.bootlin.com/linux/latest/source/kernel/sched/wait.c) |
| Linux `__wake_up_common()` | [Source](https://elixir.bootlin.com/linux/latest/source/kernel/sched/wait.c) |
| Linux `fs/pipe.c` | [Source](https://elixir.bootlin.com/linux/latest/source/fs/pipe.c) |
| Columbia OS Wait Queues | [Lecture](https://cs4118.github.io/www/2023-1/lect/10-run-wait-queues.html) |
