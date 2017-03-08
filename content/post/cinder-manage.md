+++
categories = ["云计算"]
date = "2015-11-05T18:51:10+08:00"
description = ""
title = "Openstack Cinder管理二三事(Liberty版本)"
thumbnail = ""
tags = []
draft = true
+++


最近跳槽换了家公司，又是继续搞Openstack（感觉要一条道走到黑了😂）。正好新的L版已经发布了，为了避免以后眼红新功能，所以打算直接用L版的。环境基本上搭好了，接下来就是整理一下cinder的功能，那么就先看看官方的管理手册吧，[原地址](http://docs.openstack.org/admin-guide-cloud/blockstorage.html)。我这里摘录一些自己感兴趣或者觉着重要的来说一下。

<!--more-->

## 多后端配置

多后端的配置种每个后端都有一个volume\_backend\_name。这个名字是可以相同的，这时候sheduler会去选择某个后端处理请求。

假如现在我们有三个卷组，cinder-volumes1、cinder-volumes2、cinder-volumes3，那么在cinder.conf中可以这么配置。

```ini
[DEFAULT]
enabled_backends = lvm1, lvm2, lvm3
default_backend = lvm1

[lvm1]
volume_group = cinder-volumes1
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name=LVM

[lvm2]
volume_group = cinder-volumes2
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name=LVM

[lvm3]
volume_group = cinder-volumes3
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name=LVM_B
```

在这个配置中，lvm1和lvm2有同样的后端名LVM。当云盘创建请求LVM后端去创建时，scheduler会用容量调度去选择合适的后端。

但是这么配置了，前端请求并不知道有哪些后端。这时候我们需要把后端和类型进行绑定。

```
cinder type-create lvm
cinder type-key lvm set volume_backend_name=LVM
```

然后我们在前端或者命令行创建时就能愉快的选择使用哪个后端了。

```
cinder create --volume-type lvm test
```



后面更新。
