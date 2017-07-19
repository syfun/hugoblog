+++
categories = ["云计算"]
date = "2015-06-24T22:05:33+08:00"
description = ""
title = "深入CloudFoundry一周年（转载）"
thumbnail = ""
tags = ["云计算"]

+++


本文转自[深入CloudFoundry一周年（原版）](http://ju.outofmemory.cn/entry/22259)，有些地方稍微做些修改，毕竟是几年前的文章了。


## 篇前语

本文在《程序员》杂志2013年1月刊刊登过，但由于篇幅排版等原因，部份内容被删除。研究院博客这次发表的版本是原稿版。可能文笔措辞有些许粗糙，但希望能给大家带来更详尽的信息。

## 前言

2012年4月份，VMware突然发布了业内第一个开源的PaaS，Cloud Foundry；紧接着5月，Redhat发布了另外一个开源的PaaS，OpenShift。从此PaaS不再神秘，开始成为技术圈内的热议话题。我们研究院在CloudFoundry发布时就开始投入研究，在实验室内部署了基于Cloud Foundry私有的PaaS。到了10月份，我们总结部份研究成果，整理成博客发表在研究院的轻微博上。

Cloud Foundry经过一年多的发展，文章内很多内容都已经不合时宜，这篇一周年特别版的结构重新整理，文章内容部份借鉴我的同事颜开所写的《新版CloudFoundry揭秘》以及一些同行的文章、演讲进行改写；加入部份笔者在OpenStack APAC会议上所作报告《ElasticArchitecture in Cloud Foundry and Deploy with OpenStack》内容，而CloudFoundry的安装配置、扩展运行时、自定义服务等内容，本专栏会有后续专文介绍，这里就不再重复。

<!--more-->

## 正文

### 一、什么是PaaS？

PaaS这个话题已经不再新鲜，但是与云计算的其他概念一样，让人感觉云里雾里的。在所有让人摸不着头脑的定义里面，笔者最喜欢下面这张EMC World 2012展示的图，它从使用角度介绍了PaaS的含义。

![](/img/iaas-paas.jpg)

从开发者的角度来看，如果我们开发了一个应用程序，需要部署。传统的IT需要为你准备网络、存储、服务器、虚拟化、操作系统、中间件、应用运行环境等一系列设施；而如果你的企业上了IaaS，例如OpenStack，那这些IaaS套件就会帮你把网络到操作系统全包办了，你可以避免走冗长的流程去申请机器，公网IP的这些活；但是这时还没完，因为IaaS只能帮你到给你若干台装好Linux或者Windows，但是你不能按需得到运行环境，也不可以在资源不够的时候把你的应用进行横向扩展，而PaaS就接管剩下的工作，为你提供中间件和运行环境。在企业上了PaaS后，开发者只需要管理自己的应用及应用数据即可，所有设施相关的都可以包办给PaaS平台。这里可以看到三点：

1、PaaS基于IaaS之上的；

2、PaaS为应用开发者提供中间件及运行环境；

3、PaaS管理应用的部署，扩展等生命周期。

其实上面第三点很重要，如果没有第三点，IaaS也可以通过一些技巧提供中间件和运行环境。譬如我有一个Ruby on Rails的应用，需要用到Mysql，那系统管理员完全可以把Ruby环境和Mysql集成到IaaS的模板里面。但是这样的设计无法满足下面的场景：我目前应用有10个节点，由于访问压力增加，应用资源不够了，现需要动态增加为20个节点，但我提供的服务不可以接受中断。

当然，一定要较真的话，结合一些DevOp的知识也可以做到，但开发这些DevOp的脚本可以认为是一个最小化的PaaS了。

### 二、PaaS的基本架构

为了抽象出，PaaS的架构，我们先来看看下图：

![](/img/cloudfoundry.png)
![](/img/heroku.jpg)
![](/img/openshift.png)

这是三个非常著名的PaaS平台的高层架构图，从上到下分别是：CloudFoundry，Heroku和OpenShift。每个个架构图，笔者都加了两条分隔线，我们可以看到它们拥有相似的三层架构：

a)  Router，主要负责找到相对硬的App访问点。在Heroku中叫做Routing Mesh，在OpenShift中叫做HA Proxy；

