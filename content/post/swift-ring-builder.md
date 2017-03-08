+++
thumbnail = ""
tags = []
categories = ["云计算"]
date = "2014-05-11T19:23:24+08:00"
description = ""
title = "Opensatck swift-ring-builder源码分析"

+++


swift中关键的部分便是ring，那ring是如何创建的呢？

我们来看bin目录下的swift-ring-builder:
```python
import sys

from swift.cli.ringbuilder import main


if __name__ == "__main__":
    sys.exit(main())
```

<!--more-->

调用swift.cli.ringbuilder中的main函数:
```python
def main(arguments=None):
    global argv, backup_dir, builder, builder_file, ring_file
    if arguments:
        argv = arguments
    else:
        # from sys import argv as sys_argv
        argv = sys_argv

    # 参数只有一个时，也就是只是swift-ring-builder时，打印帮助信息
    if len(argv) < 2:
        print "swift-ring-builder %(MAJOR_VERSION)s.%(MINOR_VERSION)s\n" % \
              globals()
        print Commands.default.__doc__.strip()
        print
        # 去掉内置方法和default方法
        cmds = [c for c, f in Commands.__dict__.iteritems()
                if f.__doc__ and c[0] != '_' and c != 'default']
        cmds.sort()
        for cmd in cmds:
            # 打印方法的doc
            print Commands.__dict__[cmd].__doc__.strip()
            print
        print parse_search_value.__doc__.strip()
        print
        # wrap用来包装一段文本，返回值是文本每一行的列表
        for line in wrap(' '.join(cmds), 79, initial_indent='Quick list: ',
                         subsequent_indent='            '):
            print line
        print('Exit codes: 0 = operation successful\n'
              '            1 = operation completed with warnings\n'
              '            2 = error')
        exit(EXIT_SUCCESS)

    # parse_builder_ring_filename_args函数从第二个参数中解析出builder_file和ring_file
    builder_file, ring_file = parse_builder_ring_filename_args(argv)

    # 如果builder_file已存在，加载这个file
    if exists(builder_file):
        builder = RingBuilder.load(builder_file)
    elif len(argv) < 3 or argv[2] not in('create', 'write_builder'):
        print 'Ring Builder file does not exist: %s' % argv[1]
        exit(EXIT_ERROR)

    # 创建备份目录
    backup_dir = pathjoin(dirname(argv[1]), 'backups')
    try:
        mkdir(backup_dir)
    except OSError as err:
        if err.errno != EEXIST:
            raise

    if len(argv) == 2:
        command = "default"
    else:
        command = argv[2]
    if argv[0].endswith('-safe'):
        try:
            with lock_parent_directory(abspath(argv[1]), 15):
                Commands.__dict__.get(command, Commands.unknown.im_func)()
        except exceptions.LockTimeout:
            print "Ring/builder dir currently locked."
            exit(2)
    else:
        # 执行commands对应的方法
        Commands.__dict__.get(command, Commands.unknown.im_func)()
```
我们来看下Commands里有多少主要的方法：
create， default， search， list\_parts， add， set\_weight， set\_info， remove， rebalance， validate， write\_ring， write\_builder， set\_min\_part\_hours， set\_replicas

通过建环的过程来分析几个重要的函数。

    swift-ring-builder account.builder create 3 2 1

1.create 方法，生成buidler file:
```python
def create():
    """
	swift-ring-builder <builder_file> create <part_power> <replicas>
                                             <min_part_hours>
    Creates <builder_file> with 2^<part_power> partitions and <replicas>.
    <min_part_hours> is number of hours to restrict moving a partition more
    than once.
    """
    if len(argv) < 6:
        print Commands.create.__doc__.strip()
        exit(EXIT_ERROR)
	# 初始化RingBuilder，RingBuilder是用来建立Ring的类，__init__接收三个参数，
	# part_power, number of partitions = 2**part_power
    # replicas, 备份数
	# min_part_hours， minimum number of hours between partition changes
    builder = RingBuilder(int(argv[3]), float(argv[4]), int(argv[5]))
    backup_dir = pathjoin(dirname(argv[1]), 'backups')
    try:
		# 创建备份目录
        mkdir(backup_dir)
    except OSError as err:
        if err.errno != EEXIST:
            raise
    # 将builder存储在文件中
    builder.save(pathjoin(backup_dir, '%d.' % time() + basename(argv[1])))
    builder.save(argv[1])
    exit(EXIT_SUCCESS)
```
命令的第4,5,6参数分别是part_power, repicas, min_part_hours

    swift-ring-builder account.builder add r1z1-192.168.0.2:6002/sdd 3
	swift-ring-builder account.builder add r1z2-192.168.0.3:6002/sdd 1

