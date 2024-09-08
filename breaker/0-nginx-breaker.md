# nginx熔断机制源代码分析

参看:

- [nginx的负载均衡](https://www.cnblogs.com/ExMan/p/11879190.html)

## 1. 数据结构

一个upstream的配置类似于如下(参看: https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server):

```
upstream loadbalance_1{
  server 192.168.1.20:8080 weight=8 max_fails=1 fail_timeout=3s;
  server 192.168.1.20:8081 weight=8 max_fails=1 fail_timeout=3s;
  server 192.168.1.20:8082 weight=8 max_fails=1 fail_timeout=3s;
}
```

需要注意的是，如果server指令的第一个参数是IP和端口，那么一条server指令只对应一个真实节点。如果server指令的第一个参数是域名，一条server指令可能对应多个真实节点。

他们的server成员是相同的，可以通过name成员区分。

接下来先介绍几个所关联到的数据结构：

- ngx_http_upstream_rr_peers_t：保存着一个upstream的所有peers信息

```
typedef struct ngx_http_upstream_rr_peers_s  ngx_http_upstream_rr_peers_t;

struct ngx_http_upstream_rr_peers_s {
    ngx_uint_t                      number;                 //后端服务器(peer)的数量

#if (NGX_HTTP_UPSTREAM_ZONE)
    ngx_slab_pool_t                *shpool;
    ngx_atomic_t                    rwlock;
    ngx_http_upstream_rr_peers_t   *zone_next;
#endif

    ngx_uint_t                      total_weight;           //总的权重(包括配置为down状态的server)
    ngx_uint_t                      tries;                  //所有配置为非down状态server peer的数量

    unsigned                        single:1;               //是否只有一台后端服务器
    unsigned                        weighted:1;             //是否使用权重

    ngx_str_t                      *name;                   //upstream配置块的名称

    ngx_http_upstream_rr_peers_t   *next;                   //backup服务器集群

    ngx_http_upstream_rr_peer_t    *peer;                   //后端服务器组成的链表
};

```

ngx_http_upstream_rr_peers_t的初始化是在ngx_http_upstream_init_round_robin()中完成的：

```
//src/http/ngx_http_upstream_round_robin.c

ngx_int_t
ngx_http_upstream_init_round_robin(ngx_conf_t *cf,
    ngx_http_upstream_srv_conf_t *us)
{
}
```

- ngx_http_upstream_rr_peer_t: 代表着后端的一台远程主机

```
//src/http/ngx_http_upstream_round_robin.h

typedef struct ngx_http_upstream_rr_peer_s   ngx_http_upstream_rr_peer_t;

struct ngx_http_upstream_rr_peer_s {
    struct sockaddr                *sockaddr;             //后端远程主机的服务器地址
    socklen_t                       socklen;              //地址的长度
    ngx_str_t                       name;                 //后端服务器地址的字符串, server.addrs[i].name
    ngx_str_t                       server;               //server的名称，server.name

    ngx_int_t                       current_weight;       //当前的权重，动态调整，初始值为0
    ngx_int_t                       effective_weight;     //有效的权重，会因为失败而降低
    ngx_int_t                       weight;               //配置项指定的权重，固定值

    ngx_uint_t                      conns;                //当前的连接数
    ngx_uint_t                      max_conns;            //每一个peer的最大连接数

    ngx_uint_t                      fails;                //'一段时间'内，已经失败的次数
    time_t                          accessed;             //最近以此失败的时间点
    time_t                          checked;              //用于检查是否超过了'一段时间'

    ngx_uint_t                      max_fails;            //'一段时间'内，最大的失败次数，固定值
    time_t                          fail_timeout;         //'一段时间'的值，固定值
    ngx_msec_t                      slow_start;
    ngx_msec_t                      start_time;

    ngx_uint_t                      down;                 //服务器永久不可用标志

#if (NGX_HTTP_SSL || NGX_COMPAT)
    void                           *ssl_session;
    int                             ssl_session_len;
#endif

#if (NGX_HTTP_UPSTREAM_ZONE)
    ngx_atomic_t                    lock;
#endif

    ngx_http_upstream_rr_peer_t    *next;                 //指向下一个后端，用于构成链表

    NGX_COMPAT_BEGIN(32)
    NGX_COMPAT_END
};
```
这里关于'max_fails'与'fail_timeout'再简单加以说明：Nginx基于连接探测，如果发现后端异常，在单位周期为fail_timeout设置的时间中达到max_fails次数，则这个周期内将把节点标记为不可用，并等待下一个周期（同样时长为fail_timeout)再一次去请求，判断连接是否成功。如果成功，将恢复之前的轮询方式，如果仍然不可用将在下一个周期(fail_timeout)再试一次。