b)  App的运行容器，它必须是同构可互相替代的。在CloudFounry中叫作App Exec Engine，也就是下文所说的DEA，在Heroku中叫Dyna Grid，在OpenShift中就是上图画有”PHP”的gear；

c)  系统服务节点。

PaaS主要为了大规模访问而设计，所以要求每一层都支持Failover，也就是说任何一个部份都有多节点运行，任何一个节点都可以挂掉，但要求其他的同组件节点可以替代已挂节点的工作；每一个层都可以扩展，以满足大并发需求；资源非独占性，如果某一组件资源有富裕，可以回收资源供给其他组件使用。后面两点就是我们常说的弹性。后文我们将可以看到CloudFoundry是如何做到以上几点的。

### 三、CloudFoundry的架构及模块

从总体地看，CloudFoundry的架构如下：

![](/img/cloudfoundry2.jpg)

如果看过《深入CloudFoundry》原文，我们可以发现CloudFoundry的组件增加了很多。但是它的核心组件并没有变化，增加的组件可以认为是原架构基础上的细化和专门化。譬如Stager组件是为了解决打包（Stage）过程需要操作大量文件，操作时间长的问题。所以Stager模块作为独立进程，接受这些工作逐个运行，而不阻塞作为核心组件的CloudController。

CloudFoundry的核心组件有：

1、Router：顾名思义，Router组件在CloudFoundry中是对所有进来的Request进行路由。进入Router的request主要有两类：首先是来自VMC Client或者STS的，由CloudFoundry使用者发出的，管理型指令。例如：列出你所有apps的vmc apps，提交一个apps等等。这类request会被路由到App Life Management组件，又叫Cloud Controller组件去；第二类是外界对你所部署的apps访问的request。这部份requests会被路由到App execution，又或者叫做DEAs的组件去。所有进入CloudFoundry系统的requests都会经过Router组件，看到这里可能会有朋友会担心Router成为单点，从而成为整个云的瓶颈。但是CloudFoundry作为云系统，其设计的核心就是去单点依赖，组件平行扩充，且可替代的以保证扩展性，这是CloudFoundry，甚至所有云计算系统的设计原则，后文会讨论CloudFoundry如何做到这点，目前只要知道，系统可以部署多个Routers共同处理进来的requests，但是Router上层的Load Balance不在CloudFoundry的实现范围，CloudFoundry只保证所有的request是无状态的，这样就使上层均衡附载选择面非常非常大了，例如可以通过DNS做，也可以部署硬件的Load Balancer，或者简单点，弄台ngnix作负载均衡器，都是可行的。

在第一个版本中，Router作为一个nginx脚本存在。所以的请求都必须经过Ruby代码，然后加以转发。这个设计干净利落，不过Ruby也因此转发了大量的数据，容易引起性能问题，所以在新版本中做了如下的改进（左边为第一版本，右边为新版）：

![](/img/cloudfoundry3.jpg)

a)  使用Nginx的Lua扩展，在Lua中加入URL查询逻辑和统计逻辑；

b)  如果Lua不知道当前的URL应该对应到底下哪一个DEA，则会发一个Request到router_uls_server.rb（也就是上图的”Upstream LocatorSVC”）；

c)  router_uls_server.rb是一个简单的Sinatra应用，它存储了所有URL与DEA IP:Port对应数据。另外它也管理了访问Session数据（这个后面会说）。

这样一来，大量的业务请求在Lua查询过一次位置后，变成了nginx直连下属业务，而不再经过router。逻辑和数据完美分离。性能和稳定性都大幅提高了。

