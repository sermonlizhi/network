## 一、总体思路和目标

#### DNS的主要思路

- 分层的、基于域的命名机制

- 若干分布式的数据库完成域到IP的地址的转换

- 运行在UDP之上端口为53的应用服务

- 核心的Internet功能，但以应用层协议实现

  <!--在网络边缘处理复杂性-->

#### DNS主要目的

- 实现主机名-IP地址的转换(name/IP translate)
- 其他目的
  - 主机别名到规范名称的转换
  - 邮件服务器别名到邮件服务器正规名的转换
  - 负载均衡

#### DNS域名结构

- 一个层面命名设备会有很多重名

- DNS采用层次树状结构的命名方法

- Internet根被划分为几百个顶级域(top lever domains)

  - 通用的(generic)

    .com	.edu	.gov	.int	.mil	.net	.org	.firm……

  - 国家的

    .cn	.us	.nl	.jp

- 每个(子)域下面可划分为若干个子域(subdomains)

- 树叶是主机

#### DNS名字空间(全球共有13个根名字服务器)

- 域名

  - 从本域网上，直到树根

  - 中间使用”.“间隔不同的级别

    ustc.edu.cn/auto.ustc.edu.cn/www.auto.ustc.edu.cn

  - 域的域名：可以用于表示一个域

  - 主机的域名：一个域上的一个主机

- 域名的管理

  - 一个域管理其下的子域

    .jp 被划分为ac.jp	co.jp

    .cn 被划分为 edu.cn  com.cn

    创建一个新的域，必须征得它所属域的同意

  - 域与物理网络无关(逻辑意义上的)

    域遵从组织界限，而不是物理网络

    <!--一个域的主机可以不在一个网络-->

    <!--一个网络的主机不一定在一个域-->

    域的划分是逻辑的，而不是物理的

#### TLD(Top Lever Domain)服务器

顶级域服务器：负责顶级域名(如con,org,net,edr)和所有国家级的顶级域名(cn,us,uk,jp)

- Network solutions 公司维护com TLD服务i其
- Educause 公司维护edu TLD服务器

#### 区域名字服务器维护资源记录

- 资源记录(resource records)

  - 作用：维护 域名-IP地址(其他)的映射关系
  - 位置：Name Server的分布式数据库中

- 资源记录(RR)格式

  - Domain_name：域名
  - Ttl：time to live 生存时间(权威，缓冲记录)
  - Class类别：对于Internet，值为IN
  - Value值：可以是数字，域名或ASCⅡ码
  - Type类别：资源记录的类型

- DNS记录

  保存资源记录(RR)的分布式数据库

  RR格式：（name,value,type,ttl）

  - Type=A

    - Name为主机
    - Value为IP地址

  - TYPE=CNAME

    - Name为规范名字的别名

      www.ibm.com的规范名字为：servereast.backup2.ibm.com

    - value为规范名字

  - TYPE=NS

    - Name为域名(如foo.com)
    - Value为该域名的权威服务器(保存整个区域域名到IP地址的映射)的域名

  - TYPE=MX

    - Value为name对应的邮件服务器的名称

  TTL：生存时间，决定了资源记录应当从缓存中删除的时间

## 二、DNS工作原理

​	`域名系统由一系列的Name Server构成`

#### DNS大致工作流程

- 应用调用 解析器(resolver)

- 解析器作为客户 向Name Server发出查询报文(封装在UDP段中)

  上网的终端上线时需要以下信息(手动配置或DHCP协议自动配置)：

  IP地址、子网掩码、Default Gateway、Local Name Server

- Name Server返回响应报文(name/ip)

#### 本地名字服务器(Local Name Server)

理论可以配置任何一个名称服务器，但一般指定离的最近的Name Server

​	1、并不严格属于层次结构

​	2、每个ISP都有一个本地DNS服务器

​     	<!--也称为”默认名字服务器“-->

​	3、当一个主机发起一个DNS查询时，查询被送到其本地DNS服务器

​		<!--起着代理的作用，将查询转发到层次结构中-->

- 名字解析过程

  - 目标名称再Local Name Server中

    情况1：查询的名字在该区域内部

    情况2：缓存

    当本地名字服务器不能解析名字时，联系根名称服务器顺着根-TLD一直找到权威名字服务器

- 递归查询

  - 名字解析负担都放在当前联络的名字服务器
  - 问题：根服务器的负担太重
  - 解决方案：迭代查询(iterated queries)

- 迭代查询

  主机cis.poly.edu想知道主机gaia.cs.umass.edu的IP地址

  - 根(及各级域名)服务器返回的不是查询结果，而是下一个NS的地址

  - 最后由权威名字服务器给出解析结果

    ”当前联系的服务器给出可以联系的服务器的名字“

    ”我不知道这个名字，但可以向这个服务器请求“

#### DNS协议、报文

DNS协议：DNS查询和响应的报文格式相同

报文首部：

- 标识符(ID)：16位，查询和响应通过ID关联，使得NS可以同时查询多个
- Flags：查询/应答 、希望递归、递归可用、应答为权威

#### 缓存

- 一旦名字服务器获得了一个新的映射，就将该映射缓存起来

- 根服务器通常都在本地服务器中缓存着

  使得根服务器不用经常被访问

  目的：提高效率

  可能存在的问题：如果情况变化，缓存结果和权威资源记录不一致

  解决方案：TTL(默认2天)

#### 新增一个域

- 在上级域的名字服务器中增加两条记录，指向这个新增的子域的域名和域名服务器的地址
- 在新增子域的名字服务器上运行名字服务器，负责本域的名字解析：名字->IP地址

  例子：在com域中建立一个Networkutopia

1. 到注册登记机构注册域名Networkutopia
   - 需要向该机构提供权威DNS服务器(基本的和辅助的)的名字和IP地址
   - 登记机构在com TLD服务器中插入两条RR记录
2. 在networkutopia.com的权威服务器中确保有以下记录
   - 用于web服务器的www.networkutopia.com的类型为A的记录
   - 用于邮件服务器mail.networkutopia.com类型为MX的记录
