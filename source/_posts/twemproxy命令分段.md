---
title: twemproxy命令分段
date: 2024-09-01 21:30:05
categories:
    - twemproxy
---
# 起源
twemproxy作为路由层，需要判断命令的转发，比如mget命令涉及的多个key涉及多个底层redis节点。
``` bash
MGET key1 key2 key3
key1 -> redis1:6379
key2 -> redis2:6379
key3 -> redis3:6379
```
则proxy需要做命令分段转发，并最终聚合返回给客户端的操作。


# 原理解析

如下图所示，先将原命令拆分成子命令，并发送子命令给对应的redis节点
![](msg_frag.svg)

假如此时某个redis节点回复了sub_msg，会将对应计数+1，当计数值等于nfrag，说明全部sub_msg收到，则对命令回复做聚合，再回复给用户侧。



# 源码解析

1. 成功解析完了一条命令，放置于msg后，调用分段函数。

``` c
// nc_request.c/req_recv_done()
void
req_recv_done(struct context *ctx, struct conn *conn, struct msg *msg,
              struct msg *nmsg)
{
    rstatus_t status;
    struct server_pool *pool;
    struct msg_tqh frag_msgq;
    ...
    // 此处分段
    status = msg->fragment(msg, array_n(&pool->server), &frag_msgq);
    ...
}
```

2. 假设backend为redis

``` c
rstatus_t
redis_fragment(struct msg *r, uint32_t nserver, struct msg_tqh *frag_msgq)
{
    // key只有一个，无需分段，返回
    if (1 == array_n(r->keys)){
        return NC_OK;
    }

    switch (r->type) {
    case MSG_REQ_REDIS_MGET:
    case MSG_REQ_REDIS_DEL:
    case MSG_REQ_REDIS_TOUCH:
    case MSG_REQ_REDIS_UNLINK:
        return redis_fragment_argx(r, nserver, frag_msgq, 1);

    // mset
    case MSG_REQ_REDIS_MSET:
        return redis_fragment_argx(r, nserver, frag_msgq, 2);

    default:
        return NC_OK;
    }
}
```

3. 以mget为例，其参数个数即为需要转发判断的key数量

下面的代码即是分段的过程，其实写的很好很清晰。最终会将分段的`submsg`挂在`msg_tqh`队列上。

已经给了详细注释。
``` c
// 需要熟悉RESP协议
// https://redis.io/docs/latest/develop/reference/protocol-spec/
static rstatus_t
redis_fragment_argx(struct msg *r, uint32_t nserver, struct msg_tqh *frag_msgq,
                    uint32_t key_step)
{
    struct mbuf *mbuf;
    struct msg **sub_msgs;
    uint32_t i;
    rstatus_t status;
    struct array *keys = r->keys;

    ASSERT(array_n(keys) == (r->narg - 1) / key_step);

    // 预先申请server数量的指针数组，用于分段操作，最终会有一些指针为null，即不会有key路由到对应的server
    sub_msgs = nc_zalloc(nserver * sizeof(*sub_msgs));
    ...
    ASSERT(r->frag_seq == NULL);
    // frag_seq是指针数组，与key数量同长，每一个都是msg指针，指向对应的submsg，若有多个key分到一个submsg，这里会体现出来
    r->frag_seq = nc_alloc(array_n(keys) * sizeof(*r->frag_seq));
    ...

    mbuf = STAILQ_FIRST(&r->mhdr);
    mbuf->pos = mbuf->start;

    // 根据RESP协议，先把前三行给处理掉，从key的实际值开始处理key
    for (i = 0; i < 3; i++) {                 /* eat *narg\r\n$4\r\nMGET\r\n */
        for (; *(mbuf->pos) != '\n';) {
            mbuf->pos++;
        }
        mbuf->pos++;
    }

    r->frag_id = msg_gen_frag_id(); //frag_id在全部的分段中相同
    r->nfrag = 0;   // 最终有多少个submsg
    r->frag_owner = r;  // 所有的msg的owner都是最初的msg

    // 下面所有代码的处理过程都在构建submsg的命令格式，先构造submsg键的部分，然后往mbuf头部插入命令名 
    for (i = 0; i < array_n(keys); i++) {        /* for each key */
        struct msg *sub_msg;
        struct keypos *kpos = array_get(keys, i);
        uint32_t idx = msg_backend_idx(r, kpos->start, kpos->end - kpos->start);
        ASSERT(idx < nserver);

        if (sub_msgs[idx] == NULL) {
            sub_msgs[idx] = msg_get(r->owner, r->request, r->redis);
            if (sub_msgs[idx] == NULL) {
                nc_free(sub_msgs);
                return NC_ENOMEM;
            }
        }
        r->frag_seq[i] = sub_msg = sub_msgs[idx];

        sub_msg->narg++;
        status = redis_append_key(sub_msg, kpos->start, kpos->end - kpos->start);
        if (status != NC_OK) {
            nc_free(sub_msgs);
            return status;
        }

        if (key_step == 1) {                            /* mget,del */
            continue;
        } else {                                        /* mset */
            status = redis_copy_bulk(NULL, r);          /* eat key */
            if (status != NC_OK) {
                nc_free(sub_msgs);
                return status;
            }

            status = redis_copy_bulk(sub_msg, r);
            if (status != NC_OK) {
                nc_free(sub_msgs);
                return status;
            }

            sub_msg->narg++;
        }
    }

    /*
     * prepend mget header, and forward the command (command type+key(s)+suffix)
     * to the corresponding server(s)
     */
    for (i = 0; i < nserver; i++) {
        struct msg *sub_msg = sub_msgs[i];
        // 这步需要理解含义，因为有些server没有对应key路由过去，那么跳过
        if (sub_msg == NULL) {
            continue;
        }

        if (r->type == MSG_REQ_REDIS_MGET) {
            status = msg_prepend_format(sub_msg, "*%d\r\n$4\r\nmget\r\n",
                                        sub_msg->narg + 1);
        } else if (r->type == MSG_REQ_REDIS_DEL) {
            status = msg_prepend_format(sub_msg, "*%d\r\n$3\r\ndel\r\n",
                                        sub_msg->narg + 1);
        } else if (r->type == MSG_REQ_REDIS_MSET) {
            status = msg_prepend_format(sub_msg, "*%d\r\n$4\r\nmset\r\n",
                                        sub_msg->narg + 1);
        } else if (r->type == MSG_REQ_REDIS_TOUCH) {
            status = msg_prepend_format(sub_msg, "*%d\r\n$5\r\ntouch\r\n",
                                        sub_msg->narg + 1);
        } else if (r->type == MSG_REQ_REDIS_UNLINK) {
            status = msg_prepend_format(sub_msg, "*%d\r\n$6\r\nunlink\r\n",
                                        sub_msg->narg + 1);
        } else {
            NOT_REACHED();
        }
        if (status != NC_OK) {
            nc_free(sub_msgs);
            return status;
        }

        sub_msg->type = r->type;
        sub_msg->frag_id = r->frag_id;
        sub_msg->frag_owner = r->frag_owner;
        // 这里将构造的sub_msg插入了frag_msgq，后续即围绕这个队列做处理
        TAILQ_INSERT_TAIL(frag_msgq, sub_msg, m_tqe);
        r->nfrag++;
    }

    nc_free(sub_msgs);
    return NC_OK;
}
```