另外一个大改进在于在前版设计中，当Router接收到请求后，会随机分配一个Droplet来处理这个请求，这种方式使得用户没有办法使用Session，因为连续的HTTP请求会被分发到不同的应用实例上处理。新版本设计中增加了对SESSION的支持，当Router发现用户的请求中带了cookie信息，它会在Cookie里暗藏一个Droplet的host, port地址。当有新的请求进来，Router通过解析Cookie得到上次的应用实例，然后尽量转发到同一台Droplet上。而这部份信息与上面的查询类似，会先存在于router_uls_server.rb，当Lua知道后会保存在Nginx内部提高效率。

还有一点可以利用的是，router_uls_server.rb作为一个基于Sinatra的服务，并且知道PaaS内所有映射、会话关系，是我们二次开发一个很有用的切入点。

2、DEA (Droplet ExecutionAgent): 首先要解析下什么叫做Droplet。Droplet在CloudFoundry的概念里面是指一个把你提交的源代码，以及CloudFoundry配套好的运行环境，再加上一些管理脚本，例如Start/Stop这些小脚本全部压缩好在一起的tar包。还有一个概念，叫做Staging app，就是指制作上面描述这个包，然后把它存储好的过程。CloudFoundry会自动保存这个Droplet，直到你start一个app的时候，一台部署了DEA模块的服务器会来拿一个Droplet的copy去运行。所以如果你扩展你的app到10个instances，那这个Droplet就被会复制十份，让10个DEA服务器拿去运行。

下图是DEA模块的新架构图（左编为原版，右边为新版）：

![](/img/cloudfoundry4.jpg)

当CloudFoundry刚刚推出的时候， Droplet包含了应用运行时启动，停止等简单命令。用户应用可以随意访问文件系统，也可以在内网畅通无阻，跑满CPU，占尽内存，写满磁盘。你一切可以想到的破坏性操作都可以做到，太可怕了。CloudFoundry显然不会放任这样的情况太久，现在他们开发出了Warden，一个程序运行容器。这个容器提供了一个孤立的环境，Droplet只可以获得受限的CPU,内存，磁盘访问权限，网络权限，再没有办法搞破坏了。

Warden在Linux上的实现是将Linux 内核的资源分成若干个namespace加以区分，底层的机制是CGROUP。这样的设计比虚拟机性能好，启动快，也能够获得足够的安全性。在网络方面，每一个Warden实例有一个虚拟网络接口，每个接口有一个IP，而DEA内有一个子网，这些网络接口就连在这个子网上。安全可以通过iptables来保证。在磁盘方面，每个warden实例有一个自己的filesystem。这些filesystem使用aufs实现的。Aufs可以共享warden之间的只读内容，区分只写的内容，提高了磁盘空间的利用率。因为aufs只能在固定大小的文件上读写，所以磁盘也没有出现写满的可能性。

LXC是另一个LinuxContainer。那为什么不使用它，而开发了Warden呢。因为LXC的实现是和Linux绑死的，CloudFoundry希望warden能运转在各个不同的平台，而不只是Linux。另外Warden提供了一个Daemon和若干Api来操作，LXC提供的是系统工具。还有最重要的一点是LXC过于庞大，Warden只需要其中的一点点功能就可以了，更少的代码便于调试。

除了增加隔绝性外，DEA的基本运行原理并没有发生根本改变：Cloud Controller模块（下面会介绍）会发送start/stop等基本的apps管理请求给DEA，dea.rb接收这些请求，然后从NFS里面找到合适的Droplet。前面说到Droplet其实是一个带有运行脚本的，带运行环境的tar包，DEA只需要把它拿过来解压，并即行里面的start脚本，就可以让这个app跑起来。到此，app算是可以访问，并start起来了，换句话说就是有这台服务器的某一个端口已经在待命，只要有request从这个端口进来，这个app就可以接收并返回正确的信息。接着dea.rb要做些善后的工作：1、把这个信息告诉Router模块。我们前面说到，所有进入CloudFoundry的requests都是由Router模块处理并转发的，包括用户对app的访问request，一个app起来后，需要告诉router，让它根据load balance等原则，把合适的request转进来，使这个app的instance能够干起活；2、一些统计性的工作，例如要把这个用户又新部署了一个app告诉Cloud Controller，以作quota控制等；3、把运行信息告诉Health Manager模块，实时报告该app的instance运行情况。另外DEA还要负责部份对Droplet的查询工作，譬如，如果用户通过Cloud Controller想查询一个app的log信息，那DEA需要从该Droplet里面取到log返回等等。

