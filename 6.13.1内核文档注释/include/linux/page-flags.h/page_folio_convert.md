
```C
static __always_inline unsigned long _compound_head(const struct page *page)
{
	unsigned long head = READ_ONCE(page->compound_head);

	if (unlikely(head & 1))
		return head - 1;
	return (unsigned long)page_fixed_fake_head(page);
}
```
函数 _compound_head可以接受head page或者tail page。<br>
在普通的情况下`page_fixed_fake_head(page)`直接什么都不做返回page本身。<br>
因此如果传进去的就是head page那么直接返回自己。(但是地址被以unsigned long的形式返回回来了)。
如果不是的话结果会直接返回page->compound_head-1。那么因此有理由推测page->compound_head字段应该是tail page用来保存head page地址但是最后一个bit始终多了1。

```C
#define compound_head(page)	((typeof(page))_compound_head(page)):
```
- `#define compound_head(page)	((typeof(page))_compound_head(page))`
- `#define compound_head(page) ...`: 定义一个宏 compound_head。
- `((typeof(page))_compound_head(page))`: 宏展开。
- `_compound_head(page)`: 调用我们刚刚分析的 _compound_head 函数。
- `(typeof(page))`: 获取输入 page 参数的类型。 如果 page 的类型为 struct page *，则 typeof(page) 为 struct page *。
- `((typeof(page)) ...)`: 将 _compound_head(page) 的结果强制转换为与输入 page 相同的类型。 由于 _compound_head 返回 unsigned long，因此此强制转换将其转换回 struct page *（或 const struct page *，具体取决于输入 page 类型）。
compound_head 的目的:

此宏提供了一种类型安全的方式来获取复合页面（或 folio）的头页面。 它本质上包装了 _compound_head 函数，并确保返回类型与输入页面类型匹配。 这对于类型正确性以及避免在不同上下文中使用宏时出现编译器警告或错误非常重要。

**`#define page_folio(p)		(_Generic((p), ...))`:**

```c
#define page_folio(p)		(_Generic((p),				\
	const struct page *:	(const struct folio *)_compound_head(p), \
	struct page *:		(struct folio *)_compound_head(p)))
```

*   **`#define page_folio(p) ...`**: 定义一个宏 `page_folio`。
*   **`_Generic((p), ...)`**:  C11 泛型选择表达式。 它允许您根据第一个参数 `(p)` 的类型选择不同的表达式。
*   **`const struct page *:	(const struct folio *)_compound_head(p)`**:  如果 `p` 的类型为 `const struct page *`，则选择此表达式：
    *   `(const struct folio *)_compound_head(p)`: 调用 `_compound_head(p)` 以获取头页面（作为 `unsigned long`），然后将结果强制转换为 `const struct folio *`。 这执行了**从 `struct page *` 到 `struct folio *` 的转换**。
*   **`struct page *:		(struct folio *)_compound_head(p)`**: 如果 `p` 的类型为 `struct page *`，则选择此表达式：
    *   `(struct folio *)_compound_head(p)`:  与上面类似，但强制转换为 `struct folio *`（非 const）。

**`page_folio` 的目的**:

此宏是**将 `struct page *` 转换为 `struct folio *` 的关键**。 它利用 `_compound_head` 函数获取头页面（现在被视为 folio 头），然后执行类型强制转换到 `struct folio *`。 `_Generic` 表达式通过处理 `const struct page *` 和 `struct page *` 输入类型来确保类型安全。

**来自注释的重要上下文：**

*   “每个页面都是 folio 的一部分。” - 这是一个基本概念。 Folio 旨在包含所有页面，无论它们是大型 folio 的一部分还是单个页面 folio。
*   “在 `@page` 上不需要引用或锁。” - 这对于性能很重要。 但是，它也引入了竞争条件：“可能与 folio 分裂竞争，因此在获得 folio 的引用后，应重新检查 folio 是否仍然包含此页面。” 这意味着在调用 `page_folio` 之后，如果您需要依赖 folio 的稳定性，则必须获取对 folio 的引用并重新验证原始页面是否仍然是该 folio 的一部分。

