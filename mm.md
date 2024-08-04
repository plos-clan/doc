# mm

Plant OS内存管理的实现

## Warnings

## Using

## Testing

## TODO

虚拟内存

## Principle

### 基本原理

在32位模式下，线性地址的组成分为三个部分

```[graph]
digraph G {
    "dir" [
        label = "<f0>Directory | <f1>Table | <f2>Offset"
        shape = record
    ]
    "cr3" [
        label = "cr3"
        shape = record
    ]
    "first+" [
        label = "+"
    ]
    "second+" [
        label = "+"
    ]
    "third+" [
        label = "+"
    ]
    subgraph _ {
        "页目录" [
            label = "页目录||<f0>|"
            shape = record
        ]
        "页表" [
            label = "页表||<f0>|"
            shape = record
        ]
        "页" [
            label = "页||<f0>|"
            shape = record
        ]
        rank = same
    }
    "cr3" -> "first+"
    "dir":f0 -> "first+"
    "first+" -> "页目录":f0
    "页目录":f0 -> "second+"
    "dir":f1 -> "second+"
    "second+" -> "页表":f0
    "页表":f0 -> "third+"
    "dir":f2 -> "third+"
    "third+" -> "页":f0
    
}
```

依据此原理，我们可以维护页目录和页表来设置线性地址与物理地址的对应关系

通过设置页目录和页表的属性，例如，关闭某个页的写属性，那么在ring3下访问这个页，就会报PF（Page Fault），那么我们就可以处理这个PF，重新设置这个页的属性，这也是COW（Copy on write，写时复制)的原理。

### pde_clone

以下是代码：

```c
u32 pde_clone(u32 addr) {
  for (int i = DIDX(0x70000000) * 4; i < 0x1000; i += 4) {
    u32 *pde_entry = (u32 *)(addr + i);
    u32  p         = *pde_entry & (0xfffff000);
    pages[IDX(*pde_entry)].count++;
    *pde_entry &= ~PAGE_WRABLE;
    for (int j = 0; j < 0x1000; j += 4) {
      u32 *pte_entry = (u32 *)(p + j);
      if ((page_get_attr(get_line_address(i / 4, j / 4, 0)) & PAGE_USER)) {
        pages[IDX(*pte_entry)].count++;
        if (page_get_attr(get_line_address(i / 4, j / 4, 0)) & PAGE_SHARED) {
          *pte_entry |= PAGE_WRABLE;
          continue;
        }
      }
      *pte_entry &= ~PAGE_WRABLE;
    }
  }
  u32 result = (u32)page_malloc_one_no_mark();
  memcpy((void *)result, (void *)addr, 0x1000);
  flush_tlb(result);
  flush_tlb(addr);
  asm_set_cr3(addr);

  return result;
}
```

```c
for (int i = DIDX(0x70000000) * 4; i < 0x1000; i += 4) 
```

遍历 **0x70000000 到 末尾的** 所对应的 **页目录**：这里不使用"i < DIDX(0xf0000000)"（0xf0000000是用户地址结束的地方），是因为，本来这后面也不应该被使用，所以我们这里就干脆全部设置了，省事，也防止一些bug，比如哪里做完某些操作之后，忘记将某些敏感地址写回不可写后，给恶意程序有可乘之机。

```c
    u32 *pde_entry = (u32 *)(addr + i);
    u32  p         = *pde_entry & (0xfffff000);
    pages[IDX(*pde_entry)].count++;
    *pde_entry &= ~PAGE_WRABLE;
```

`pde_entry`: pde中表项的地址
`p` : pde表项低十二位是属性位，前面就是物理页的索引，因而将第十二位置0，之后来读取其存储的pte地址
`pages[IDX(*pde_entry)].count++` 、`*pde_entry &= ~PAGE_WRABLE`：毕竟是克隆一份pde，所以相应的这里也要增加一份引用，然后我们将这个地方设置成不可写（为了实现写时复制这一特性）

```c
for (int j = 0; j < 0x1000; j += 4)
```

遍历PTE的表项

```c
      u32 *pte_entry = (u32 *)(p + j);
      if ((page_get_attr(get_line_address(i / 4, j / 4, 0)) & PAGE_USER)) {
        pages[IDX(*pte_entry)].count++;
        if (page_get_attr(get_line_address(i / 4, j / 4, 0)) & PAGE_SHARED) {
          *pte_entry |= PAGE_WRABLE;
          continue;
        }
      }
      *pte_entry &= ~PAGE_WRABLE;
```

`pte_entry` ：PTE表项的地址

我们先获取到这个pte表项所对应页框（物理）的属性，如果用户可访问，其引用自增<br>

## Dependencies

mem.c -- malloc free realloc( 依赖于 `libc/alloc`) kmalloc kree krealloc(依赖于`page.c`)的实现

## Contribute
