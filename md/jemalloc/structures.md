## jemalloc源码分析之基础数据结构

这篇文章主要是分析jemalloc中的基础数据结构, 然后大致介绍各个文件的作用, 最后简单梳理下各个结构体之间的关系.


目录:

1. 目录结构
2. 循环链表
3. 哈希表
4. 位图
5. 最小堆
6. 基数树
7. 红黑树
8. 内存分配pages
9. 原子操作atomic
10. 锁mutex
11. 时间操作nstime
12. 触发器ticker
13. 死锁监控witness
14. 断言assert
15. 数据统计ctl.h
16. 内存屏障mb.h
17. 伪随机数prng.h
18. 性能分析prof.h
19. smoothstep.h
20. spin.h
21. stats.h
22. util.h
23. 内存模块base
24. 内存模块std
25. 内存模块extent
26. 内存模块arena
27. 内存模块tcache




### 目录结构

因为jemalloc是C的程序, 自然其主要的逻辑都在.h和.c文件中. 通过tree命令, 我们打印出其目录树:


    ├── include
    │   ├── jemalloc
    │   │   ├── internal
    │   │   │   ├── arena.h
    │   │   │   ├── assert.h
    │   │   │   ├── atomic.h
    │   │   │   ├── base.h
    │   │   │   ├── bitmap.h
    │   │   │   ├── ckh.h
    │   │   │   ├── ctl.h
    │   │   │   ├── extent.h
    │   │   │   ├── extent_dss.h
    │   │   │   ├── extent_mmap.h
    │   │   │   ├── hash.h
    │   │   │   ├── jemalloc_internal.h
    │   │   │   ├── jemalloc_internal.h.in
    │   │   │   ├── jemalloc_internal_decls.h
    │   │   │   ├── jemalloc_internal_defs.h
    │   │   │   ├── jemalloc_internal_defs.h.in
    │   │   │   ├── jemalloc_internal_macros.h
    │   │   │   ├── large.h
    │   │   │   ├── mb.h
    │   │   │   ├── mutex.h
    │   │   │   ├── nstime.h
    │   │   │   ├── pages.h
    │   │   │   ├── ph.h
    │   │   │   ├── private_namespace.h
    │   │   │   ├── private_namespace.sh
    │   │   │   ├── private_symbols.txt
    │   │   │   ├── private_unnamespace.h
    │   │   │   ├── private_unnamespace.sh
    │   │   │   ├── prng.h
    │   │   │   ├── prof.h
    │   │   │   ├── public_namespace.h
    │   │   │   ├── public_namespace.sh
    │   │   │   ├── public_symbols.txt
    │   │   │   ├── public_unnamespace.h
    │   │   │   ├── public_unnamespace.sh
    │   │   │   ├── ql.h
    │   │   │   ├── qr.h
    │   │   │   ├── rb.h
    │   │   │   ├── rtree.h
    │   │   │   ├── size_classes.h
    │   │   │   ├── size_classes.sh
    │   │   │   ├── smoothstep.h
    │   │   │   ├── smoothstep.sh
    │   │   │   ├── spin.h
    │   │   │   ├── stats.h
    │   │   │   ├── tcache.h
    │   │   │   ├── ticker.h
    │   │   │   ├── tsd.h
    │   │   │   ├── util.h
    │   │   │   └── witness.h
    │   │   ├── jemalloc.h
    │   │   ├── jemalloc_defs.h
    │   │   ├── jemalloc_macros.h
    │   │   ├── jemalloc_mangle.h
    │   │   ├── jemalloc_mangle_jet.h
    │   │   ├── jemalloc_protos.h
    │   │   ├── jemalloc_protos_jet.h
    │   │   ├── jemalloc_rename.h
    │   │   ├── jemalloc_typedefs.h
    │   └── msvc_compat
    │       ├── C99
    │       │   ├── stdbool.h
    │       │   └── stdint.h
    │       ├── strings.h
    │       └── windows_extra.h
    ├── jemalloc.pc
    └── src
        ├── arena.c
        ├── atomic.c
        ├── base.c
        ├── bitmap.c
        ├── ckh.c
        ├── ctl.c
        ├── extent.c
        ├── extent_dss.c
        ├── extent_mmap.c
        ├── hash.c
        ├── jemalloc.c
        ├── large.c
        ├── mb.c
        ├── mutex.c
        ├── nstime.c
        ├── pages.c
        ├── prng.c
        ├── prof.c
        ├── rtree.c
        ├── spin.c
        ├── stats.c
        ├── tcache.c
        ├── ticker.c
        ├── tsd.c
        ├── util.c
        ├── witness.c
        └── zone.c