4. 将origin-msg放到c_conn的out_q（只放了一个msg），再逐个转发submsg，并都放到s_conn的imsg_q

``` c

    for (sub_msg = TAILQ_FIRST(&frag_msgq); sub_msg != NULL; sub_msg = tmsg) {
        tmsg = TAILQ_NEXT(sub_msg, m_tqe);

        TAILQ_REMOVE(&frag_msgq, sub_msg, m_tqe);
        req_forward(ctx, conn, sub_msg);
    }
```

5. proxy收到一条完整msg回复，会将小段的sub_msg从s_conn的out_q中pop出来，并增加收到的分段计数，显然计数等于分段数时，可以开始聚合处理

``` c
// nc_response.c
static void
rsp_forward(struct context *ctx, struct conn *s_conn, struct msg *msg)
{
    ...
    /* dequeue peer message (request) from server */
    pmsg = TAILQ_FIRST(&s_conn->omsg_q);
    // 取出sub_msg
    s_conn->dequeue_outq(ctx, s_conn, pmsg);
    pmsg->done = 1;

    /* establish msg <-> pmsg (response <-> request) link */
    pmsg->peer = msg;
    msg->peer = pmsg;
    // 预聚合
    msg->pre_coalesce(msg);
    // 判断是否整个命令分段全部处理完毕
    if (req_done(c_conn, TAILQ_FIRST(&c_conn->omsg_q))) {
        status = event_add_out(ctx->evb, c_conn);
        if (status != NC_OK) {
            c_conn->err = errno;
        }
    }

    rsp_forward_stats(ctx, s_conn->owner, msg, msgsize);
}
```

``` c
// nc_redis.c
void
redis_pre_coalesce(struct msg *r)
{
    // 因为传参为rsp，peer为sub_msg
    struct msg *pr = r->peer; /* peer request */
    struct mbuf *mbuf;
    // 非分段请求不做任何操作
    if (pr->frag_id == 0) {
        /* do nothing, if not a response to a fragmented request */
        return;
    }
    // sub_msg完成个数计数+1
    pr->frag_owner->nfrag_done++;
    ...
}
```

