# kprobe
cpu高排查思路:
1. pid定位
2. perf看函数
3. bpftrace追源头

## example
```bash
# bpftrace -e 'kprobe:worker_thread {printf("call from pid %d, func %s\n", pid, func);}'
# bpftrace -e 'kprobe:worker_thread { @[pid, func] = count(); }'
```