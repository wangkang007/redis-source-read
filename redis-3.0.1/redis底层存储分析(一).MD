# redis底层存储分析(一)
## 前言
本文是基于redis3.0.1源码进行的解析，本文章主要分析redis存储key时候的数据结构
## dict 数据结构
```c
/*存储具体的key和value*/
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    /* 链表解决hash冲突*/
    struct dictEntry *next;
} dictEntry;

typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;
```
总体数据结构图如下所示
![](https://github.com/wangkang007/redis-source-read/raw/master/redis-3.0.1/static/redis_dict.png)

### dictEntry的设计
```c

typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    /* 链表解决hash冲突*/
    struct dictEntry *next;
} dictEntry;

```
dictEntry主要是使用*void 来存储实际的数据，value的的存储做了一个优化，把value分成不同的数据类型进行存储，正常情况下来讲用void *val来存储肯定也不会存在浪费空间一说，毕竟只是一个指针。这块按照我的理解其实也没必要做成这样，void *val其实也可以转成任意的类型，这块还是比较质疑redis，感觉这复杂度会变高了。

dictEntry *next的话主要是解决hash冲突，针对于这种解决冲突的方式，java里面有很好的实现，java用的是红黑树来解决的，红黑树查找效率会更高一些


### dictht的设计
```c

typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

```

dictEntry **table是具体的存储，dict在其他语言里面大多数都叫字典，在java里面叫HashMap，实际实现都是大同小异的。size是标记table数组的大小，used用来标记当前dict中存储的元素数量

#### 如何确认dictht该扩容
```c
static unsigned int dict_force_resize_ratio = 5;
static int _dictExpandIfNeeded(dict *d)
{
    //........省略部分代码
    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}

```

扩容是需要满足两个条件，当used>size&&(dict_can_resize||d->ht[0].used/d->ht[0].size > dict_force_resize_ratio)，从官方的默认条件来看扩容是需要确认当前是否可以扩容，或者used的大小是size的5倍,

小扩展：`dict_force_resize_ratio` redis负载因子是5也就是说used是size的5倍的时候才会进行扩展，但是java里hashmap的实现是

#### 扩容与收缩 rehash

```c

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;

```

首先来解释下为什么需要rehash？rehash主要是因为数据结构的问题，首先底层是用数组+链表的结构来存储的,一旦发生hash冲突，就需要用链表来解决，但是如果一旦量变大之后，hash冲突过多就会导致链表过长，查询效率变低，所以这个时候需要进行rehash，重新分配数组大小，重新去规划entry的存储位置。

rehash过程如下：

```c

dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;
    //判断是否正在处于rehash的状态，
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}

```

`dictht ht[2]`redis 采用两个dict来存储，ht[0]做为实际的存储，ht[1]做为迁移的中间存储，迁移成功后会释放调ht[0]的空间，并且让ht[0]->ht[1],  rehash的过程发生在添加的时候，redis采用的是逐步迁移方案，防止因dict过大迁移时导致卡顿阻塞，所以把压力分担给每一次调用，假如迁移需要1s，但是如果分担给1000次调用用才能迁移完，那么每次最多也就增加1ms。而在java中是采用的立即迁移，个人认为java这种方式还是可以进行再进一步的优化

```c

int dictRehash(dict *d, int n) {
    //这块是用来限制最大空节点的访问次数，比如有一个很大的数组，但是只有其中的某些地方存储了数据，这个时候如果都遍历完，就会产生性能问题，所以定义了最大empty_visits，每次只能访问数组中d->rehashidx+n*10的下标索引
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    //如果当前已经不需要rehash，那么直接返回
    if (!dictIsRehashing(d)) return 0;
    //进行逐步迁移，每次只迁移d->rehashidx 到 d->rehashidx+n的数据
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        //判断如果访问NULL次数超过了n*10就直接返回
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        //拿到ht[0]中需要迁移的entiy
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        //从ht[0]迁移到ht[1]
        while(de) {
            unsigned int h;

            nextde = de->next;
            /* Get the index in the new hash table */
            //重新计算hash
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        //标志已经迁移成功了，可以直接指向NULL
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    //如果d->ht[0].used == 0 那么表示已经完成了迁移，这个之后需要把ht[0]->ht[1]，然后reset ht[1]的指向
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}

```
上面代码对应的过程如下：

过程一：
![](https://github.com/wangkang007/redis-source-read/raw/master/redis-3.0.1/static/redis_rehash_one.png)

过程二：