6. 分段判断是否收集完毕
``` c
bool
req_done(const struct conn *conn, struct msg *msg)
{
    struct msg *cmsg;        /* current message */
    uint64_t id;             /* fragment id */
    uint32_t nfragment;      /* # fragment */

    ASSERT(conn->client && !conn->proxy);
    ASSERT(msg->request);

    if (!msg->done) {
        return false;
    }
    // 非分段命令，不进行任何操作
    id = msg->frag_id;
    if (id == 0) {
        return true;
    }

    if (msg->fdone) {
        /* request has already been marked as done */
        return true;
    }
    // 核心位置，若没有收集完整不会往后走，直接返回false
    if (msg->nfrag_done < msg->nfrag) {
        return false;
    }

    /* check all fragments of the given request vector are done */
    // TODO: 下面的检查我还不理解为什么要做，理论上下面的全部检查都是重复的才对，我猜测redis可能会有重复回复，所以前面判断不准确，需要额外检查
    // TODO: 找了很久没找到哪里把这些msg连起来的，看代码压根没连起来，所以下面代码好像没起作用
    for (cmsg = TAILQ_PREV(msg, msg_tqh, c_tqe);
         cmsg != NULL && cmsg->frag_id == id;
         cmsg = TAILQ_PREV(cmsg, msg_tqh, c_tqe)) {

        if (!cmsg->done) {
            return false;
        }
    }
    // 往后遍历
    for (cmsg = TAILQ_NEXT(msg, c_tqe);
         cmsg != NULL && cmsg->frag_id == id;
         cmsg = TAILQ_NEXT(cmsg, c_tqe)) {

        if (!cmsg->done) {
            return false;
        }
    }

    /*
     * At this point, all the fragments including the last fragment have
     * been received.
     *
     * Mark all fragments of the given request vector to be done to speed up
     * future req_done calls for any of fragments of this request
     */

    msg->fdone = 1;
    nfragment = 0;

    for (cmsg = TAILQ_PREV(msg, msg_tqh, c_tqe);
         cmsg != NULL && cmsg->frag_id == id;
         cmsg = TAILQ_PREV(cmsg, msg_tqh, c_tqe)) {
        cmsg->fdone = 1;
        nfragment++;
    }

    for (cmsg = TAILQ_NEXT(msg, c_tqe);
         cmsg != NULL && cmsg->frag_id == id;
         cmsg = TAILQ_NEXT(cmsg, c_tqe)) {
        cmsg->fdone = 1;
        nfragment++;
    }

    ASSERT(msg->frag_owner->nfrag == nfragment);

    msg->post_coalesce(msg->frag_owner);

    log_debug(LOG_DEBUG, "req from c %d with fid %"PRIu64" and %"PRIu32" "
              "fragments is done", conn->sd, id, nfragment);

    return true;
}
```

7. 聚合操作`msg->post_coalesce(msg->frag_owner)`，以mget为例

``` c
/*
 * Post-coalesce handler is invoked when the message is a response to
 * the fragmented multi vector request - 'mget' or 'del' and all the
 * responses to the fragmented request vector has been received and
 * the fragmented request is consider to be done
 */
void
redis_post_coalesce(struct msg *r)
{
    struct msg *pr = r->peer; /* peer response */

    ASSERT(!pr->request);
    ASSERT(r->request && (r->frag_owner == r));
    if (r->error || r->ferror) {
        /* do nothing, if msg is in error */
        return;
    }

    switch (r->type) {
    case MSG_REQ_REDIS_MGET:
        return redis_post_coalesce_mget(r);

    case MSG_REQ_REDIS_DEL:
    case MSG_REQ_REDIS_TOUCH:
    case MSG_REQ_REDIS_UNLINK:
        return redis_post_coalesce_del_or_touch(r);

    case MSG_REQ_REDIS_MSET:
        return redis_post_coalesce_mset(r);

    default:
        NOT_REACHED();
    }
}
```

``` c
static void
redis_post_coalesce_mget(struct msg *request)
{   
    // 这个response的msg实际是空的
    struct msg *response = request->peer;
    struct msg *sub_msg;
    rstatus_t status;
    uint32_t i;
    // 往mbuf写resp协议需要的回复字符串个数
    status = msg_prepend_format(response, "*%d\r\n", request->narg - 1);
    if (status != NC_OK) {
        /*
         * the fragments is still in c_conn->omsg_q, we have to discard all of them,
         * we just close the conn here
         */
        response->owner->err = 1;
        return;
    }
    for (i = 0; i < array_n(request->keys); i++) {      /* for each key */
        // peer对应的是sub_msg的rsp
        sub_msg = request->frag_seq[i]->peer;           /* get it's peer response */
        if (sub_msg == NULL) {
            response->owner->err = 1;
            return;
        }
        // 这个只会copy一个bulk，也就是一个key
        status = redis_copy_bulk(response, sub_msg);
        if (status != NC_OK) {
            response->owner->err = 1;
            return;
        }
    }
}
```

8. 到这里已经回复聚合完毕了，之后回复流程和一般情况无差异