**`#define folio_page(folio, n)	nth_page(&(folio)->page, n)`:**

```c
#define folio_page(folio, n)	nth_page(&(folio)->page, n)
```

*   **`#define folio_page(folio, n) ...`**: 定义一个宏 `folio_page`。
*   **`nth_page(&(folio)->page, n)`**: 宏展开。
    *   `&(folio)->page`:  获取 `struct folio` 中 `page` 成员的地址。 假设 `struct folio` 具有名为 `page` 的成员，该成员很可能是 folio 的第一个页面（头页面）。
    *   `nth_page(..., n)`: 使用 folio 的头页面的地址和索引 `n` 调用 `nth_page` 宏（在下面定义）。

**`folio_page` 的目的**:

此宏用于**获取 folio *内* 的第 `n` 个页面**。 它假设 folio 由连续的页面组成，并且 `n` 是相对于 folio 起始页面（头页面）的页面索引。 它使用 `nth_page` 宏来执行实际的页面检索。

**`#if defined(CONFIG_SPARSEMEM) && !defined(CONFIG_SPARSEMEM_VMEMMAP)` 和 `#else` 代码块，用于 `nth_page` 和 `folio_page_idx`:**

```c
#if defined(CONFIG_SPARSEMEM) && !defined(CONFIG_SPARSEMEM_VMEMMAP)
#define nth_page(page,n) pfn_to_page(page_to_pfn((page)) + (n))
#define folio_page_idx(folio, p)	(page_to_pfn(p) - folio_pfn(folio))
#else
#define nth_page(page,n) ((page) + (n))
#endif
```

*   **`#if defined(CONFIG_SPARSEMEM) && !defined(CONFIG_SPARSEMEM_VMEMMAP)`**: 基于内核配置选项的条件编译。
    *   `CONFIG_SPARSEMEM`:  稀疏内存支持。 在稀疏内存系统中，物理内存不一定在物理地址空间中是连续的。
    *   `CONFIG_SPARSEMEM_VMEMMAP`:  带有 vmemmap 的稀疏内存。 vmemmap 是一种将所有物理内存映射到内核虚拟地址空间的技术，即使在稀疏内存系统中也是如此。
    *   该条件检查是否启用了稀疏内存，但 *未* 启用 vmemmap。 这是一种特定的稀疏内存配置。
*   **`#if` 代码块（没有 vmemmap 的稀疏内存）：**
    *   `#define nth_page(page,n) pfn_to_page(page_to_pfn((page)) + (n))`:
        *   `page_to_pfn((page))`: 将 `struct page *` 转换为其页帧号 (PFN)。 PFN 是物理页号。
        *   `page_to_pfn((page)) + (n)`: 将索引 `n` 添加到 PFN。 这计算初始 `page` 之后第 `n` 个页面的 PFN。
        *   `pfn_to_page(...)`: 将 PFN 转换回 `struct page *`。
        *   **含义**: 在没有 vmemmap 的稀疏内存中，页面地址不一定与其索引线性相关。 因此，要获取第 `n` 个页面，您需要转换为 PFN，添加偏移量，然后再转换回 `struct page *`。
    *   `#define folio_page_idx(folio, p)	(page_to_pfn(p) - folio_pfn(folio))`:
        *   `folio_pfn(folio)`: 获取 folio 的头页面的 PFN。
        *   `page_to_pfn(p)`: 获取输入页面 `p` 的 PFN。
        *   `page_to_pfn(p) - folio_pfn(folio)`: 计算 PFN 的差值，这给出了页面 `p` 在 folio 中的索引。
        *   **含义**: 通过从 `p` 的 PFN 中减去 folio 的头页面的 PFN，来计算页面 `p` 在 folio 中的索引。
*   **`#else` 代码块（非稀疏内存或带有 vmemmap 的稀疏内存）：**
    *   `#define nth_page(page,n) ((page) + (n))`:
        *   `((page) + (n))`:  仅执行指针算术。 将 `n` 乘以 `struct page` 的大小添加到 `page` 指针。
        *   **含义**: 在非稀疏内存或带有 vmemmap 的稀疏内存中，页面地址与其索引线性相关。 因此，您可以通过指针算术直接获取第 `n` 个页面。