参看：
```
1. https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server

2. https://blog.51cto.com/kexiaoke/2316851
```

- ngx_http_upstream_rr_peer_data_t: round-robin的per request负载均衡数据

```
//src/http/ngx_http_upstream_round_robin.h

typedef struct {
    ngx_uint_t                      config;
    ngx_http_upstream_rr_peers_t   *peers;            //后端集群
    ngx_http_upstream_rr_peer_t    *current;          //当前使用的后端服务器
    uintptr_t                      *tried;            //指向后端服务器的位图
    uintptr_t                       data;             //当后端服务器数量较少时，用于存放其位图
} ngx_http_upstream_rr_peer_data_t;

```


## 2. round-robin方式下如何实现负载均衡和熔断

主要涉及到peer的选取和peer的释放，接下来我们分析。

### 2.1 初始化http request从upstream中获取peer

upstream round-robin的初始化是在ngx_http_upstream_init_round_robin_peer()中实现的：

```
ngx_int_t
ngx_http_upstream_init_round_robin_peer(ngx_http_request_t *r,
    ngx_http_upstream_srv_conf_t *us)
{
	r->upstream->peer.get = ngx_http_upstream_get_round_robin_peer;
    r->upstream->peer.free = ngx_http_upstream_free_round_robin_peer;
    r->upstream->peer.tries = ngx_http_upstream_tries(rrp->peers);
}
```

从上面可以看到，upstream是通过回调ngx_http_upstream_get_round_robin_peer()来获取连接；通过ngx_http_upstream_free_round_robin_peer()来释放连接；r->upstream->peer.tries被初始化成了所有配置为非down状态server peer的数量（包括backup的server)

### 2.2 peer的选取
round-robin方式下peer的选取是通过调用ngx_http_upstream_get_round_robin_peer()函数实现的：

```
//src/http/ngx_http_upstream_round_robin.c

ngx_int_t
ngx_http_upstream_get_round_robin_peer(ngx_peer_connection_t *pc, void *data)
{
	...

    if (peers->single) {
        peer = peers->peer;

        if (peer->down) {
            goto failed;
        }

        if (peer->max_conns && peer->conns >= peer->max_conns) {
            goto failed;
        }

        rrp->current = peer;

    } else {

        /* there are several peers */

        peer = ngx_http_upstream_get_peer(rrp);

        if (peer == NULL) {
            goto failed;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                       "get rr peer, current: %p %i",
                       peer, peer->current_weight);
    }

	...
}

```
我们看到实现很简单，如果是单peer的话，那么直接选择；如果是多peers，则调用ngx_http_upstream_get_peer()。这里在讲解该函数之前我们还有必要再细述一下ngx_htp_upstream_rr_peer_t结构中的几个变量：

```
typedef struct ngx_http_upstream_rr_peer_s   ngx_http_upstream_rr_peer_t;

struct ngx_http_upstream_rr_peer_s {
    ngx_uint_t                      fails;                //'一段时间'内，已经失败的次数
    time_t                          accessed;             //最近以此失败的时间点
    time_t                          checked;              //用于检查是否超过了'一段时间'

    ngx_uint_t                      max_fails;            //'一段时间'内，最大的失败次数，固定值
    time_t                          fail_timeout;         //'一段时间'的值，固定值
};
```

如下图所示：

