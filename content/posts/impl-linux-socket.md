+++
title = "实现简单的 socket 子协议"
date = "2025-02-13"

[taxonomies]
tags = ["entry"]
+++

驱动层实现 socket 的接口的例子，先看看 `socket` 是怎么调用的：

```clike
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
```

三个字段，看看内核是怎么一步一步调用到后面的（还需要包括注册接口的部分），当然 read 和 write 这些接口也都是要去了解的（。

* socket
* bind (server)
* connect (client)
* read
* write

# syscall socket

先从 `SYSCALL_DEFINE3(socket)` 开始，

```clike
__sys_socket(family, type, protocol)
  struct socket *sock
  sock_create(family, type, protocol, &sock)
    __sock_create(current->nsproxy->net_ns, family, type, protocol, sock, 0)
  sock_map_fd(sock, ...)
```

```clike
__sock_create(net, family, type, protocol, res, kern)
  struct socket *sock = sock_alloc()
  sock->type = type;
  struct net_proto_family *pf = net_families[family]
                                ^^^^^^^^^^^^ sock_register(ops)
                                               net_families[ops->family] = ops
  pf->create(net, sock, protocol, kern)
```

这里就开始有函数指针抽象了，就是在 `sock_register` 里面注册的。

# write

看看写入，还是从 syscall 开始， `SYSCALL_DEFINE3(write)`：

```clike
ksys_write(fd, buf, count)
  struct fd f = fdget_pos(fd)
  ppos = file_ppos(f.file)
  vfs_write(f.file, buf, count, ppos)
    if f.file->f_op->write?(f.file, buf, count, ppos)
    elif f.file->f_op->write_iter?
      new_sync_write(f.file, buf, count, ppos)
        ...
        f.file->f_op->write_iter(...)
```

又是 ops，这里大概能定位到是 `socket_file_ops`，追溯一下怎么跟 fd 关联起来的，

```clike
__sys_socket
  sock_map_fd
    sock_alloc_file
      alloc_file_pseudo(SOCK_INODE(sode), sock_mnt, dname, ..., &socket_file_ops)
```

好吧，关联回来了，就是在创建 socket 的时候注册了。所以在 socket 的时候会创建 socket 以及 net proto，那就继续往下层 `write_iter` 看看。

```clike
sock_write_iter(iocb, from)
  file = iocb->ki_filp
  struct msghdr msg = {.msg_iter = *from, .msg_iocb = iocb}
  if (file->f_flags & O_NONBLOCK || ...)
    msg.msg_flags = MSG_DONTWAIT // can use the msg_flags for timeout or else
  sock_sendmsg(sock, &msg)
    sock_sendmsg_nosec(sock, &msg)
      sock->ops->sendmsg(sock, &msg, msg_data_left(msg))
            ^^^ inet_create
                  struct net_proto_family inet_family_ops = {.create = inet_create}
                    sock_register(...)
```

这里出现了我们想要的 sock→ops，可以看到就是通过 `struct net_proto_family::create` 创建出来的，这就回到了最开始的 `socket` 创建。

后面的一些调用其实不怎么看也行，基本上就是这样的调用链：

syscall → socket_file_ops (generic) → struct proto_ops (sock_register)

中间 socket 层传递的是 struct socket，到了 proto 层后传递的就是 struct sock，个人理解 `struct net_proto_family` 更多的是像一层胶水层，将 socket 和 sock 绑定到一起？

后面其实还有一层 `struct proto`，这个就是真正具体的协议层次的实现了。

# proto

好吧，实际上 `struct proto` 还是要参与内核的，更好的设计来说 ops 应该就要是 `struct proto` 的一部分，不知道为什么要单独分离。

换言之你要进行的例如 `sk_alloc` 等都依赖 `proto_register`，并且最后会使用 `sock_init_data` 将 sock 绑定到 socket 上。相当于 sock 实际上就是 socket 的一个 impl。好吧其实还是不太对的样子，可能需要再好好梳理一下。

## `proto_register`

这个函数实际上也没干啥事呢，如果参数 `alloc_slab` 为真的话就会通过 `kmem_cache_create` 创建一个 slab 内存池。然后给 `prot→inuse_idx` 赋值，加入到 `proto_list` 链表中，这些操作基本上都是在给 proc 节点使用的，类似调试信息？

所以这个 register 感觉真没啥用呢。。看着 `struct proto` 实际上很多函数指针成员都是重复的？

## `sk_alloc`

这里基本上就是 `struct proto` 的绑定的地方，作为参数，所以来看看有什么发现。

```clike
sk_alloc(net, family, priority, struct proto* prot, kern)
  struct sock *sk = sk_prot_alloc(prot, priority | __GFP_ZERO, family)
    if !slab kmalloc(prot->obj_size, priority)
  sk->sk_family = family
  sk->sk_prot = sk->sk_prot_creator = prot
  sk->sk_kern_sock = kern
```

这里指出了两个地方， `sk_prot` 和 `sk_prot_creator`（感觉自己在人肉 DFS），看看 xref？

貌似没怎么见到有意思的调用。

目前看下来感觉 proto 只是为了架构的统一性所以保留了，例如 IPv4 TCP，那么 `proto_ops` 会是 IPv4，然后通过 `sk→sk_prot` 最后转发给 TCP 协议处理。

如果本身协议不复杂，那么 `struct proto` 多数字段都是没啥用的，但是为了架构的统一，所以就算是简单的 CAN 等通信也要实现一层 proto。

# back to socket

这三个层级貌似跟 `socket` 设计还是有关的，三个参数，每个参数基本都代表了一个抽象层（函数 socket 自身也被封装在了 struct file 之下）。

最后一层的 `struct proto` 基本上就是协议自己在使用了，貌似没看到 linux 内核有专门的操作，对于模块实现来说，只需要实现到 `struct proto_ops` 基本就够了（具体还是看实现协议的复杂和抽象程度）。

总结来说就是，在 module init 的层次上，

* 需要 `proto_register` 注册一个 `struct proto`
* 需要 `sock_register` 注册 `struct net_proto_family`，包含 create 函数

之后 socket 会调用 create 函数，它需要

* 使用 `sk_alloc` 分配 sock 内存
* 将 `struct proto_ops` 绑定到 socket 上
* 初始化 sock，一般也都需要设置 `sock::sk_destruct` 函数。

这里都可以参考 `inet_create` 之类的函数。