2.add 方法，添加设备到builder file中：
```python
def add():
	"""
	swift-ring-builder <builder_file> add
    [r<region>]z<zone>-<ip>:<port>[R<r_ip>:<r_port>]/<device_name>_<meta>
     <weight>
    [[r<region>]z<zone>-<ip>:<port>[R<r_ip>:<r_port>]/<device_name>_<meta>
     <weight>] ...

    Where <r_ip> and <r_port> are replication ip and port.

	or

	swift-ring-builder <builder_file> add
    [--region <region>] --zone <zone> --ip <ip> --port <port>
    --replication-ip <r_ip> --replication-port <r_port>
    --device <device_name> --meta <meta> --weight <weight>

    Adds devices to the ring with the given information. No partitions will be
    assigned to the new device until after running 'rebalance'. This is so you
    can make multiple device changes and rebalance them all just once.
    """
    if len(argv) < 5 or len(argv) % 2 != 1:
        print Commands.add.__doc__.strip()
        exit(EXIT_ERROR)

	# _parse_add_values，从参数中解析出设备信息
    for new_dev in _parse_add_values(argv[3:]):
        for dev in builder.devs:
            if dev is None:
                continue
            if dev['ip'] == new_dev['ip'] and \
                    dev['port'] == new_dev['port'] and \
                    dev['device'] == new_dev['device']:
                print 'Device %d already uses %s:%d/%s.' % \
                      (dev['id'], dev['ip'], dev['port'], dev['device'])
                print "The on-disk ring builder is unchanged.\n"
                exit(EXIT_ERROR)
		# 添加新设备
        dev_id = builder.add_dev(new_dev)
        print('Device %s with %s weight got id %s' %
              (format_device(new_dev), new_dev['weight'], dev_id))
	# 存储builder
    builder.save(argv[1])
    exit(EXIT_SUCCESS)
```

两个设备的属性如下：
<table>
  <tr>
    <td></td>
    <td>dev0</td>
    <td>dev1</td>
  </tr>
  <tr>
    <td>id</td>
    <td>0</td>
    <td>1</td>
  </tr>
  <tr>
    <td>weight</td>
    <td>3</td>
    <td>1</td>
  </tr>
  <tr>
    <td>region</td>
    <td>1</td>
    <td>1</td>
  </tr>
  <tr>
    <td>zone</td>
    <td>1</td>
    <td>2</td>
  </tr>
  <tr>
    <td>ip</td>
    <td>192.168.0.2</td>
    <td>192.168.0.3</td>
  </tr>
  <tr>
    <td>port</td>
    <td>6002</td>
    <td>6002</td>
  </tr>
  <tr>
    <td>device</td>
    <td>sdd</td>
    <td>sdd</td>
  </tr>
  <tr>
    <td>parts_wanted</td>
    <td>12</td>
    <td>4</td>
  </tr>
</table>

    swift-ring-builder account.builder rebalance