3、CloudController:  Cloud Controller是CloudFoundry的管理模块。主要工作包括：

a)  对apps的增删改读；

b)  启动、停止应用程序（通过DEA）；

c)  Stagingapps（把apps打包成一个droplet，通过Stager）；

d)  修改应用程序运行环境，包括instance、mem等等；

e)  管理service，包括service与app的绑定等；

f)  Cloud环境的管理；

g)  修改Cloud的用户信息（通过UAA，ACM）；

h)  查看CloudFoundry，以及每一个app的log信息。

这似乎有点复杂，但简单的说，可以很简单：就是与VMC和STS交互的服务器端。VMC和STS与CloudFoundry通信采用的是restful接口，另一方面Cloud Controller是一个典型的Ruby on Rails项目，从VMC或者STS接到JSON格式的协议，然后写入Cloud ControllerDatabase，并发消息到各模快去控制管理整个云。

我们以部署一个App到CloudFoundry为例，在我们在敲完那条简单的push命令后，VMC开始工作，在做完一轮的用户鉴权、查看所部署的apps数量是否超过预定数额，问了一堆相关app的问题后，需要发4个指令：

a)  发一个POST到“apps”，创建一个app;

b)  发一个PUT到“apps/:name/application”，上传app;

c)  发一个GET到“apps/:name/”，取得app状态，看看是否已经启动；

d)  如果没有启动，发一个PUT到“apps/:name/”，使其启动。

第一版的CloudController是基于Ruby On Rails的，在新版中，为了与CloudFoundry其他模块一致，并且让架构更加简单，CloudController用Sinatra进行了重写，并且加入了更多的模块去细化CloudController的工作。

另外一个重要的改进是，第一个版本的Droplet是通过NFS共享的，但这样会带来安全、性能等问题，而新版中进行了两大改进：

a)  移除NFS，采用自己开发的，简单的blobstore来存放Droplet;

b)  为Ruby项目进行了优化，把常用的Gem保存在package cache里面。所以在打包Ruby项目的时候不需要到公网上下载Gem文件，而是从CloudFoundry内部的Cache获得，大大加速了Stage过程。

随着CloudFoundry的逐渐成熟，权限管理功能在新的版本进行了很大的加强，在原有的用户模型基础上，加入了组织、用户空间的概念。

![](/img/cloudfoundry5.jpg)

用户模型的认证是由UAA模块实现的，它可以与企业已有的认证系统进行整合，例如LDAP，CAS等；鉴权是由ACM模块实现的。

![](/img/cloudfoundry6.jpg)

上图示例了一个用户访问CloudController的API的过程。我们可以分别看到UAA与ACM模块在一套鉴权流程各自扮演的角色。

UAA与ACM需要展开的内容很多，碍于篇幅原因，这里就不展开介绍了，有兴趣的朋友可以参考：[UAA](/img/https://github.com/cloudfoundry/uaa/tree/master/docs)可能研究院后续会有专门的博客介绍相关技术。

4、Health Manager: 做的事情不复杂，简单的说是从各个DEA里面拿到运行信息，然后进行统计分析，报告等。统计数据会与CloudController的设定指标进行比对，并提供Alert等。Health Manager模块目前还不是十分完善，但是Cloud Manage栈里面，自动化health管理、分析是一个很重要的领域，而这方面可以扩展的地方也很多，结合Orchestration Engine可以使云自管理、自预警；而与BI方面技术结合，可以统计运营情况，合理分配资源等。这方面CloudFoundry还在发展之中。

5、Services: 服务从上文的PaaS三层模型来看属于第三层，Cloud Foundry把Service模块设计成一个独立的、插件式的模块，第三方可以方便把自己的服务整合成Cloud Foundry服务。在Github上与服务相关，主要关注两个子项目：

a) vcap-services-base：顾名思义，包括Cloud Foundry服务的框架及核心类库。如果开发自定义的服务，我们需要引用到里面的类；

