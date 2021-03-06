上线准备

1 多租户权限

- 应用和虚拟主机一一对应,即一个应用对应一个虚拟主机.
比如系统分为多个业务模块,订单业务对应一个虚拟主机,数据业务对应一个虚拟主机.

- 每个微服务实例应用对应一个用户,当有一个应用实例多个用户,则需要考虑安全性和便利性.

- 评估全部应用所需要的队列,消息量和大小,修改用户的资源限制配置.

2 系统默认参数优化

通过`rabbitmqctl status`可以看到当前节点的参数信息.

    root@5d8b5909379c:/# rabbitmqctl status
    Status of node rabbit@5d8b5909379c
    [{pid,228},
     {running_applications,
         [{rabbitmq_management,"RabbitMQ Management Console","3.6.10"},
          {rabbitmq_web_dispatch,"RabbitMQ Web Dispatcher","3.6.10"},
          {rabbitmq_management_agent,"RabbitMQ Management Agent","3.6.10"},
          {rabbit,"RabbitMQ","3.6.10"},
          {mnesia,"MNESIA  CXC 138 12","4.14.2"},
          {inets,"INETS  CXC 138 49","6.3.4"},
          {amqp_client,"RabbitMQ AMQP Client","3.6.10"},
          {rabbit_common,
              "Modules shared by rabbitmq-server and rabbitmq-erlang-client",
              "3.6.10"},
          {xmerl,"XML parser","1.3.12"},
          {cowboy,"Small, fast, modular HTTP server.","1.0.4"},
          {cowlib,"Support library for manipulating Web protocols.","1.0.2"},
          {ranch,"Socket acceptor pool for TCP protocols.","1.3.0"},
          {ssl,"Erlang/OTP SSL application","8.1"},
          {public_key,"Public key infrastructure","1.3"},
          {crypto,"CRYPTO","3.7.2"},
          {os_mon,"CPO  CXC 138 46","2.4.1"},
          {compiler,"ERTS  CXC 138 10","7.0.3"},
          {syntax_tools,"Syntax tools","2.1.1"},
          {asn1,"The Erlang ASN1 compiler version 4.0.4","4.0.4"},
          {sasl,"SASL  CXC 138 11","3.0.2"},
          {stdlib,"ERTS  CXC 138 10","3.2"},
          {kernel,"ERTS  CXC 138 10","5.1.1"}]},
     {os,{unix,linux}},
     {erlang_version,
         "Erlang/OTP 19 [erts-8.2.1] [source] [64-bit] [smp:2:2] [async-threads:64] [hipe] [kernel-poll:true]\n"},
     {memory,
         [{total,71868640},
          {connection_readers,0},
          {connection_writers,0},
          {connection_channels,0},
          {connection_other,2832},
          {queue_procs,2832},
          {queue_slave_procs,0},
          {plugins,1327784},
          {other_proc,20779968},
          {mnesia,61624},
          {metrics,193408},
          {mgmt_db,342104},
          {msg_index,42584},
          {other_ets,2509304},
          {binary,462472},
          {code,24680786},
          {atom,1033401},
          {other_system,20620117}]},
     {alarms,[]},
     {listeners,[{clustering,25672,"::"},{amqp,5672,"::"},{http,15672,"::"}]},
     {vm_memory_high_watermark,0.4},
     {vm_memory_limit,3297615872},
     {disk_free_limit,50000000},
     {disk_free,82436243456},
     {file_descriptors,
         [{total_limit,1048476},
          {total_used,2},
          {sockets_limit,943626},
          {sockets_used,0}]},
     {processes,[{limit,1048576},{used,322}]},
     {run_queue,0},
     {uptime,12495},
     {kernel,{net_ticktime,60}}]

其中几点比较重要的:
running_applications 应用运行的一些参数
memory 内存参数
listeners 端口相关的信息
vm_memory_high_watermark 内存高使用率标记,超过这个值,将停止接收新消息 ,默认是0.4
vm_memory_limit 内存限制大小 默认大小是:系统内存*vm_memory_high_watermark,例如我的机器是 8G内存,那么可用的内存为8*0.4=3.2G,也就是3G左右的可用内存
disk_free_limit 默认50M
disk_free 当前空闲磁盘空间
file_descriptors 文件描述符设置
processes


对于`vm_memory_high_watermark`官方给的建议:
- 托管RabbitMQ的节点至少应该有128MB的内存可用。
- 推荐的vm_memory_high_watermark范围是  0.40到0.66
- 不推荐使用0.7 以上的值,操作系统和文件系统必须至少占用内存的30％,否则性能可能由于分页而严重恶化。

可以通过rabbitmqctl设置,例如,直接使用absolute,可以更加精确的控制使用内存.

    rabbitmqctl set_vm_memory_high_watermark 0.4
    rabbitmqctl set_vm_memory_high_watermark absolute 2G

配置文件配置,例如

    [{rabbit, [{vm_memory_high_watermark, {absolute, "1024MiB"}}]}].

还可以配合paging来使用,用来在到达告警值时,尝试换页来释放内存.例如,下面在最大内存0.75倍时,开始换页,到达最大内存后,依然阻塞.

    [{rabbit, [{vm_memory_high_watermark_paging_ratio, 0.75},{vm_memory_high_watermark, 0.4}]}].


对于`disk_free_limit`官方的建议:
- {disk_free_limit,{mem_relative,1.0}} 最小建议磁盘空间大小应等同内存大小.例如,在一台4g内存的rabbitmq主机上,如果磁盘空间大小4g,那么rabbitmq将不在接收新消息,直到队列被消费,磁盘空间有空闲为止.
- {disk_free_limit,{mem_relative,1.5}} 生产环境下比较安全的磁盘空间大小应该是内存的1.5倍.例如在4g内存的rabbitmq节点上,那么需要至少6g磁盘空间,如果向磁盘写入4g,并重启,那么重启后2g磁盘空间小于6g,rabbitmq将不在接收新消息.
- {disk_free_limit,{mem_relative,2.0}} 默认选择 最保守的设置是使用所有的磁盘空间,如果希望能使用所有磁盘空间,就使用该设置.

可以通过rabbitmqctl设置,例如
    //大于set_vm_memory_high_watermark
    rabbitmqctl set_disk_free_limit 2G
    rabbitmqctl set_disk_free_limit mem_relative 1.0

配置文件配置,例如

    [{rabbit, [{vm_memory_high_watermark, {absolute, "1024MiB"}}]}].

对于`file_descriptors`官方建议
最小50k.一般计算方式:文件描述符大小=并发连接数*95%*2+队列数.

3 安全相关
建议使用tls安全加密.要排除已有问的加密方式例如sslv3

4 使用自动重连
默认启用,自动重连和拓扑重连.自动重连不会重新创建队列等,拓扑重连则会重新创建队列等.

5 集群

- 建议集群的节点为奇数个,3,...
- 建议publisher和consumer尽可能的连接一个node,减少不必要的流量损失.
- 磁盘存储一半节点就足够了.

6 分区策略
如果没有特别需要,可以使用autoheal分区策略

7 时钟同步
rabbitmq节点之间不需要同步,但是一些插件可能依赖时钟.可以使用NTP来保证时钟同步.