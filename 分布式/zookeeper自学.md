### zookeeper安装

安装环境：奇数台机器，本次选取3台机器，系统为centos；
安装步骤：

1. 安装jdk：略；sql：略
2. 下载zookeeper-3.4.5.tar：因为自己有现成的，所以没有去网上下载
3. 将zookeeper-3.4.5.tar传到某一台机器中
   [root@Master binto]# rz
4. 解压
   root@Master zookeeper]# tar -zxvf zookeeper-3.4.5.tar.gz
5. 重命名
   mv zookeeper-3.4.5 zookeeper（重命名文件夹zookeeper-3.4.5为zookeeper,注意更改后我的路径为/home/binto/zookeeper）

6. 修改环境变量

   - 切换用户到root：su

   - 修改文件：vi /etc/profile

     添加内容：

     ```
     export ZOOKEEPER_HOME=/home/binto/zookeeper
     
     export PATH=$PATH:$ZOOKEEPER_HOME/bin
     ```

   - 重新编译文件：source /etc/profile

   - 注意：3台zookeeper都需要修改

   - 修改完成后切换回hadoop用户：su - hadoop

7. 修改配置文件

   - 用hadoop用户操作
     cd zookeeper/conf
     cp zoo_sample.cfg zoo.cfg
   - vi zoo.cfg，然后添加内容：
     dataDir=/home/binto/zookeeper/datadataLogDir=/home/binto/zookeeper/logserver.1=Master:2888:3888 (主机名, 心跳端口、数据端口)server.2=Slave1:2888:3888server.3=Slave2:2888:3888
   - 创建文件夹：
     /home/binto/zookeeper/
     mkdir data
     mkdir log
   - 在data文件夹下新建myid文件，myid的文件内容为：1
     cd ../data
     vi myid

8. 将集群下发到其他机器上
   scp -r /home/binto/zookeeper root@Slave2:/home/binto/

9. 修改其他机器的配置文件
   到slave1上：修改myid为：2
   到slave2上：修改myid为：3

10. 启动（每台机器）
    zkServer.sh start

11. 查看集群状态

    - jps（查看进程）
    - zkServer.sh status（查看集群状态，主从信息）



### zookeeper数据结构、常用命令

见“**ZooKeeper数据模型和常见命令.md**”

https://github.com/Snailclimb/JavaGuide/blob/Snailclimb-patch-1/%E4%B8%BB%E6%B5%81%E6%A1%86%E6%9E%B6/ZooKeeper%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B%E5%92%8C%E5%B8%B8%E8%A7%81%E5%91%BD%E4%BB%A4.md

### **zookeeper-java**

**1.**  **基本使用**

 org.apache.zookeeper.Zookeeper是客户端入口主类，负责建立与server的会话

它提供了表 1 所示几类主要方法  ：

| 功能         | 描述                                |
| ------------ | ----------------------------------- |
| create       | 在本地目录树中创建一个节点          |
| delete       | 删除一个节点                        |
| exists       | 测试本地是否存在目标节点            |
| get/set data | 从目标节点上读取 / 写数据           |
| get/set ACL  | 获取 / 设置目标节点访问控制列表信息 |
| get children | 检索一个子节点上的列表              |
| sync         | 等待要被传送的数据                  |

**2.**  **简单实例**

```java
public class SimpleZkClient {

	private static final String connectString = "mini1:2181,mini2:2181,mini3:2181";
	private static final int sessionTimeout = 2000;

	ZooKeeper zkClient = null;

	@Before
	public void init() throws Exception {
		zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
			@Override
			public void process(WatchedEvent event) {
				// 收到事件通知后的回调函数（应该是我们自己的事件处理逻辑）
				System.out.println(event.getType() + "---" + event.getPath());
				try {
					zkClient.getChildren("/", true);
				} catch (Exception e) {
				}
			}
		});

	}

	/**
	 * 数据的增删改查
	 * 
	 * @throws InterruptedException
	 * @throws KeeperException
	 */

	// 创建数据节点到zk中
	public void testCreate() throws KeeperException, InterruptedException {
		// 参数1：要创建的节点的路径 参数2：节点大数据 参数3：节点的权限 参数4：节点的类型
		String nodeCreated = zkClient.create("/eclipse", "hellozk".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
		//上传的数据可以是任何类型，但都要转成byte[]
	}

	//判断znode是否存在
	@Test	
	public void testExist() throws Exception{
		Stat stat = zkClient.exists("/eclipse", false);
		System.out.println(stat==null?"not exist":"exist");
		
		
	}
	
	// 获取子节点
	@Test
	public void getChildren() throws Exception {
		List<String> children = zkClient.getChildren("/", true);
		for (String child : children) {
			System.out.println(child);
		}
		Thread.sleep(Long.MAX_VALUE);
	}

	//获取znode的数据
	@Test
	public void getData() throws Exception {
		
		byte[] data = zkClient.getData("/eclipse", false, null);
		System.out.println(new String(data));
		
	}
	
	//删除znode
	@Test
	public void deleteZnode() throws Exception {
		
		//参数2：指定要删除的版本，-1表示删除所有版本
		zkClient.delete("/eclipse", -1);
		
		
	}
	//删除znode
	@Test
	public void setData() throws Exception {
		
		zkClient.setData("/app1", "imissyou angelababy".getBytes(), -1);
		
		byte[] data = zkClient.getData("/app1", false, null);
		System.out.println(new String(data));
		
	}
	
	
}
```