我们首先从跟业务无关的链表等数据结构分析起.




### 循环链表

qr.h实现的双向循环链表(不带头节点), 特点是尾部插入的时间复杂度是O(1).

其定义为

    #define	qr(a_type)							\
    struct {								\
    	a_type	*qre_next;						\
    	a_type	*qre_prev;						\
    }

注: 下面提到的所有内容都可以通过相关的关键字找到具体的文件及行数.

在extent_t结构中有使用到:

    extent_t {
        /*
         * Linkage for arena's extents_dirty and arena_bin_t's slabs_full rings.
         */
        qr(extent_t)		qr_link;
    }

展开即是

    struct {
        extent_t	*qre_next;
        extent_t	*qre_prev;
    } qr_link;

相关的操作:

    创建节点:
    qr_new(a_qr, a_field)

    查找下个节点:
    qr_next(a_qr, a_field) ((a_qr)->a_field.qre_next)

    查找上个节点:
    qr_prev(a_qr, a_field) ((a_qr)->a_field.qre_prev)

    将a_qr插入到a_qrelm前面
    qr_before_insert(a_qrelm, a_qr, a_field)

    将a_rq插入到a_qrelm后面
    qr_after_insert(a_qrelm, a_qr, a_field)

    1. 如果a_qr_a与a_qr_b不在同一个链表中, 那么是将a_qr_b插入到a_qr_a的前面.
       这个在将slab插入到slab_full中有用到.
    2. 如果在同一个链表中, 会将这个链表拆分成两个循环链表.
    qr_meld(a_qr_a, a_qr_b, a_field)

    qr_split(a_qr_a, a_qr_b, a_field)等同于qr_meld

    把a_qr从链表中删除
    qr_remove(a_qr, a_field)

    正向遍历a_qr
    qr_foreach(var, a_qr, a_field)

    反向遍历a_qr
    qr_reverse_foreach(var, a_qr, a_field)


ql.h实现的是带头节点的单向循环链表, 特点在获取最后一个节点的下个节点是NULL而不是首个节点.

其头节点定义为

    #define	ql_head(a_type)							\
    struct {								\
    	a_type *qlh_first;						\
    }

在arena\_s, tsd\_init\_head\_s, witness\_t, tsd\_s中都有使用.

    arena_s {
        /*
         * List of tcaches for extant threads associated with this arena.
         * Stats from these are merged incrementally, and at exit if
         * opt_stats_print is enabled.
         */
        ql_head(tcache_t)	tcache_ql;
        /* Extant large allocations. */
        ql_head(extent_t)	large;
        /* Cache of extent structures that were allocated via base_alloc(). */
        ql_head(extent_t)	extent_cache;
    }

    struct tsd_init_head_s {
    	ql_head(tsd_init_block_t)	blocks;
    };

    typedef ql_head(witness_t) witness_list_t;
    tsd_s {
        tsd_state_t state;
        tcache_t *tcache;
        uint64_t thread_allocated;
        uint64_t thread_deallocated;
        prof_tdata_t *prof_tdata;
        arena_t *iarena;
        arena_t *arena;
        arena_tdata_t *arenas_tdata;
        unsigned int narenas_tdata;
        _Bool arenas_tdata_bypass;
        tcache_enabled_t tcache_enabled;
        rtree_ctx_t rtree_ctx;
        witness_list_t witnesses;
        rtree_elm_witness_tsd_t rtree_elm_witnesses;
        _Bool witness_fork;
    }