![quic-layer](https://raw.githubusercontent.com/ivanzz1001/distribute_storage_system/master/breaker/image/ngx-rr-break.jpg)

从上面我们可以看到fail_timeout时间段是不连续的，它是由upstream request在一个时间段结束之后来开启一个新的fail_timeout时间段。

在开启一个新的fail_timeout时，首先会将checked设置为当前时间；之后每当受到一个失败的响应，会将accessed以及checked都设置为当前时间。因此从这里我们知道accessed保存的是最近一次fail的时间，而checked会跟随accessed往前移动，并且有如下：

```
if(checked == accessed){
	//一定处于同一个fail_timeout时间段，且当前至少有一次失败情况发生
}else if (accessed < checked){
	//说明accessed所在的时间段落后于checked所在的时间段
}
```

接下来我们来看ngx_http_upstream_get_peer()的实现：

```
//src/http/ngx_http_upstream_round_robin.c

static ngx_http_upstream_rr_peer_t *
ngx_http_upstream_get_peer(ngx_http_upstream_rr_peer_data_t *rrp)
{
    time_t                        now;
    uintptr_t                     m;
    ngx_int_t                     total;
    ngx_uint_t                    i, n, p;
    ngx_http_upstream_rr_peer_t  *peer, *best;

    now = ngx_time();

    best = NULL;
    total = 0;

#if (NGX_SUPPRESS_WARN)
    p = 0;
#endif


	//遍历upstream所配置的所有后端服务器(ps：不包括backup的）
    for (peer = rrp->peers->peer, i = 0;
         peer;
         peer = peer->next, i++)
    {
        n = i / (8 * sizeof(uintptr_t));
        m = (uintptr_t) 1 << i % (8 * sizeof(uintptr_t));


		//通过查询per request负载均衡数据所关联的位图，如果该peer已经尝试连接过了，则continue
        if (rrp->tried[n] & m) {
            continue;
        }


		//如果该peer处于down状态，则continue 
        if (peer->down) {
            continue;
        }


		//若满足下面3个条件，则说明当前peer出现故障，此时直接continue 
		//1) 配置了最大失败次数；
		//2) 当前失败次数大于等于所配置的最大失败次数
		//3) 当前时间与peer->checked在同一个fail_timeout时间段内
        if (peer->max_fails
            && peer->fails >= peer->max_fails
            && now - peer->checked <= peer->fail_timeout)
        {
            continue;
        }

		
		//若当前peer的连接数已经超过了最大配置值，则continue
        if (peer->max_conns && peer->conns >= peer->max_conns) {
            continue;
        }


		//修改peer的权重相关信息（当前权重、有效权重）
        peer->current_weight += peer->effective_weight;
        total += peer->effective_weight;

        if (peer->effective_weight < peer->weight) {
            peer->effective_weight++;
        }


		//选出具有最高权重的peer保存在best中
        if (best == NULL || peer->current_weight > best->current_weight) {
            best = peer;
            p = i;
        }
    }

    if (best == NULL) {
        return NULL;
    }

    rrp->current = best;

	//更新位图，表明peer已经使用过了
    n = p / (8 * sizeof(uintptr_t));
    m = (uintptr_t) 1 << p % (8 * sizeof(uintptr_t));

    rrp->tried[n] |= m;


	//这里total保存了所有可用peer的权重的累加和，由于best我们已经选中了，那么就需要调整其权重，以便下一次
	//执行round-robin时，我们可以选到其他peer
    best->current_weight -= total;


	//如果满足一个新的fail_timeout时间段，则更新checked
    if (now - best->checked > best->fail_timeout) {
        best->checked = now;
    }

    return best;
}


```

### 2.3 peers的释放

```
//src/http/ngx_http_upstream_round_robin.c

void
ngx_http_upstream_free_round_robin_peer(ngx_peer_connection_t *pc, void *data,
    ngx_uint_t state)
{
    ngx_http_upstream_rr_peer_data_t  *rrp = data;

    time_t                       now;
    ngx_http_upstream_rr_peer_t  *peer;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                   "free rr peer %ui %ui", pc->tries, state);

    /* TODO: NGX_PEER_KEEPALIVE */

    peer = rrp->current;

    ngx_http_upstream_rr_peers_rlock(rrp->peers);
    ngx_http_upstream_rr_peer_lock(rrp->peers, peer);

    if (rrp->peers->single) {

        peer->conns--;

        ngx_http_upstream_rr_peer_unlock(rrp->peers, peer);
        ngx_http_upstream_rr_peers_unlock(rrp->peers);

        pc->tries = 0;
        return;
    }

    if (state & NGX_PEER_FAILED) {
        now = ngx_time();

        peer->fails++;
        peer->accessed = now;
        peer->checked = now;

        if (peer->max_fails) {
            peer->effective_weight -= peer->weight / peer->max_fails;

            if (peer->fails >= peer->max_fails) {
                ngx_log_error(NGX_LOG_WARN, pc->log, 0,
                              "upstream server temporarily disabled");
            }
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                       "free rr peer failed: %p %i",
                       peer, peer->effective_weight);

        if (peer->effective_weight < 0) {
            peer->effective_weight = 0;
        }

    } else {

        /* mark peer live if check passed */

        if (peer->accessed < peer->checked) {
            peer->fails = 0;
        }
    }

    peer->conns--;

    ngx_http_upstream_rr_peer_unlock(rrp->peers, peer);
    ngx_http_upstream_rr_peers_unlock(rrp->peers);

    if (pc->tries) {
        pc->tries--;
    }
}

```

这里peer的释放比较简单，就是根据request在该peer上执行的状态，调整peer的fails、accessed、checked、effective_weight信息。