**3.**  **实例：服务器上下线时进行动态感知**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190517172418996.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
server端：

```java
public class DistributedServer {
	private static final String connectString = "mini1:2181,mini2:2181,mini3:2181";
	private static final int sessionTimeout = 2000;
	private static final String parentNode = "/servers";

	private ZooKeeper zk = null;

	/**
	 * 创建到zk的客户端连接
	 * 
	 * @throws Exception
	 */
	public void getConnect() throws Exception {

		zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
			@Override
			public void process(WatchedEvent event) {
				// 收到事件通知后的回调函数（应该是我们自己的事件处理逻辑）
				System.out.println(event.getType() + "---" + event.getPath());
				try {
					zk.getChildren("/", true);
				} catch (Exception e) {
				}
			}
		});

	}

	/**
	 * 向zk集群注册服务器信息
	 * 
	 * @param hostname
	 * @throws Exception
	 */
	public void registerServer(String hostname) throws Exception {

		String create = zk.create(parentNode + "/server", hostname.getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
		System.out.println(hostname + "is online.." + create);

	}

	/**
	 * 业务功能
	 * 
	 * @throws InterruptedException
	 */
	public void handleBussiness(String hostname) throws InterruptedException {
		System.out.println(hostname + "start working.....");
		Thread.sleep(Long.MAX_VALUE);
	}

	public static void main(String[] args) throws Exception {

		// 获取zk连接
		DistributedServer server = new DistributedServer();
		server.getConnect();

		// 利用zk连接注册服务器信息
		server.registerServer(args[0]);

		// 启动业务功能
		server.handleBussiness(args[0]);

	}

}
```

client端：

```java
public class DistributedClient {

	private static final String connectString = "mini1:2181,mini2:2181,mini3:2181";
	private static final int sessionTimeout = 2000;
	private static final String parentNode = "/servers";
	// 注意:加volatile的意义何在？
	private volatile List<String> serverList;
	private ZooKeeper zk = null;

	/**
	 * 创建到zk的客户端连接
	 * 
	 * @throws Exception
	 */
	public void getConnect() throws Exception {

		zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
			@Override
			public void process(WatchedEvent event) {
				// 收到事件通知后的回调函数（应该是我们自己的事件处理逻辑）
				try {
					//重新更新服务器列表，并且注册了监听
					getServerList();

				} catch (Exception e) {
				}
			}
		});

	}

	/**
	 * 获取服务器信息列表
	 * 
	 * @throws Exception
	 */
	public void getServerList() throws Exception {

		// 获取服务器子节点信息，并且对父节点进行监听
		List<String> children = zk.getChildren(parentNode, true);

		// 先创建一个局部的list来存服务器信息
		List<String> servers = new ArrayList<String>();
		for (String child : children) {
			// child只是子节点的节点名
			byte[] data = zk.getData(parentNode + "/" + child, false, null);
			servers.add(new String(data));
		}
		// 把servers赋值给成员变量serverList，已提供给各业务线程使用
		serverList = servers;
		
		//打印服务器列表
		System.out.println(serverList);
		
	}

	/**
	 * 业务功能
	 * 
	 * @throws InterruptedException
	 */
	public void handleBussiness() throws InterruptedException {
		System.out.println("client start working.....");
		Thread.sleep(Long.MAX_VALUE);
	}
	
	
	
	
	public static void main(String[] args) throws Exception {

		// 获取zk连接
		DistributedClient client = new DistributedClient();
		client.getConnect();
		// 获取servers的子节点信息（并监听），从中获取服务器信息列表
		client.getServerList();

		// 业务线程启动
		client.handleBussiness();
		
	}

}
```