其操作包括:

    初始化头节点
    ql_head_initializer(a_head) {NULL}

    申明一个节点
    ql_elm(a_type)	qr(a_type)

    初始化头节点
    ql_new(a_head)

    初始化节点
    ql_elm_new(a_elm, a_field) qr_new((a_elm), a_field)

    取第一个节点
    ql_first(a_head) ((a_head)->qlh_first)

    取最后一个节点
    ql_last(a_head, a_field)

    // 取a_elm的下个节点, 尾节点的下个节点是NULL
    ql_next(a_head, a_elm, a_field)

    获取e_elm的上个节点, 首节点的上个节点是NULL
    ql_prev(a_head, a_elm, a_field)

    把a_elm插入到a_qlelm前面
    ql_before_insert(a_head, a_qlelm, a_elm, a_field)

    把a_elm插入到a_qlelm后面
    ql_after_insert(a_qlelm, a_elm, a_field)

    a_elm插入到第一个节点
    ql_head_insert(a_head, a_elm, a_field)

    a_elm插入到最后一个节点
    ql_tail_insert(a_head, a_elm, a_field)

    将a_elm从链表中移除
    ql_remove(a_head, a_elm, a_field)

    移除第一个节点
    ql_head_remove(a_head, a_type, a_field)

    移除最后一个节点
    ql_tail_remove(a_head, a_type, a_field)

    正向遍历
    ql_foreach(a_var, a_head, a_field)

    反向遍历
    ql_reverse_foreach(a_var, a_head, a_field)




### 哈希表

