# Buddy System 答辩

## 内容点

### 地址对齐：

以页为最小单位。每个阶的块必须按“块大小”对齐。判断是否对齐用 PFN 按位与或取模即可。

基本单位

页大小：4KB。PGSHIFT=12。

阶 o 的块大小：s_pages = 2^o 页。字节数 s_bytes = s_pages << PGSHIFT。

对齐规则

头页 PFN 必须是 s_pages 的整数倍。
公式：pfn % s_pages == 0 等价于 pfn & (s_pages - 1) == 0。

同一条规则在字节地址上也成立。
page2pa(p) & (s_bytes - 1) == 0。

为什么必须对齐

Buddy 用异或找伙伴：buddy_pfn = pfn ^ s_pages。
只有当低 o 位全为 0（也就是对齐到 2^o）时，翻转第 o 位才会把区间 [pfn, pfn+2^o-1] 映射到紧邻且不重叠的伙伴 [buddy_pfn, buddy_pfn+2^o-1]。

若未对齐，异或会落在区间中间。块边界被破坏，无法安全合并。

实现位置

初始化切块：选择“最大且对齐”的 k。条件
2^(k+1) <= n 且 pfn & (2^(k+1)-1) == 0。保证入表的每个块天然对齐。

拆分：2^o 一分为二得到两个 2^(o-1)。原头的低 o 位为 0，所以左右半块的低 o-1 位也为 0。天然对齐到下一阶。

释放合并前检查：
check_if_head_page(p, 2^o) 或 turn_page_to_pfn(p) & (2^o-1) == 0。不对齐直接拒绝合并。

快速例子

例1：o=3，s_pages=8。pfn=24 二进制 11000。
24 % 8 == 0 → 对齐。伙伴 24 ^ 8 = 16。两个块 [16..23] 和 [24..31] 紧邻可合并。

例2：pfn=26，26 % 8 != 0 → 非头页。不能当作 8 页块的头，也不能参与 8 页层级的合并。

判定写法对照

PFN判定：(turn_page_to_pfn(p) & (ORDER_OF_PAGES(o) - 1)) == 0

字节判定：(page2pa(p) & ((ORDER_OF_PAGES(o) << PGSHIFT) - 1)) == 0

结论：以 4KB 页为基。阶 o 必须按 2^o 页对齐。用 PFN 与 (2^o-1) 为零即可判定。对齐保证 XOR 伙伴唯一且可合并。

### free释放任意n页步骤：

1. 预处理
   a-->遍历 [base, base+n)；确认 `!PageReserved && !PageProperty`；`set_page_ref(p,0)`。

2. 对齐切块
   a-->设 `pfn=turn_page_to_pfn(base)`，`remain=n`。
   b-->在每轮选择尽量大的阶 `k`，满足
     `2^(k+1) <= remain` 且 `(pfn & (2^(k+1)-1)) == 0`。
   c-->得到块大小 `s=2^k`，这保证块头对齐，可与伙伴合并。

3. 释放并合并（对每个块执行）
   a-->调用 `free_one_block_and_merge(base, k)`：
    - 记 `s=2^k`，断言 `base` 按 `s` 对齐；标记 `property=s`，`SetPageProperty`。
    - while 合并：令 `q = buddy_of(base, k)`；若 `q` 越界或 `!PageProperty(q)` 或 `q->property != s` 则停。
    - 否则从链表摘掉 `q`，清标记；取地址更小者为新头 `base=min(base,q)`；`k++`，`s<<=1`；更新 `property` 与标记；继续尝试更高一阶。
    - 合并结束，把最终块挂回 `flist(k)`。

4. 前进指针
   a-->`base += s`，`pfn += s`，`remain -= s`；回到步骤2，直到 `remain==0`。

5. 记账
   a-->一次性 `BUDDY_NR_FREE += n`。

要点与理由：

* “按对齐的最大 2^k 切块”保证每块都是合法头页，后续才可能与伙伴整块合并。
* 自始至终总账不变：释放区间覆盖的页数精确回到自由表，累计增加正好为 `n`。



## Make qemu 如何启动的呢？

进入内核 ucore 后：

```text
start -> 清 BSS/初始化控制台 -> 解析设备树/内存范围
      -> pmm_init()
         -> 选择 pmm_manager = buddy_pmm_manager
         -> buddy_init_memmap(base, n)   // 把可用物理内存按 2^k 对齐分块挂入各阶空闲链
      -> 后续 alloc_pages()/free_pages() 可用
```

全流程：

```
make qemu
→ 生成 obj/*.o → 链接出 bin/kernel → objcopy 成 bin/ucore.img
→ QEMU+OpenSBI 启动并把镜像装到 0x80200000 → 跳入内核入口
→ 内核初始化内存管理（buddy）→ 开始能 alloc_pages/free_pages
```