b) vcap-services：目前Cloud Foundry支持的，包括官方及大部份第三方贡献的服务。这个项目的根文件目录是以服务名称分的，我们可以选择感兴趣的去分别研究。

由此可见，Service模块的设计十分方便第三方提供自定义服务。从架构来说， Cloud Foundry服务部份使用了模板方法设计模式，我们通过重写钩子方法来实现自己的服务，如果不需要特别逻辑可以使用默认方法。从客户输入VMC命令开始，一个完整的服务访问流程如下图：

![](/img/cloudfoundry7.jpg)

a)  开发人员通过VMC创建一个Service;

b)  VMC把其转为Restful接口到CloudController；

c)  CloudController通过Restful接口发送创建命令到Service Gateway；

d)  ServiceGateway通过NAT（下文会提到）发送provision命令道Service Node；

e)  ServiceNode创建Service；

f)  Service Node把返回创建的Service访问方法回Service Gateway；

g)  ServiceGateway返回访问方法到CloudController；

h)  CloudController把创建的Service访问放法注入到应用运行环境；

i)  最终用户可以直接访问其服务。

现实情况中，种种原因使有些系统服务难以，或者不愿意迁移到云端，为此Cloud Foundry 引入了Service Broker模块。

Service Broker可以使部署在Cloud Foundry上的应用可以访问本地的服务。ServiceBroker的使用方法如下：

![](/img/cloudfoundry8.gif)

a)  我们必须准备好系统,例如postgress。我们配置好程序和防火墙，让CloudFoundry能通过类似postgres://xyzhr:secret@db.xyzcorp.com:5432/xyz_hr_db 的URL来访问到服务。

b)  调用create service，系统会在ServiceBroker中记录你的配置信息。这样就算大功告成了。Bind和其他的过程都有ServiceBroker完成，其实仅仅就是记录信息，没有实际操作。使用这个新的Service的时候和使用CloudFoundry的内部Service没有两样，配置参数都会通过环境变量传入。所以当App访问Service的时候，就与ServiceBroker无关了。

对应上面的Service流程，Service Broker的访问流程如下，这里就不重复叙述了：

![](/img/cloudfoundry9.jpg)

6、NATS (Message bus)： 

CloudFoundry的架构是基于消息发布与订阅的，联系各模块的是一个叫nats的组件。NATS是由CloudFoundry的架构师Derek开发的一个基于事件驱动的轻量级支持发布、订阅机制的消息系统。它基于EventMachine开发，以事件驱动。第一版本CloudFoundry被人诟病的一个问题就是NATS服务器是单节点，性能方面从实际应用情况看，是让人满意的；但HA方面的问题难以避免，成为整套系统HA的软肋。但新版NATS终于支持多服务器节点，NATS服务器间通过THIN来做通信。NATS的Github开源地址是：[NATS](https://github.com/derekcollison/nats)。代码量不多但设计很精妙，推荐下载来慢慢研究。

CloudFoundry作为一个弹性设计的，多模块的分布式系统，且支持模块自发现，错误自检，保证模块间低耦合。其根本原理就是基于消息发布订阅机制构建。每台服务器上的每个模块会根据自己的消息类别，向Message Bus发布多个消息主题；而同时也向自己需要交互的模块，按照需要的信息内容的消息主题订阅消息。

如前所说，CloudFoundry的核心是一套消息系统，如果想了解CloudFoundry的来龙去脉，去跟踪它里面复杂的消息机制是非常好的方法。譬如：一个DEA被加入CloudFoundry集群中，它需要向大家吼一下，以表明它已经准备好服务了，它会发布一个主题是”dea.start”的消息：

```ruby
NATS.publish('dea.start',@hello_message_json)
```

@ hello_message_json中包括DEA的UUID, ip, port, 版本信息等内容。

再例如，Cloud Controller需要启动一个Droplet的instance：

a)  首先一个DEA在启动的时候，会首先会对自己UUID的消息主题进行订阅。

