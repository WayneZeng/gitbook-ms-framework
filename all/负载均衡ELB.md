### 话不多说前言
负载均衡 简单来说就像分班考试，部分同学去重点班A，部分去奥数班B，部分去少年班C，作者本人去的是普通基础

### 书中自有黄金屋

- 客官,你看完了前言,如此优秀
- 【支付宝扫码】奖励自己一下，后面还有。

![image](https://github.com/WayneZeng/gitbook-ms-framework/blob/master/asset/common/alipay_redpacket.jpeg?raw=true)


### ELB
Elastic Load Balancing（ELB）以亚马逊提供的ELB为例，
主要提供三种负载均衡器：应用负载均衡（ALB），网络负载均衡，经典负载均衡。

### 几种负载均衡器比较
* 经典负载均衡器
  - 请求级别和连接级别
  - 可在多个 Amazon EC2 实例之间提供基本的负载均衡。Classic Load Balancer 适用于在 EC2-Classic 网络内构建的应用程序
  
* 网络负载均衡
  - 网络负载均衡器运行于连接级别（第 4 层），
  - 可将流量路由至 Amazon Virtual Private Cloud (Amazon VPC) 内的不同目标，每秒能够处理数百万请求，同时能保持超低延迟。
  - 网络负载均衡器还针对处理突发和不稳定的流量模式进行了优化。
  
* 应用负载均衡器件
  - 最适合 HTTP 和 HTTPS 流量的负载均衡，
  - 面向交付包括微服务和容器在内的现代应用程序架构，提供高级请求路由功能。

### 负载均衡算法和实现
* 轮询法
  - 所有请求按照顺序分发，不常用，比如分班，按报名顺序分班
  
```java
// 加锁，保证线程安全
  public String robin() {
    synchronized (serverIndex) {
      if (serverIndex >= serverList.size()) {
        serverIndex = -1;
      }
      return serverList.get(++serverIndex);
    }
  }
```

* 加权轮询
  - 轮询基础上加权重，好一点的机器权重大，也不常用了，以分班为例，教师大的分过去学生多
  
* 随机／加权随机
  - 随机分配，次数多的时候接近于轮询，以分班为例，就是抽签分班。
  
```java
public String random() {
    int randomIndex = new Random().nextInt(serverList.size());
    return serverList.get(randomIndex);
  }
```

* 最小连接 
  - 那个服务器连接少，就分流到那个机器，在机器性能不是均等的时候，这个方法容易把请求分到性能很差的机器上，也不好
  - 以分班为例，就是报名人数少的班级，优先调剂。  
  
* 源头地址hash
  - 教材案例
  
```java
  public String hash(String remoteIp) {
    Integer hashCode = remoteIp.hashCode();
    serverIndex = hashCode % serverList.size();
    return serverList.get(serverIndex);
  }

```
  
* 一致性hash 
  - 便于扩展不影响大部分原有节点
  - 为了均衡加上了虚拟节点
```java
public class ConsistHash {

  private String[] servers;

  /**
   * 真实结点列表
   */
  private static List<String> realNodes = new LinkedList<>();

  /**
   * 虚拟节点，key表示虚拟节点的hash值，value表示虚拟节点的名称
   */
  private static SortedMap<Integer, String> virtualNodes =
      new TreeMap<>();

  /**
   * 虚拟节点数目
   */
  private static final int VIRTUAL_NODES = 150;

  public ConsistHash(String[] servers) {
    this.servers = servers;
  }

  private void getNode() {
    // 原始的服务器添加到真实结点列表中
    Collections.addAll(realNodes, servers);

    // 添加虚拟节点
    for (String str : realNodes) {
      for (int i = 0; i < VIRTUAL_NODES; i++) {
        String virtualNodeName = str + "&&VN" + String.valueOf(i);
        int hash = getHash(virtualNodeName);
        virtualNodes.put(hash, virtualNodeName);
      }
    }
  }

  /**
   * 使用FNV1_32_HASH算法计算服务器的Hash值,这里不使用重写hashCode的方法，最终效果没区别
   */
  private static int getHash(String str) {
    final int p = 16777619;
    int hash = (int) 2166136261L;
    for (int i = 0; i < str.length(); i++) {
      hash = (hash ^ str.charAt(i)) * p;
    }
    hash += hash << 13;
    hash ^= hash >> 7;
    hash += hash << 3;
    hash ^= hash >> 17;
    hash += hash << 5;
    
    if (hash < 0) {
      hash = Math.abs(hash);
    }
    return hash;
  }

  /**
   * 得到应当路由到的结点
   */
  public static String getServer(String node) {
    // 得到带路由的结点的Hash值
    int hash = getHash(node);
    // 得到大于该Hash值的所有Map
    SortedMap<Integer, String> subMap =
        virtualNodes.tailMap(hash);
    // 第一个Key就是顺时针过去离node最近的那个结点
    Integer i = subMap.firstKey();
    // 返回对应的虚拟节点名称，这里字符串稍微截取一下
    String virtualNode = subMap.get(i);
    return virtualNode.substring(0, virtualNode.indexOf("&&"));
  }
}
```

### 负载均衡系统设计需要考虑的其他因素
* 实例健康检查（这个通常配合注册中心实现）
* 机器自动扩展（用一致性负载均衡在扩容时影响机器少）
* 监控服务，根据cpu使用量大小和流量大小，处理时长自动完成添加或者缩减实例服务
* 应用负载均衡可以按照功能分发到对应的集群，比如登录给登录集群，支付给支付集群


这一篇就到这里了，喜欢请打赏5毛买包辣条

![客观常来啊](./../asset/image/alipay.jpeg)