**`nth_page` 和 `folio_page_idx` 的目的**:

*   **`nth_page`**:  提供了一种从给定的 `struct page *` 开始获取第 `n` 个页面的方法。 实现方式因是否使用没有 vmemmap 的稀疏内存而异，以处理地址空间差异。
*   **`folio_page_idx`**:  计算给定页面在 folio 中的索引。 它仅为没有 vmemmap 的稀疏内存定义，在其中需要 PFN 差异来确定索引。 在其他内存模型中，如果需要，您可能可以直接使用指针算术来计算索引。

**Folio 设计理念讨论：**

您对 folio 设计原则的理解非常准确。 让我们总结和详细说明：

**您理解的正确要点：**

*   **Folio 作为头页面的替代品：** 是的，folio 旨在成为表示页面缓存单元的主要结构，有效地在许多上下文中取代传统的“头页面”概念。
*   **避免频繁的头/尾页面检查：** 绝对正确。 历史上，处理复合页面涉及频繁的检查，以确定 `struct page` 是头页面还是尾页面。 Folio 旨在抽象化这一点。 使用 folio，您主要与 `struct folio` 结构交互，并且头/尾区分在许多 API 中变得不太明确。
*   **专注于使用 folio 结构来存储元数据：** 正确。 `struct folio` 旨在保存应用于整个 folio（可以是单个页面或多个页面）的元数据。 这集中了元数据管理，并简化了对页面缓存单元的操作。
*   **64 字节大小，用于与 `page` 的类型转换兼容性：** 是的，Matthew Wilcox 明确地将 `struct folio` 设计为 64 字节（在许多架构上与 `struct page` 相同），以实现高效的类型转换，特别是从 `struct page *`（头页面）到 `struct folio *`。 这对于 `page_folio` 宏以及在过渡到 folio 期间最大限度地减少代码更改至关重要。

**引用计数和 Folio：**

您提出了关于引用计数的一个非常重要的观点。 以下是 folio 如何解决它的：

*   **Folio 本身管理引用计数：** `struct folio` 结构本身包含引用计数机制（可能在其成员中，尽管在这些代码片段中未明确显示）。 当您获得 `struct folio *` 时（例如，使用 `page_folio`），通常期望您使用 folio 特定的函数（如 `folio_get`、`folio_put` 等，这些函数未在此处显示，但属于 folio API 的一部分）来管理其引用计数。
*   **从 `struct page` 到 `struct folio` 的隐式引用：** 当 `struct page` 是 folio 的一部分时，从 `struct page` 到 `struct folio` 存在隐式引用（通过 `page->compound_head` 或类似机制）。 这确保了只要其任何组成页面正在使用，folio 就会保持活动状态。
*   **类型转换和引用管理：** `page_folio` 宏促进了从 `struct page *` 到 `struct folio *` 的类型转换。 当您使用 `page_folio(page)` 时，您会获得一个 `struct folio *`，它表示包含 `page` 的 folio。 然后，您操作此 `struct folio *` 并使用 folio 特定的 API 管理其引用计数。 底层的 `struct page` 的引用计数在某些较低级别的操作中可能仍然相关，但对于更高级别的大多数页面缓存操作，您使用 folio 引用。

**Folio 设计的优点：**

*   **简化的 API：** Folio 旨在为与页面缓存单元交互提供更统一和更简单的 API，从而降低处理头/尾页面的复杂性。
*   **改进的性能：** 通过将元数据集中在 `struct folio` 中并优化 folio 单元的操作，它可以潜在地提高页面缓存性能，特别是对于受益于大型 folio 的工作负载。
*   **更好的抽象：** Folio 提供了对底层页面管理的更高级别的抽象，使页面缓存代码更模块化且更易于维护。
*   **大型 Folio 支持：** Folio 旨在无缝支持大型 folio（多个连续页面被视为单个单元），这可以提高 TLB 效率并减少大型内存区域的页表开销。
