---
title: redis6.0-zset源码阅读
date: 2021-02-05 07:39:00
tags:
- redis
- src
- redis6.0
---


首先redis中的一个key就是redisObject结构, 一个key所属的基本数据类型通过type判断, 一个type如zset的实现方式通过encoding判断, 具体数据在ptr中
```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

当zset满足以下任一条件时, zset的存储方式是zip list, 以下代码在zadd里面

1. zip list的长度 > 常量server.zset_max_ziplist_entries 默认为128
2. 存储的member长度 > 常量server.zset_max_ziplist_value 默认为64

```
if (zzlLength(zobj->ptr) > server.zset_max_ziplist_entries ||
    sdslen(ele) > server.zset_max_ziplist_value)
{
}
```

本文重点介绍skip list存储方式, zset的encoding==OBJ_ENCODING_SKIPLIST时, 由字典和跳表组成

```
typedef struct zskiplistNode {
    sds ele;
    double score;
    // 回退结点
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        // 通过索引level向前遍历的下一个节点
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

下面通过源码查看各个元素所起的作用

本文介绍zset的以下操作:

1. zrank
2. zrevrange
3. zadd

从搜索开始更容易理解redis的skip list结构

## zrank

```
    {"zrank",zrankCommand,3,
     "read-only fast @sortedset",
     0,NULL,1,1,1,0,0,0},
```

```
void zrankCommand(client *c) {
    zrankGenericCommand(c, 0);
}

void zrevrankCommand(client *c) {
    zrankGenericCommand(c, 1);
}
```

zrank逻辑最简单, 看下代码
```
long zsetRank(robj *zobj, sds ele, int reverse) {
    unsigned long llen;
    unsigned long rank;

    // zset元素个数
    llen = zsetLength(zobj);

    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // 忽略zip list的逻辑
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        // skip list是根据score按序存储
        // 要根据member查score, 然后才能找到(member, score)对所对应结点的排名
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        dictEntry *de;
        double score;

        // zs->dict是字典, 根据member通过dict获取对应node
        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            score = *(double*)dictGetVal(de);
            // 获得排名
            rank = zslGetRank(zsl,score,ele);
            /* Existing elements always have a rank. */
            serverAssert(rank != 0);
            if (reverse)
                return llen-rank;
            else
                return rank-1;
        } else {
            return -1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
}

```

看一下score和member对所对应结点的排名
```
// 传member是因为zset跳表元素可以重复
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 在同一级level中根据score向后查找
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        if (x->ele && sdscmp(x->ele,ele) == 0) {
            return rank;
        }
        // 同一级level没找到, 向下一级level查
    }
    return 0;
}
```

总结zrank过程:

判断redisObject.encoding, 如果是skip list实现:

1. 首先通过字典dict根据member获取对应member的score
2. 然后根据(member, score)从skiplist中查找对应结点的排名

由以上过程可以看出skip list的结构和zskiplistNode中各个变量的作用, 对于理解skip list的insert算法有帮助

```
typedef struct zskiplistNode {
    sds ele;
    double score;
    // 回退结点(zrevrange回退搜索)
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        // 通过索引level向前遍历的下一个节点(根据score查找元素)
        struct zskiplistNode *forward;
        // 距离同级level的上一个结点的步长(利用排名搜索时有用)
        unsigned long span;
    } level[];
} zskiplistNode;
```

## zrevrange

入口函数

```
    {"zrevrange",zrevrangeCommand,-4,
     "read-only @sortedset",
     0,NULL,1,1,1,0,0,0},
```

zrevrange和zrange 调用相同的方法

```
void zrangeCommand(client *c) {
    zrangeGenericCommand(c,0);
}

void zrevrangeCommand(client *c) {
    zrangeGenericCommand(c,1);
}
```

```
void zrangeGenericCommand(client *c, int reverse) {
    robj *key = c->argv[1];
    robj *zobj;
    int withscores = 0;
    long start;
    long end;
    long llen;
    long rangelen;

    // 读取start end (zrevrange key start end)
    if ((getLongFromObjectOrReply(c, c->argv[2], &start, NULL) != C_OK) ||
        (getLongFromObjectOrReply(c, c->argv[3], &end, NULL) != C_OK)) return;

    // 提取参数withscores
    if (c->argc == 5 && !strcasecmp(c->argv[4]->ptr,"withscores")) {
        withscores = 1;
    } else if (c->argc >= 5) {
        addReply(c,shared.syntaxerr);
        return;
    }

    if ((zobj = lookupKeyReadOrReply(c,key,shared.emptyarray)) == NULL
         || checkType(c,zobj,OBJ_ZSET)) return;

    /* Sanitize indexes. */
    llen = zsetLength(zobj);
    // 小于0则表示倒序
    if (start < 0) start = llen+start;
    if (end < 0) end = llen+end;
    if (start < 0) start = 0;

    /* Invariant: start >= 0, so this test will be true when end < 0.
     * The range is empty when start > end or start >= length. */
    // 如果为负数则都为负数
    if (start > end || start >= llen) {
        addReply(c,shared.emptyarray);
        return;
    }
    if (end >= llen) end = llen-1;
    rangelen = (end-start)+1;

    /* Return the result in form of a multi-bulk reply. RESP3 clients
     * will receive sub arrays with score->element, while RESP2 returned
     * a flat array. */
    if (withscores && c->resp == 2)
        addReplyArrayLen(c, rangelen*2);
    else
        addReplyArrayLen(c, rangelen);

    // 搜索zip list
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // 这里忽略了zip list的搜索过程
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) { // 搜索skip list
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        zskiplistNode *ln; // ln存储起始结点
        sds ele;

        /* Check if starting point is trivial, before doing log(N) lookup. */
        if (reverse) {
            // zrevrange
            ln = zsl->tail;
            if (start > 0)
                // 倒序第llen-start个
                ln = zslGetElementByRank(zsl,llen-start);
        } else {
            // zrange
            // zsl->header不存储元素
            ln = zsl->header->level[0].forward;
            if (start > 0)
                ln = zslGetElementByRank(zsl,start+1);
        }

        // 找到了起始结点ln, 还需要从ln向后遍历rangelen个结点
        while(rangelen--) {
            serverAssertWithInfo(c,zobj,ln != NULL);
            ele = ln->ele;
            if (withscores && c->resp > 2) addReplyArrayLen(c,2);
            addReplyBulkCBuffer(c,ele,sdslen(ele));
            if (withscores) addReplyDouble(c,ln->score);
            // 如果是逆序zrevrange, 下次访问backward
            // zrange访问level[0].forward
            ln = reverse ? ln->backward : ln->level[0].forward;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
}
```

```
// 通过排名查找元素
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 从同一级level向后查找, 直到traversed大于rank
        // traversed记录遍历过的步长
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank)
        {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        // 找到了对应的排名
        if (traversed == rank) {
            return x;
        }
    }
    return NULL;
}
```

总结zskiplist的搜索过程:

首先判断redisObject.encoding是skip list的话:

zrevrange和zrange使用相同的搜索逻辑, 都是先通过排名获取起始结点, 再从起始结点向后或向前顺序遍历

利用步长span实现排名搜索, 具体做法是从最高层索引结点开始, 找到下一个步长大于排名的结点, 然后下调一个level继续搜索, 就这样利用span和level往后遍历.

找到起始结点后, 根据是否是reverse搜索, 判断向后还是向前, node->backward是往回搜索, 向后只能通过level[0]->forwards

## zadd

入口函数

```
    {"zadd",zaddCommand,-4,
     "write use-memory fast @sortedset",
     0,NULL,1,1,1,0,0,0},
```

zaddCommand调用zaddGenericCommand, 以下代码比较长但是不复杂

```
/* This generic command implements both ZADD and ZINCRBY. */
void zaddGenericCommand(client *c, int flags) {
    static char *nanerr = "resulting score is not a number (NaN)";
    robj *key = c->argv[1];
    robj *zobj;
    sds ele;
    double score = 0, *scores = NULL;
    int j, elements;
    int scoreidx = 0;
    /* The following vars are used in order to track what the command actually
     * did during the execution, to reply to the client and to trigger the
     * notification of keyspace change. */
    int added = 0;      /* Number of new elements added. */
    int updated = 0;    /* Number of elements with updated score. */
    int processed = 0;  /* Number of elements processed, may remain zero with
                           options like XX. */

    /* Parse options. At the end 'scoreidx' is set to the argument position
     * of the score of the first score-element pair. */
    /*
     * 最后scoreidx将定位到第一个score-member对的score位置;
     *
     * ZADD_*常量和flag |=的意义是标记flag包含ZADD_*常量,
     * 用来在以后通过&元算的结果是否=0来判断是否包含ZADD_* 
     * stackoverflow有对应的解释:https://stackoverflow.com/questions/14295469/what-does-mean-pipe-equal-operator
     * */
    scoreidx = 2;
    while(scoreidx < c->argc) {
        char *opt = c->argv[scoreidx]->ptr;
        if (!strcasecmp(opt,"nx")) flags |= ZADD_NX;
        else if (!strcasecmp(opt,"xx")) flags |= ZADD_XX;
        else if (!strcasecmp(opt,"ch")) flags |= ZADD_CH;
        else if (!strcasecmp(opt,"incr")) flags |= ZADD_INCR;
        else break;
        scoreidx++;
    }

    /* Turn options into simple to check vars. */
    int incr = (flags & ZADD_INCR) != 0;
    int nx = (flags & ZADD_NX) != 0;
    int xx = (flags & ZADD_XX) != 0;
    int ch = (flags & ZADD_CH) != 0;

    /* After the options, we expect to have an even number of args, since
     * we expect any number of score-element pairs. */
    /*
     * 偶数且不为0个score-member对, 否则向客户端抛异常, 因为zadd的(member, score)成对出现
     * */
    elements = c->argc-scoreidx;
    if (elements % 2 || !elements) {
        addReply(c,shared.syntaxerr);
        return;
    }
    elements /= 2; /* Now this holds the number of score-element pairs. */

    /* Check for incompatible options. */
    /*
     * 同时包含了nx和xx参数
     * */
    if (nx && xx) {
        addReplyError(c,
            "XX and NX options at the same time are not compatible");
        return;
    }

    if (incr && elements > 1) {
        addReplyError(c,
            "INCR option supports a single increment-element pair");
        return;
    }

    /* Start parsing all the scores, we need to emit any syntax error
     * before executing additions to the sorted set, as the command should
     * either execute fully or nothing at all. */
    /*
     * 从参数argv获取score数据存放到scores数组中
     * */
    scores = zmalloc(sizeof(double)*elements);
    for (j = 0; j < elements; j++) {
        if (getDoubleFromObjectOrReply(c,c->argv[scoreidx+j*2],&scores[j],NULL)
            != C_OK) goto cleanup;
    }

    /* Lookup the key and create the sorted set if does not exist. */
    // 根据key从redis中获取redisObject
    zobj = lookupKeyWrite(c->db,key);
    if (zobj == NULL) {
        // xx参数并且不存在这个zset
        if (xx) goto reply_to_client; /* No key + XX option: nothing to do. */
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
        {
            // 满足创建zip list的条件
            zobj = createZsetObject();
        } else {
            // 否则创建skip list
            zobj = createZsetZiplistObject();
        }
        // 创建一个zset
        dbAdd(c->db,key,zobj);
    } else {
        // key存在, 但是类型不是ZSET
        if (zobj->type != OBJ_ZSET) {
            addReply(c,shared.wrongtypeerr);
            goto cleanup;
        }
    }

    // 往zset中批量添加元素
    for (j = 0; j < elements; j++) {
        double newscore;
        score = scores[j];
        int retflags = flags;

        // member
        ele = c->argv[scoreidx+1+j*2]->ptr;
        // zset中逐个添加元素
        int retval = zsetAdd(zobj, score, ele, &retflags, &newscore);
        if (retval == 0) {
            addReplyError(c,nanerr);
            goto cleanup;
        }
        if (retflags & ZADD_ADDED) added++;
        if (retflags & ZADD_UPDATED) updated++;
        if (!(retflags & ZADD_NOP)) processed++;
        score = newscore;
    }
    server.dirty += (added+updated);

reply_to_client:
    if (incr) { /* ZINCRBY or INCR option. */
        if (processed)
            addReplyDouble(c,score);
        else
            addReplyNull(c);
    } else { /* ZADD. */
        addReplyLongLong(c,ch ? added+updated : added);
    }

cleanup:
    zfree(scores);
    if (added || updated) {
        signalModifiedKey(c->db,key);
        notifyKeyspaceEvent(NOTIFY_ZSET,
            incr ? "zincr" : "zadd", key, c->db->id);
    }
}

```

以下方法我忽略了zip list的insert逻辑, 但是这里会有个是否将zip list转化为skip list的判断, 判断条件在文章开头给出了

```
int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore) {
    /* Turn options into simple to check vars. */
    int incr = (*flags & ZADD_INCR) != 0;
    int nx = (*flags & ZADD_NX) != 0;
    int xx = (*flags & ZADD_XX) != 0;
    *flags = 0; /* We'll return our response flags. */
    double curscore;

    /* NaN as input is an error regardless of all the other parameters. */
    if (isnan(score)) {
        *flags = ZADD_NAN;
        return 0;
    }

    /* Update the sorted set according to its encoding. */
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // ...
        // 忽略了zip list的逻辑
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;

        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            /* NX? Return, same element already exists. */
            // zset不存在member, 但是设置了nx参数
            if (nx) {
                // 标记ZADD_NOP
                *flags |= ZADD_NOP;
                return 1;
            }
            curscore = *(double*)dictGetVal(de);

            /* Prepare the score for the increment if needed. */
            // 设置了incr参数
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *flags |= ZADD_NAN;
                    return 0;
                }
                if (newscore) *newscore = score;
            }

            /* Remove and re-insert when score changes. */
            if (score != curscore) {
                // 更新zset的skip list对应结点的score
                znode = zslUpdateScore(zs->zsl,curscore,ele,score);
                /* Note that we did not removed the original element from
                 * the hash table representing the sorted set, so we just
                 * update the score. */
                // 以上函数更新了skip list, 但是skiplist编码的zset由一个字典和一个skip list组成
                dictGetVal(de) = &znode->score; /* Update score ptr. */
                *flags |= ZADD_UPDATED;
            }
            return 1;
        } else if (!xx) { // zset不存在这个member, 且没有设置xx参数(xx表示只有存在才更新)
            ele = sdsdup(ele);
            // 插入跳表
            znode = zslInsert(zs->zsl,score,ele);
            // 插入字典
            serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
            *flags |= ZADD_ADDED;
            if (newscore) *newscore = score;
            return 1;
        } else {
            *flags |= ZADD_NOP;
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* Never reached. */
}
```


更新skip list:

```
// 注释说调用这个方法需要调用方保证skip list中含有这个元素
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    /* We need to seek to element to update to start: this is useful anyway,
     * we'll have to update or remove it. */
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < curscore ||
                    (x->level[i].forward->score == curscore &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        // update记录搜索member过程中, 从0到level各层的转折点(我自己这么叫的)
        update[i] = x;
    }

    /* Jump to our element: note that this function assumes that the
     * element with the matching score exists. */
    // x指向的是目标结点
    x = x->level[0].forward;
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

    /* If the node, after the score update, would be still exactly
     * at the same position, we can just update the score without
     * actually removing and re-inserting the element in the skiplist. */
    if ((x->backward == NULL || x->backward->score < newscore) &&
        (x->level[0].forward == NULL || x->level[0].forward->score > newscore))
    {
        // 需要给目标元素加score, 且目标元素的下一个结点的score > 目标元素, 这种情况直接修改score, 不需要删除再插入
        x->score = newscore;
        return x;
    }

    /* No way to reuse the old node: we need to remove and insert a new
     * one at a different place. */
    // 不能重复使用旧结点的情况(需要更新跳表的结构, 包括索引)
    // 直接删除再插入
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    /* We reused the old node x->ele SDS string, free the node now
     * since zslInsert created a new one. */
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}
```


插入skip list:

```
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        // rank[0]代表了要插入元素的排名, rank[i]记录了从head到update[i]的步长
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    // 元素相同的话按字典序从小到大
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        // update数组记录搜索member所在位置时在level数组中的转折点
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    // 获取随机层数
    level = zslRandomLevel();
    if (level > zsl->level) {
        // 生成的随机层数 > 目前跳表的最高层
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            // 先初始化为zsl->length, 下面会更新
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    // 为node分配内存
    x = zslCreateNode(level,score,ele);
    // 把x插入到update[i]后面
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // backward是回退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        // 设置下一个结点的backward指针
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}

```


### 为什么zset使用跳表而不是平衡二叉树, 如红黑树, B树

[https://news.ycombinator.com/item?id=1171423](https://news.ycombinator.com/item?id=1171423)

有人关于skiplist提了自己的几点质疑:

1. skiplist 会因为指针比b树占用更多内存
2. 没有利用局部性原理导致cache失效

另外建议创建多个线程提高cpu利用率和吞吐量

作者回答:

1. zset的skip list的索引结点数量是可控制的, 可以通过ZSKIPLIST_P控制, 从而优化内存占用
2. sorted set的常用操作是按序遍历, 也就是说将跳跃表作为链表遍历. 这个操作在缓存命中上不比平衡树差
3. skip list很容易实现, 调试, 方便扩展

关于线程, 作者说经验表明redis的瓶颈在I/O, 假设你的服务能让单核满负载, 可以在同一台机器部署多个redis实例, 使用redis cluster

### redis的zset实现的skiplist做了哪些优化

1. member可以重复, 重复元素根据字典序排序
2. 增加后退指针, 可以在zrevrange操作中, 找到起始结点后利用后退指针backward回退查找

参考

原论文地址: [Skip Lists:A Probabilistic Alternative to Balanced Trees](http://homepage.cs.uiowa.edu/~ghosh/skip.pdf)

[作者关于为什么使用skiplist的说明](https://news.ycombinator.com/item?id=1171423)

[stackoverflow关于位运算 |= 的解释](https://stackoverflow.com/questions/14295469/what-does-mean-pipe-equal-operator)