```ruby
NATS.subscribe("dea.#{uuid}.start"){ |msg| process_dea_start(msg) }
```

其他模块需要通过"dea.#{uuid}.start"这个主题发送消息来使它启动，一旦这个DEA接收到消息，就会触发process_dea_start(msg)这个方法来处理启动所需要的工作；

b)  Cloud Controller或者其他模块发送消息，让UUID为xxx的DEA启动。

```ruby
NATS.publish("dea.#{dea_id}.start",json)
```

c)  DEA模块接收到消息后，就会触发process_dea_start(msg)方法。msg是由其他模块发送过来的消息内容，包括：droplet_id, instance_index, service, runtime等内容，process_dea_start会取得这些启动DEA必须的信息，然后进行一系列操作，例如从NFS中取得Droplet，解压，修改必要环境配置，运行启动脚本等等。等一切都准备好后，然后需要给Router发个消息，告诉它这个Droplet已经随时准备好报效国家，以后有相应的request记得让它来处理。

```ruby
NATS.publish('router.register',{

:dea => VCAP::Component.uuid,

:host => @local_ip,

:port => instance[:port],

:uris => options[:uris] ||instance[:uris],

:tags => {:framework =>instance[:framework], :runtime => instance[:runtime]}

                        }.to_json)
```


d)  Router模块在启动时就已经订阅”router.register”消息主题，

```ruby
NATS.subscribe('router.register'){ |msg|

msg_hash = Yajl::Parser.parse(msg,:symbolize_keys => true)

return unless uris = msg_hash[:uris]

uris.each{|uri|register_droplet(uri,msg_hash[:host],msg_hash[:port],msg_hash[:tag 

s]) }

}
```

收到前面DEA发出的信息后，会触发register_droplet方法，去绑定Droplet。到此启动一个Droplet的instance工作完成。

CloudFoundry的架构简单介绍至此，其实作为第一款开源的PaaS，CloudFoundry架构有很多可以学习借鉴的地方，很多细节上的处理是很精妙的，本文题虽为深入CloudFoundry，其实也只是浅尝即止，如果真要做到深入恐怕上面提到的每个模块都需要立篇详谈。一年后的今天重新审视CloudFoundry的学习资料，已经很丰富，网上有很多牛人已经写了很多东西探讨CloudFoundry的架构，文章最后笔者会给大家推荐个人觉得比较好的一些中文资料。总结一下，笔者从CloudFoundry的结构中学到的东西：

1、基于消息的多组件架构是实现集群的简单、且有效方法。消息可以使集群节点间解耦，使自注册，自发现这些在大规模数据中心中很重要的功能得到实现；

2、适当的抽象层，模板模式的使用，方便第三方可以方便在CloudFoundry开发扩展功能。CloudFoundry在DEA及Service层都做了抽象层处理，相对应地使开发者可以容易地为CloudFoundry开发Runtime和Service。例如，在CloudFoundry刚推出的时候，只支持Node.js, Java, Ruby，但第三方提供商、开源社区快速跟进，为CloudFoundry添加了PHP, Python的支持。这得益于CloudFoundry精巧的DEA架构设计。

### 四、安装配置CloudFoundry

写《深入CloudFoundry》的时间比较早，那时候连dev_setup都没有，更不用说更为先进的、适合大规模部署CloudFoundry使用的BOSH。原文介绍的是如何通过理解CloudFoundry各模块的关系，通过手工配置安装CloudFoundry的，笔者认为那依然是让我们理解CloudFoundry工作原理的最好方法，而且过程很有乐趣，就如我们在习惯了使用自动相机“咔嚓”一下的时候，如果有机会把玩一些老相机，躲进暗房尝试自己冲洗银盐底片，那种感受是难以言表的。

