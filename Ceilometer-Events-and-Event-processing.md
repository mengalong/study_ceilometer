# Events and Event Processing (事件和事件处理)
## Events vs Samples
除了Meters和Sample Data，Ceilometer同样可以处理事件。一个Sample代表了一个独立的数值点，通过一个Meter可以代表过去一段时间值的变化，一个Event代表了一个OpenStack服务在一个时间点发生了什么事情。这个可以包括非数值数据，比如是一个instance的状态或者网络地址。

通常情况下，通过Events我们可以知道OpenStack系统中的某个对象发生了什么变化，比如：改变一个instance的size或者创建一个镜像。

相对来说，Samples更轻量级，更快可以支持随便的处理。Events更重，信息量更大，也会持续收集。

## Event Structure （Event的结构）
为了帮助下游实现计费/统计等功能，不同的服务都定义了一系列的数据格式，Events通常包含如下信息：

event_type:

　　　用 . 分割起来的字符串，定义了事件类型，比如：compute.instace.resize.start


message_id:

　　　　UUID,代表了事件的ID

generated：

　　　　时间戳，代表了事件在原始位置的发生事件

traits：

　　　　一个扁平的kv对。包含了大量的事件详情。Traits可以使字符串、整形、浮点型、时间类型。

raw：

　　　　（可选）主要用于审计。完整的通知消息会被存储下来以便以后的评估

## Events from Notifications（来自于通知消息的事件)
Events主要是通过OpenStack中的notifications system产生。比如：nova，glance，neutron等等在系统中发生了一些需要通知的动作之后，会发出一个json格式的消息到消息队列。Ceilometer会从消息队列中消费并处理这些事件。

OpenStack中notifications的基本理念是产生所有需要的数据然后让消费者过滤掉他们不感兴趣的。为了让系统简单有效，这些通知信息都被Ceilometer当做Events来处理。通知信息(notification)的理念是：可以使复杂的JSON数据结构，被转换成一个可以识别的扁平的KV对。具体的转换方法是在配置文件中定义好的，这样可以保证notification中被定义的字段是在处理过程真实需要的，最后会被存储为Traits。

## Converting Notifications to Events(消息到事件的转换)
为了确保用户提取信息足够的简单，Notifications到Events的转换是通过配置文件来驱动的(具体是在ceilometer.conf中定义的)

这部分包含了怎样吧notification中的字段映射到Traits，以及扩展插件做一些可编程的转换。

每一个event_type(事件类型)都定义了notifications到events的映射。在对应的字段在notifaction中存在并且不为空的时候，就会增加一个Traits。

如果定义的字段不存在，那么会产生一个warning并在日志中记录下来，但是一个空的定义是可以接受的。默认情况下，任何相关字段没有被定义的notifications会被转换成一个最小集合的默认traits。这个功能也可以在ceilometer.conf 的drop_unmatched_notifications中来设置。如果这个被设置为True，那么没有在配置文件中定义的notifications就会被丢弃。

如下是在notification中存在相关字段时会被默认加入Traits的字段：

- service
- tenant_id
- request_id
- project_id
- user_id 

## Definitions file format(配置文件格式)
event的定义文件时yaml格式。包含了一个可以被映射event列表。定义的顺序是有含义的，这个list会被逆序扫描（最后定义的会是第一个命中的),以确定一个notifications event_type 对应的是哪个定义。

## Event Definitions（事件定义）
每一个事件定义都是一个包含了两个key的map（这两个key都是必须的):

event_type

　　　　这是一个event_type列表（或者字符串）.这些可以被应用于unix shell。

traits

　　　　这是一个mapping，key就是trait的name，value就是trait的定义。
## Trait Definitions（Trait 定义)
每一个Trait的定义都是一个包含了如下key的mapping：

type：（可选),代表了这个traits的数据类型(字符串)。合法的可以是：文本类型、int类型，float类型以及datatime类型。默认是text类型。

fields：

fields是在notification中定义的需要为trait提取的字段说明。这些说明可以被写在多个匹配的fields中，value会从对应存在的fields提取出来。

plugin：

（可选)，这是一个包含了如下两个key的mapping：

- name：（字符串类型)，表明了那个plugin被加载
- parameters：（可选)，keyword的mapping，通过初始化的时候传递给plugin

## Field Path Specifications（Field路径说明）
路径规范的定义在notification的JSON中，用来给trait提取value。path可以使用 . 分割的字符串（比如：payload.host)。中括号形式的也支持（比如：payload[host]).另外，如果对应的key中本身就包含了 '.' 号，那么就需要用单引号把整个key括起来，比如：payload.image_meta.'org.openstack_1_architecture'

## Example Definitions file (实例）


```
- event_type: compute.instance.*
  traits: &instance_traits
    user_id:
      fields: payload.user_id
    instance_id:
      fields: payload.instance_id
    host:
      fields: publisher_id
      plugin:
        name: split
        parameters:
          segment: 1
          max_split: 1
    service_name:
      fields: publisher_id
      plugin: split
    instance_type_id:
      type: int
      fields: payload.instance_type_id
    os_architecture:
      fields: payload.image_meta.'org.openstack__1__architecture'
    launched_at:
      type: datetime
      fields: payload.launched_at
    deleted_at:
      type: datetime
      fields: payload.deleted_at
- event_type:
    - compute.instance.exists
    - compute.instance.update
  traits:
    <<: *instance_traits
    audit_period_beginning:
      type: datetime
      fields: payload.audit_period_beginning
    audit_period_ending:
      type: datetime
      fields: payload.audit_period_ending
```