3.rebalance 方法，重新平衡虚拟节点，分配虚拟节点到设备
```python
def rebalance():
    """
	swift-ring-builder <builder_file> rebalance <seed>
    Attempts to rebalance the ring by reassigning partitions that haven't been
    recently reassigned.
    """
    def get_seed(index):
        try:
            return argv[index]
        except IndexError:
            pass
	# 设备是否有所改变
    devs_changed = builder.devs_changed
    try:
		# 获取旧的平衡
        last_balance = builder.get_balance()
		# 重新平衡
        parts, balance = builder.rebalance(seed=get_seed(3))
    except exceptions.RingBuilderError as e:
        print '-' * 79
        print("An error has occurred during ring validation. Common\n"
              "causes of failure are rings that are empty or do not\n"
              "have enough devices to accommodate the replica count.\n"
              "Original exception message:\n %s" % e.message
              )
        print '-' * 79
        exit(EXIT_ERROR)
    if not parts:
        print 'No partitions could be reassigned.'
        print 'Either none need to be or none can be due to ' \
              'min_part_hours [%s].' % builder.min_part_hours
        exit(EXIT_WARNING)
    # If we set device's weight to zero, currently balance will be set
    # special value(MAX_BALANCE) until zero weighted device return all
    # its partitions. So we cannot check balance has changed.
    # Thus we need to check balance or last_balance is special value.
    if not devs_changed and abs(last_balance - balance) < 1 and \
            not (last_balance == MAX_BALANCE and balance == MAX_BALANCE):
        print 'Cowardly refusing to save rebalance as it did not change ' \
              'at least 1%.'
    	exit(EXIT_WARNING)
    try:
        builder.validate()
    except exceptions.RingValidationError as e:
        print '-' * 79
        print("An error has occurred during ring validation. Common\n"
              "causes of failure are rings that are empty or do not\n"
              "have enough devices to accommodate the replica count.\n"
              "Original exception message:\n %s" % e.message
              )
        print '-' * 79
        exit(EXIT_ERROR)
    print 'Reassigned %d (%.02f%%) partitions. Balance is now %.02f.' % \
          (parts, 100.0 * parts / builder.parts, balance)
    status = EXIT_SUCCESS
    if balance > 5:
        print '-' * 79
        print 'NOTE: Balance of %.02f indicates you should push this ' % \
              balance
        print '      ring, wait at least %d hours, and rebalance/repush.' \
              % builder.min_part_hours
        print '-' * 79
        status = EXIT_WARNING
    ts = time()
    # 获取Ring，并保存备份
    builder.get_ring().save(
        pathjoin(backup_dir, '%d.' % ts + basename(ring_file)))
    builder.save(pathjoin(backup_dir, '%d.' % ts + basename(argv[1])))
	# 将Ring保存到ring file中
    builder.get_ring().save(ring_file)
    builder.save(argv[1])
    exit(status)
```
来看RingBuilder的rebalance方法：
```python
 def rebalance(self, seed=None):
    """
    Rebalance the ring.

    This is the main work function of the builder, as it will assign and
    reassign partitions to devices in the ring based on weights, distinct
    zones, recent reassignments, etc.

    The process doesn't always perfectly assign partitions (that'd take a
    lot more analysis and therefore a lot more time -- I had code that did
    that before). Because of this, it keeps rebalancing until the device
    skew (number of partitions a device wants compared to what it has) gets
    below 1% or doesn't change by more than 1% (only happens with ring that
    can't be balanced no matter what -- like with 3 zones of differing
    weights with replicas set to 3).

    :returns: (number_of_partitions_altered, resulting_balance)
    """

    if seed is not None:
        random.seed(seed)

    self._ring = None
	# _last_part_moves_epoch表示上一次移动part的时间， 为None表示首次平衡
    if self._last_part_moves_epoch is None:
        self._initial_balance()
        self.devs_changed = False
        return self.parts, self.get_balance()
    retval = 0
	# 更新上次移动的信息，包括移动的part和移动的时间
    self._update_last_part_moves()
    last_balance = 0
	# 调整part的数目，返回调整后的part和被移除的part数目
    new_parts, removed_part_count = self._adjust_replica2part2dev_size()
    retval += removed_part_count
	# 重新分配part
    self._reassign_parts(new_parts)
    retval += len(new_parts)
    while True:
		# 分配被移除设备上的part
        reassign_parts = self._gather_reassign_parts()
        self._reassign_parts(reassign_parts)
        retval += len(reassign_parts)
        while self._remove_devs:
            self.devs[self._remove_devs.pop()['id']] = None
        balance = self.get_balance()
		# 如果balance小于1或者所有的part都都分配了。退出循环
        if balance < 1 or abs(last_balance - balance) < 1 or \
                retval == self.parts:
            break
        last_balance = balance
    self.devs_changed = False
    self.version += 1
    return retval, balance
```

建环主要就是create，add，rebalance三步。其他的如search，list_parts，remove等都比较简单，这里不做分析。