CloudFoundry是一个PaaS，前面讨论到PaaS是基于IaaS之上的，上面CloudFoundry的部署指南是以vSphere作为IaaS。但是无论是CloudFoundry本身，还是部署工具BOSH的设计都是以IaaS不相关的。正如一年前那篇《深入CloudFoundry》所说，我们可以用OpenStack作为CloudFoundry的PaaS层。而CloudFoundry后来出现的部署系统BOSH也和我们想得不谋而合，它对IaaS层做了一层抽象，叫做CPI（Cloud Provider Interface）。

![](/img/cloudfoundry10.jpg)

而CPI里面就有专门为OpenStack准备的，叫做Piston，开源地址在：[Piston](https://github.com/piston/openstack-bosh-cpi)

如果从整体角度去理解BOSH ，其实可以发现这里面很简单。我们需要部署CloudFoundry，BOSH要做几件事情？

a)  向IaaS要一台虚拟机，导入image文件；

b)  向IaaS要存储空间，并attach到虚拟机上；

c)  配置这台虚拟机的网络，使它可以和其他CloudFoundry服务器相通；

d)  在这台虚拟机里面下载安装CloudFoundry的代码；

e)  配置CloudFoundry组件间的消息机制。

这里与IaaS相关的，就是a), b), c)三点，如果我们回到第一章什么是PaaS那张图来看，提供网络、存储、服务器，以及image中带有的OS就是IaaS所给我们提供的内容。而如果看过《深入CloudFoundry》原文说过，由于由于CloudFoundry是完全模块化设计的，基于消息机制的分布式系统，所以安装配置CloudFoundry就是把每个模块单独跑起来，然后配置其消息机制，也就是上面的d),e)两点。

我们回到BOSH，显而易见，CPI作为IaaS的facade，需要提供相对应的接口。任何一个IaaS，只需要实现以下接口即可以支持BOSH，并用来部署CloudFoundry：

a)  create_stemcell

b)  delete_stemcell

c)  create_vm

d)  delete_vm

e)  reboot_vm

f)  configure_network

g)  create_disk

h)  delete_disk

i)  attach_disk

j)  detach_disk

stemcell可以认为是我们常说的image，或者说是模板，。根据函数名，我们基本能知道每个接口的作用了吧？我们可以看到是和我们上面说的a), b), c)相对应的。

不少公司，都已经搭了自己的虚拟化平台，如这次笔者参加QCon，国内某著名电子商务公司内部就用LXC构建了一套虚拟化平台。如果他们再写个接口，暴露出上面10个接口，同样可以用来部署CloudFoundry。


## 总结

开篇的时候，笔者原本只想对原文进行适当的修改，没想到一年过去了，CloudFoundry发生了太多改变，全文基本都已经重写，可见这一年来CloudFoundry社区的活跃。这篇文章属于新旧参半，如篇前所述，更多的是希望能把CloudFoundry的原理讲明白，讲得简单，请不要把本文作为参考手册使用。

其实一年多来，CloudFoundry在国内已经很火，下面推荐一些自己觉得好，在写本文时参考过的资料：

1、[CloudFoundry中文技术文档](http://cndocs.cloudfoundry.com/getting-started.html) （该网站好像已经打不开了）。现在CloudFoundry的文档化做得相当不错，尤其是部署云平台部份，本文多次引用提及；

2、@柳烟堆雪 的[《以NATS为主线的CloudFoundry原理》](http://blog.csdn.net/resouer/article/details/8065795)。很好的一篇文章，可惜他发表时，本文已经写完，否则本文一定会有汲取不少养分；

3、[著名博客五四科学院的CloudFoundry代码解读系列](http://www.54chen.com/?s=cloud+foundry)；

4、EMC中国研究院研究员颜开的[《新版CloudFoundry揭秘》](http://www.chinacloud.cn/show.aspx?id=9966&cid=11)。因为是同事，本文大段大段地拷他的这篇文章；

5、Cloudfoundry.org博客。另外要推荐的是Cloudfoundry.org的博客，我们知道Cloudfoundry的博客分.COM上的与.ORG上的，.COM上的大多是偏商业及应用，而.ORG的博客才是个牛人的园地；

代码还是代码。不用多说，作为一个开源项目，有什么比代码更直观？！