hash.h/hash.c/ckh.h/ckh.c实现的是cuckoo hashing, [Cuckoo_hashing的wikipedia](https://en.wikipedia.org/wiki/Cuckoo_hashing),
其特点是双散列法解决Hash冲突, 即使用两个Hash函数.

Hash表的定义:

    /* Hash table cell. */
    struct ckhc_s {
    	const void	*key;
    	const void	*data;
    };

    struct ckh_s {
    #ifdef CKH_COUNT
    	/* Counters used to get an idea of performance. */
    	uint64_t	ngrows;
    	uint64_t	nshrinks;
    	uint64_t	nshrinkfails;
    	uint64_t	ninserts;
    	uint64_t	nrelocs;
    #endif

    	/* Used for pseudo-random number generation. */
    	uint64_t	prng_state;

    	/* Total number of items. */
    	size_t		count;

    	/*
    	 * Minimum and current number of hash table buckets.  There are
    	 * 2^LG_CKH_BUCKET_CELLS cells per bucket.
    	 */
    	unsigned	lg_minbuckets;
    	unsigned	lg_curbuckets;

    	/* Hash and comparison functions. */
    	ckh_hash_t	*hash;
    	ckh_keycomp_t	*keycomp;

    	/* Hash table with 2^lg_curbuckets buckets. */
    	ckhc_t		*tab;
    };

在prof\_tdata\_s结构和全局变量bt2gctx中有使用:

    struct prof_tdata_s {
    	/*
    	 * Hash of (prof_bt_t *)-->(prof_tctx_t *).  Each thread tracks
    	 * backtraces for which it has non-zero allocation/deallocation counters
    	 * associated with thread-specific prof_tctx_t objects.  Other threads
    	 * may write to prof_tctx_t contents when freeing associated objects.
    	 */
    	ckh_t			bt2tctx;
    };
    /*
     * Global hash of (prof_bt_t *)-->(prof_gctx_t *).  This is the master data
     * structure that knows about all backtraces currently captured.
     */
    static ckh_t		bt2gctx;


其操作方法:

    bool	ckh_new(tsdn_t *tsdn, ckh_t *ckh, size_t minitems, ckh_hash_t *hash,
        ckh_keycomp_t *keycomp);
    void	ckh_delete(tsdn_t *tsdn, ckh_t *ckh);
    size_t	ckh_count(ckh_t *ckh);
    bool	ckh_iter(ckh_t *ckh, size_t *tabind, void **key, void **data);
    bool	ckh_insert(tsdn_t *tsdn, ckh_t *ckh, const void *key, const void *data);
    bool	ckh_remove(tsdn_t *tsdn, ckh_t *ckh, const void *searchkey, void **key,
        void **data);
    bool	ckh_search(ckh_t *ckh, const void *searchkey, void **key, void **data);
    void	ckh_string_hash(const void *key, size_t r_hash[2]);
    bool	ckh_string_keycomp(const void *k1, const void *k2);
    void	ckh_pointer_hash(const void *key, size_t r_hash[2]);
    bool	ckh_pointer_keycomp(const void *k1, const void *k2);





### 位图

bitmap.h/bitmap.c是位图的实现, 比较特别的是在bit的个数超过特定的阀值, 会使用额外的空间形成树结构来提升搜索bit0的速度.

bitmap主要是用来搜索可用的指定大小的region用的, 一段连续的内存如8KB, 被切成1024个8B大小的region, 然后使用1024bit,
即128B的bitmap来标识每个region是否被使用.
刚开始bitmap全部被初始化为1, 那么找到第一个可用的region, 即是找到第一个bit0的位置.


对bitmap的分析我们拆成两部分, 一部分是普通实现, 另一部分是树结构的实现.

普通结构的定义:

    struct bitmap_info_s {
    	/* Logical number of bits in bitmap (stored at bottom level). */
    	size_t nbits;
    	/* Number of groups necessary for nbits. */
    	size_t ngroups;
    };
    typedef unsigned long bitmap_t;

带树结构的定义:

    struct bitmap_level_s {
    	/* Offset of this level's groups within the array of groups. */
    	size_t group_offset;
    };

    struct bitmap_info_s {
    	/* Logical number of bits in bitmap (stored at bottom level). */
    	size_t nbits;

    	/* Number of levels necessary for nbits. */
    	unsigned nlevels;

    	/*
    	 * Only the first (nlevels+1) elements are used, and levels are ordered
    	 * bottom to top (e.g. the bottom level is stored in levels[0]).
    	 */
    	bitmap_level_t levels[BITMAP_MAX_LEVELS+1];
    };
    typedef unsigned long bitmap_t;

bitmap\_info\_s在arena\_bin\_info\_s结构中使用:

    struct arena_bin_info_s {
    	/*
    	 * Metadata used to manipulate bitmaps for slabs associated with this
    	 * bin.
    	 */
    	bitmap_info_t		bitmap_info;
    };

bitmap\_t在arena\_slab\_data\_s结构体中使用:

    struct arena_slab_data_s {
    	/* Per region allocated/deallocated bitmap. */
    	bitmap_t	bitmap[BITMAP_GROUPS_MAX];
    };

其操作:

    初始化bitmap_info, 以支持至少nbits个bit位
    void	bitmap_info_init(bitmap_info_t *binfo, size_t nbits);

    初始化bitmap, 使其binfo->nbits个位全为1, 如果nbits不是sizeof(bitmap_t)的倍数, 那么剩余的bit保留为0, 代表不可用.
    void	bitmap_init(bitmap_t *bitmap, const bitmap_info_t *binfo);

    获取nbits的大小
    size_t	bitmap_size(const bitmap_info_t *binfo);

    判断bitmap中是否还有1可用
    bool	bitmap_full(bitmap_t *bitmap, const bitmap_info_t *binfo);

    第bit(范围[0, nbits))位是1返回false, 是0返回true
    bool	bitmap_get(bitmap_t *bitmap, const bitmap_info_t *binfo, size_t bit);

    将第bit位从1变成0
    void	bitmap_set(bitmap_t *bitmap, const bitmap_info_t *binfo, size_t bit);

    查找第一个bit0的位置, 并设置成1
    size_t	bitmap_sfu(bitmap_t *bitmap, const bitmap_info_t *binfo);

    将第bit位从0变成1
    void	bitmap_unset(bitmap_t *bitmap, const bitmap_info_t *binfo, size_t bit);




### 最小堆

ph.h是"The Pairing Heap"的实现, 它是一种自平衡的堆, 保证插入/删除操作都在O(log n), 而不至于退化成链表O(n)的删除.

其定义为

    #define	phn(a_type)							\
    struct {								\
    	a_type	*phn_prev;						\
    	a_type	*phn_next;						\
    	a_type	*phn_lchild;						\
    }

    /* Root structure. */
    #define	ph(a_type)							\
    struct {								\
    	a_type	*ph_root;						\
    }

在jemalloc中它只被定义成了一种特定的结构extent\_heap\_t:

    typedef ph(extent_t) extent_heap_t;

它在arena\_bin\_s, arena\_s 等结构和全局变量base\_avail中使用:

    struct arena_bin_s {
    	/*
    	 * Heap of non-full slabs.  This heap is used to assure that new
    	 * allocations come from the non-full slab that is lowest in memory.
    	 */
    	extent_heap_t		slabs_nonfull;
    };
    struct arena_s {
    	/*
    	 * Heaps of extents that were previously allocated.  These are used when
    	 * allocating extents, in an attempt to re-use address space.
    	 */
    	extent_heap_t		extents_cached[NPSIZES];
    	extent_heap_t		extents_retained[NPSIZES];
    };
    static extent_heap_t	base_avail[NSIZES];

而phn只在extent_s结构中使用, 说明上面三个结构或变量作为堆的头节点, 而extent_s.ph_link是作为值节点来定义的.

    struct extent_s {
    	union {
    		/* Linkage for per size class address-ordered heaps. */
    		phn(extent_t)		ph_link;

    		/* Linkage for arena's large and extent_cache lists. */
    		ql_elm(extent_t)	ql_link;
    	};
    };


其操作包括:

    #define	ph_proto(a_attr, a_prefix, a_ph_type, a_type)			\
    a_attr void	a_prefix##new(a_ph_type *ph);				\
    a_attr bool	a_prefix##empty(a_ph_type *ph);				\
    a_attr a_type	*a_prefix##first(a_ph_type *ph);			\
    a_attr void	a_prefix##insert(a_ph_type *ph, a_type *phn);		\
    a_attr a_type	*a_prefix##remove_first(a_ph_type *ph);			\
    a_attr void	a_prefix##remove(a_ph_type *ph, a_type *phn);

    ph_proto(, extent_heap_, extent_heap_t, extent_t)

    #define	ph_gen(a_attr, a_prefix, a_ph_type, a_type, a_field, a_cmp)	\
    a_attr void								\
    a_prefix##new(a_ph_type *ph)						\
    {									\
    									\
    	memset(ph, 0, sizeof(ph(a_type)));				\
    }
    /* Generate pairing heap functions. */
    ph_gen(, extent_heap_, extent_heap_t, extent_t, ph_link, extent_ad_comp)

    对应下来就是:

    初始化一个堆
    extent_heap_new(extent_heap_t * ph)

    判断堆是否为空
    extent_heap_empty(extent_heap_t * ph)

    获取堆中值最小的节点
    extent_heap_first(extent_heap_t * ph)

    插入一个节点
    extent_heap_insert(extent_heap_t * ph, extent_t *phn)

    删除值最小的节点
    extent_heap_remove_first(extent_heap_t * ph)

    删除堆中指定的节点
    extent_heap_remove(extent_heap_t * ph, extent_t *phn)

通过ph\_gen宏, 我们知道该堆是通过extent\_ad\_comp比较元素大小的:

    extent_ad_comp(const extent_t *a, const extent_t *b)
    {
    	uintptr_t a_addr = (uintptr_t)extent_addr_get(a);
    	uintptr_t b_addr = (uintptr_t)extent_addr_get(b);

    	return ((a_addr > b_addr) - (a_addr < b_addr));
    }

在后面的分析中, 具体为base\_extent\_alloc函数, 我们可以看出a\_addr其实就是一个extent_t变量中可用地址的起始地址.

首先通过index算出当前满足条件的堆, 然后找出堆地址最小的内存块, 具体体现在base\_alloc函数中:

    base_alloc(tsdn_t *tsdn, size_t size)
    {
    	void *ret;
    	size_t csize;
    	szind_t i;
    	extent_t *extent;

    	/*
    	 * Round size up to nearest multiple of the cacheline size, so that
    	 * there is no chance of false cache line sharing.
    	 */
    	csize = CACHELINE_CEILING(size);

    	extent = NULL;
    	malloc_mutex_lock(tsdn, &base_mtx);
    	for (i = size2index(csize); i < NSIZES; i++) {
    		extent = extent_heap_remove_first(&base_avail[i]);
    		if (extent != NULL) {
    			/* Use existing space. */
    			break;
    		}
    	}

    }

为什么要找地址最小的堆呢, 这个跟程序的内存分布有关, 分配的内存使用的mmap系统调用, mmap的分配方向的向高地址增加的,
所以选择地址小的内存, 那么使用的内存更可能都集中在低地址的位置, 从而让内存更加紧凑, 减少内存的外部碎片.




### 基数树

### 红黑树

### 内存分配pages

### 原子操作atomic

### 锁mutex

### 时间操作nstime

### 触发器ticker

### 死锁监控witness

### 断言assert

### 数据统计ctl.h

### 内存屏障mb.h

### 伪随机数prng.h

### 性能分析prof.h

### smoothstep.h

### spin.h

### stats.h

### util.h

### 内存模块base

base.h主要负责内存分配的管理, 其主要定义的全局变量

    管理extent_t结构的一组堆结构
    static extent_heap_t	base_avail[NSIZES];

    static extent_t		*base_extents;
    static size_t		base_allocated;
    static size_t		base_resident;
    static size_t		base_mapped;



### 内存模块std

### 内存模块extent

### 内存模块arena

### 内存模块tcache
