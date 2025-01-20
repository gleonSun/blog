---
layout: title
title: HBase 启停流程
date: 2021-03-18 16:43:03
tags: HBase
categories: [分布式系统, 分布式存储, HBase]
---

# 整体流程分析

版本：hbase-2.2.4
说明：分析展现的源码和脚本中会省略一部分，只保留与分析相关联的，感兴趣的可自行查阅。

## 启动

### start-hbase.sh

启动 HBase 的入口，有两种模式：单机模式和集群模式，何种模式取决于用户的配置，下文会详细说明。

```shell
# 获取当前路径，即 {HBASE_HOME}/bin
bin=`dirname "${BASH_SOURCE-$0}"`
bin=`cd "$bin">/dev/null; pwd`

# 加载 bin 目录下的 hbase-config.sh 文件
. "$bin"/hbase-config.sh

# 判断加载 hbase-config.sh 是否成功，失败则退出，通常最后命令的退出状态为 0 表示没有错误
errCode=$?
if [ $errCode -ne 0 ]
then
  exit $errCode
fi

# 此处用户一般不传参，所以默认将 start 赋值给 commandToRun
if [ "$1" = "autostart" ]
then
  commandToRun="--autostart-window-size ${AUTOSTART_WINDOW_SIZE} --autostart-window-retry-limit ${AUTOSTART_WINDOW_RETRY_LIMIT} autostart"
else
  commandToRun="start"
fi

# 通过 HBase 源码中的 HBaseConfTool 获取 conf/hbase-site.xml 中参数 hbase.cluster.distributed 的配置值，表示是否为集群模式，接下文附 1
distMode=`$bin/hbase --config "$HBASE_CONF_DIR" org.apache.hadoop.hbase.util.HBaseConfTool hbase.cluster.distributed | head -n 1`

# 当 distMode 为 false 时，启动单机测试版，此时 HMaster 和 HRegionServer 以及内嵌的 MiniZooKeeperCluster 均在同一个 JVM 里启动，接下文附 2
if [ "$distMode" == 'false' ]
then
  "$bin"/hbase-daemon.sh --config "${HBASE_CONF_DIR}" $commandToRun master
# 当该值为 true 时，启动 HBase 集群，接下文附 2。分别启动 Zookeeper、HMaster 和 HRegionServer，其中 Zookeeper 的启动情况分两种，一种是 HBase 管理的，一种是独立部署的，取决于是否在 hbase-env.sh 中配置 HBASE_MANAGES_ZK 参数，为 true 时由 HBase 管理。
else
  "$bin"/hbase-daemons.sh --config "${HBASE_CONF_DIR}" $commandToRun zookeeper
  "$bin"/hbase-daemon.sh --config "${HBASE_CONF_DIR}" $commandToRun master
  "$bin"/hbase-daemons.sh --config "${HBASE_CONF_DIR}" \
    --hosts "${HBASE_REGIONSERVERS}" $commandToRun regionserver
  "$bin"/hbase-daemons.sh --config "${HBASE_CONF_DIR}" \
    --hosts "${HBASE_BACKUP_MASTERS}" $commandToRun master-backup
fi
```

#### 附 1

调用 HBaseConfTool 及相关的 HBaseConfiguration 源码部分，可以清楚地看到读取了配置文件 hbase-default.xml 和 hbase-site.xml，通过脚本传入的 key 来获取相应的 value

```java
public class HBaseConfTool {
    public static void main(String args[]) {
        if (args.length < 1) {
          System.err.println("Usage: HBaseConfTool <CONFIGURATION_KEY>");
          System.exit(1);
          return;
        }
    
        Configuration conf = HBaseConfiguration.create();
        System.out.println(conf.get(args[0]));
      }
}

---

public class HBaseConfiguration extends Configuration {
  
    public static Configuration create() {
        conf.setClassLoader(HBaseConfiguration.class.getClassLoader());
        return addHbaseResources(conf);
      }

    public static Configuration addHbaseResources(Configuration conf) {
        conf.addResource("hbase-default.xml");
        conf.addResource("hbase-site.xml");
    
        checkDefaultsVersion(conf);
        return conf;
      }
}
```

#### 附 2

由 HMaster 接受脚本传入的参数，调用 ServerCommandLine 中的 doMain 方法解析后再通过 HMasterCommandLine 进行启动，单机版和集群版仅仅是在 HMasterCommandLine 中的 run 方法中判断后走了不同的逻辑。

```java
public class HMaster extends HRegionServer implements MasterServices {
    
    public static void main(String [] args) {
        LOG.info("STARTING service " + HMaster.class.getSimpleName());
        VersionInfo.logVersion();
        new HMasterCommandLine(HMaster.class).doMain(args);
      }
}

---

public abstract class ServerCommandLine extends Configured implements Tool {

    public void doMain(String args[]) {
        try {
          // 加载 HBase 的配置文件，并调用 ToolRunner 类
          int ret = ToolRunner.run(HBaseConfiguration.create(), this, args);
          if (ret != 0) {
            System.exit(ret);
          }
        } catch (Exception e) {
          LOG.error("Failed to run", e);
          System.exit(-1);
        }
      }
}

---

public class ToolRunner {
    public static int run(Configuration conf, Tool tool, String[] args) throws Exception {
        ……        
        // 调用 HMasterCommandLine 的 run 方法
        return tool.run(toolArgs);
    }
}

---

public class HMasterCommandLine extends ServerCommandLine {
    
    public int run(String args[]) throws Exception {
        // 添加默认参数
        ……
    
        CommandLine cmd;
        try {
          // 解析参数，失败则最终会调用 HMasterCommandLine 的 getUsage 方法返回操作指示，此处将 start 作为 args 加入到 cmd 中
          cmd = new GnuParser().parse(opt, args);
        } catch (ParseException e) {
          LOG.error("Could not parse: ", e);
          usage(null);
          return 1;
        }
    
        // 配置参数
        ……
                
        // 最终解析完成的剩下的参数，此处为 start
        @SuppressWarnings("unchecked")
        List<String> remainingArgs = cmd.getArgList();
        if (remainingArgs.size() != 1) {
          usage(null);
          return 1;
        }
    
        String command = remainingArgs.get(0);
        
        // 根据接收到的 command 调用相应方法
        if ("start".equals(command)) {
          return startMaster();
        } else if ("stop".equals(command)) {
          return stopMaster();
        } else if ("clear".equals(command)) {
          return (ZNodeClearer.clear(getConf()) ? 0 : 1);
        } else {
          usage("Invalid command: " + command);
          return 1;
        }
      }

    private int startMaster() {
        // 获取配置参数
        Configuration conf = getConf();
        // TraceUtil 是个包装类，以一种简化的方式提供了访问 htrace 4+ 的函数，Apache HTrace 是 Cloudera 开源出来的一个分布式系统跟踪框架，支持HDFS和HBase等系统，为应用提供请求跟踪和性能分析
        TraceUtil.initTracer(conf);
    
        try {
          // 这里从配置文件中识别出当前是单机模式还是集群模式，单机模式下指的是 LocalHBaseCluster 实例，会在同一个 JVM 里启动 Master 和 RegionServer
          if (LocalHBaseCluster.isLocal(conf)) {
            DefaultMetricsSystem.setMiniClusterMode(true);
            // 单机模式下启动 MiniZooKeeperCluster 作为 Zookeeper 服务，该类中的许多代码都是从 Zookeeper 的测试代码中剥离出来的
            final MiniZooKeeperCluster zooKeeperCluster = new MiniZooKeeperCluster(conf);
            // 从配置文件获取 hbase.zookeeper.property.dataDir 配置的参数作为 Zookeeper 数据目录
            File zkDataPath = new File(conf.get(HConstants.ZOOKEEPER_DATA_DIR));
    
            // find out the default client port
            int zkClientPort = 0;
    
            // 从 hbase.zookeeper.quorum 参数解析并获取 Zookeeper 配置的端口号
            String zkserver = conf.get(HConstants.ZOOKEEPER_QUORUM);
            if (zkserver != null) {
              String[] zkservers = zkserver.split(",");
              // 单机模式仅支持一个 Zookeeper 服务
              if (zkservers.length > 1) {
                // In local mode deployment, we have the master + a region server and zookeeper server
                // started in the same process. Therefore, we only support one zookeeper server.
                String errorMsg = "Could not start ZK with " + zkservers.length +
                    " ZK servers in local mode deployment. Aborting as clients (e.g. shell) will not "
                    + "be able to find this ZK quorum.";
                  System.err.println(errorMsg);
                  throw new IOException(errorMsg);
              }
    
              String[] parts = zkservers[0].split(":");
    
              if (parts.length == 2) {
                // the second part is the client port
                zkClientPort = Integer.parseInt(parts [1]);
              }
            }
            // If the client port could not be find in server quorum conf, try another conf
            if (zkClientPort == 0) {
              zkClientPort = conf.getInt(HConstants.ZOOKEEPER_CLIENT_PORT, 0);
              // The client port has to be set by now; if not, throw exception.
              if (zkClientPort == 0) {
                throw new IOException("No config value for " + HConstants.ZOOKEEPER_CLIENT_PORT);
              }
            }
            zooKeeperCluster.setDefaultClientPort(zkClientPort);
            // set the ZK tick time if specified
            int zkTickTime = conf.getInt(HConstants.ZOOKEEPER_TICK_TIME, 0);
            if (zkTickTime > 0) {
              zooKeeperCluster.setTickTime(zkTickTime);
            }
    
            // 如果启用了安全认证，需要配置 Zookeeper 的 keytab 文件和 principal 等
            // login the zookeeper server principal (if using security)
            ZKUtil.loginServer(conf, HConstants.ZK_SERVER_KEYTAB_FILE,
              HConstants.ZK_SERVER_KERBEROS_PRINCIPAL, null);
            int localZKClusterSessionTimeout =
              conf.getInt(HConstants.ZK_SESSION_TIMEOUT + ".localHBaseCluster", 10*1000);
            conf.setInt(HConstants.ZK_SESSION_TIMEOUT, localZKClusterSessionTimeout);
            LOG.info("Starting a zookeeper cluster");
            
            // 启动 Zookeeper 服务
            int clientPort = zooKeeperCluster.startup(zkDataPath);
            // Zookeeper 启动失败会输出相应信息
            if (clientPort != zkClientPort) {
              String errorMsg = "Could not start ZK at requested port of " +
                zkClientPort + ".  ZK was started at port: " + clientPort +
                ".  Aborting as clients (e.g. shell) will not be able to find " +
                "this ZK quorum.";
              System.err.println(errorMsg);
              throw new IOException(errorMsg);
            }
            // 启动成功则设置 HBase 有关 Zookeeper 的参数
            conf.set(HConstants.ZOOKEEPER_CLIENT_PORT, Integer.toString(clientPort));
    
            // Need to have the zk cluster shutdown when master is shutdown.
            // Run a subclass that does the zk cluster shutdown on its way out.
            int mastersCount = conf.getInt("hbase.masters", 1);
            int regionServersCount = conf.getInt("hbase.regionservers", 1);
            // Set start timeout to 5 minutes for cmd line start operations
            conf.setIfUnset("hbase.master.start.timeout.localHBaseCluster", "300000");
            LOG.info("Starting up instance of localHBaseCluster; master=" + mastersCount +
              ", regionserversCount=" + regionServersCount);
            
            // LocalHMaster 继承自 HMaster，和 HRegionServer 同时启动，在停止的同时也停止 Zookeeper 服务
            LocalHBaseCluster cluster = new LocalHBaseCluster(conf, mastersCount, regionServersCount,
              LocalHMaster.class, HRegionServer.class);
            
            // 将运行的 zooKeeperCluster 置于 LocalHMaster 中，以便在 LocalHMaster 停止的时候停止 Zookeeper 服务
            ((LocalHMaster)cluster.getMaster(0)).setZKCluster(zooKeeperCluster);
            // 调用 LocalHBaseCluster 的 startup 方法启动
            cluster.startup();
            waitOnMasterThreads(cluster);
          } else {
            // 启动集群模式
            // 记录有关当前正在运行的JVM进程的信息，包括环境变量，可以通过配置 hbase.envvars.logging.disabled 为 true 禁用
            logProcessInfo(getConf());
            // 通过反射 HMaster 的构造方法对其进行实例化
            HMaster master = HMaster.constructMaster(masterClass, conf);
            // 如果此时请求关闭 HMaster 则不会启动
            if (master.isStopped()) {
              LOG.info("Won't bring the Master up as a shutdown is requested");
              return 1;
            }
            // 启动 HMaster，调用 HMaster 的 run 方法进行处理
            master.start();
            // 等待 HMaster 启动成功
            master.join();
            // 异常信息则输出错误信息
            if(master.isAborted())
              throw new RuntimeException("HMaster Aborted");
          }
        } catch (Throwable t) {
          LOG.error("Master exiting", t);
          return 1;
        }
        return 0;
      }
      
    // 由 HMasterCommandLine 在 startMaster 方法中启动单机模式的 HBase 时调用  
    private void waitOnMasterThreads(LocalHBaseCluster cluster) throws InterruptedException{
        List<JVMClusterUtil.MasterThread> masters = cluster.getMasters();
        List<JVMClusterUtil.RegionServerThread> regionservers = cluster.getRegionServers();
    
        if (masters != null) {
          for (JVMClusterUtil.MasterThread t : masters) {
            // 先等待 MasterThread 启动完成再启动 RegionServerThread，如果出现异常则关闭 RegionServer 并输出错误信息
            t.join();
            if(t.getMaster().isAborted()) {
              closeAllRegionServerThreads(regionservers);
              throw new RuntimeException("HMaster Aborted");
            }
          }
        }
      }
}

---

public class LocalHBaseCluster {

    public LocalHBaseCluster(final Configuration conf, final int noMasters,
        final int noRegionServers, final Class<? extends HMaster> masterClass,
        final Class<? extends HRegionServer> regionServerClass)
      throws IOException {
        this.conf = conf;
    
        // 获取及配置 HBase 相关参数
        ……
        
        // 设置 masterClass，此处即为传入的 LocalHMaster
        this.masterClass = (Class<? extends HMaster>)
          conf.getClass(HConstants.MASTER_IMPL, masterClass);
          
        // 最终调用至 JVMClusterUtil 工具类的 createMasterThread 方法，通过反射调用继承自 HMaster 的子类构造方法进行实例化，得到 MasterThread
        for (int i = 0; i < noMasters; i++) {
          addMaster(new Configuration(conf), i);
        }
        
        // 设置 regionServerClass，此处即为传入的 HRegionServer
        this.regionServerClass =
          (Class<? extends HRegionServer>)conf.getClass(HConstants.REGION_SERVER_IMPL,
           regionServerClass);
    
        // 最终调用至 JVMClusterUtil 工具类的 createRegionServerThread 方法，通过反射调用继承自 HRegionServer 的子类构造方法进行实例化，得到 RegionServerThread
        for (int i = 0; i < noRegionServers; i++) {
          addRegionServer(new Configuration(conf), i);
        }
      }
      
    // 启动前面实例化好的 masterThreads 和 regionThreads，等待启动完成，至此单机模式下的 HMaster 和 HRegionServer 均已启动，可以正常提供服务
    public void startup() throws IOException {
        JVMClusterUtil.startup(this.masterThreads, this.regionThreads);
    }
}

---

public static class LocalHMaster extends HMaster {
    private MiniZooKeeperCluster zkcluster = null;

    public LocalHMaster(Configuration conf)
    throws IOException, KeeperException, InterruptedException {
      super(conf);
    }

    @Override
    public void run() {
      // 调用父类 HMaster 的 run 方法
      super.run();
      // 调用 MiniZooKeeperCluster 的 shutdown 方法，停止单机模式下的 Zookeeper 服务
      if (this.zkcluster != null) {
        try {
          this.zkcluster.shutdown();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }

    void setZKCluster(final MiniZooKeeperCluster zkcluster) {
      this.zkcluster = zkcluster;
    }
  }
```

### hbase-config.sh

用于获取配置参数的脚本，会去加载 hbase-env.sh 中设置的环境变量。

```shell
# 获取当前路径
this="${BASH_SOURCE-$0}"

# 解析 ${BASH_SOURCE-$0} 有可能是 softlink 的问题
while [ -h "$this" ]; do
  ls=`ls -ld "$this"`
  link=`expr "$ls" : '.*-> \(.*\)$'`
  if expr "$link" : '.*/.*' > /dev/null; then
    this="$link"
  else
    this=`dirname "$this"`/"$link"
  fi
done

# convert relative path to absolute path
bin=`dirname "$this"`
script=`basename "$this"`
bin=`cd "$bin">/dev/null; pwd`
this="$bin/$script"

# 将 HBASE_HOME 设置为 HBase 安装的根目录
if [ -z "$HBASE_HOME" ]; then
  export HBASE_HOME=`dirname "$this"`/..
fi

# 检查是否有可选参数传入，接受到则进行相应的参数设置
……

# 以下项为参数设置
# Allow alternate hbase conf dir location.
HBASE_CONF_DIR="${HBASE_CONF_DIR:-$HBASE_HOME/conf}"
# List of hbase regions servers.
HBASE_REGIONSERVERS="${HBASE_REGIONSERVERS:-$HBASE_CONF_DIR/regionservers}"
# List of hbase secondary masters.
HBASE_BACKUP_MASTERS="${HBASE_BACKUP_MASTERS:-$HBASE_CONF_DIR/backup-masters}"
if [ -n "$HBASE_JMX_BASE" ] && [ -z "$HBASE_JMX_OPTS" ]; then
  HBASE_JMX_OPTS="$HBASE_JMX_BASE"
fi

……

# 加载 hbase-env.sh
if [ -z "$HBASE_ENV_INIT" ] && [ -f "${HBASE_CONF_DIR}/hbase-env.sh" ]; then
  . "${HBASE_CONF_DIR}/hbase-env.sh"
  export HBASE_ENV_INIT="true"
fi

# 检测 HBASE_REGIONSERVER_MLOCK 是否设置为 true，主要是判断系统是否使用了 mlock 来锁住内存，防止这段内存被操作系统放到 swap 空间，即使该程序已经有一段时间没有访问这段空间
if [ "$HBASE_REGIONSERVER_MLOCK" = "true" ]; then
  MLOCK_AGENT="$HBASE_HOME/lib/native/libmlockall_agent.so"
  if [ ! -f "$MLOCK_AGENT" ]; then
    cat 1>&2 <<EOF
Unable to find mlockall_agent, hbase must be compiled with -Pnative
EOF
    exit 1
  fi
  // 配置 HBASE_REGIONSERVER_UID
  if [ -z "$HBASE_REGIONSERVER_UID" ] || [ "$HBASE_REGIONSERVER_UID" == "$USER" ]; then
      HBASE_REGIONSERVER_OPTS="$HBASE_REGIONSERVER_OPTS -agentpath:$MLOCK_AGENT"
  else
      HBASE_REGIONSERVER_OPTS="$HBASE_REGIONSERVER_OPTS -agentpath:$MLOCK_AGENT=user=$HBASE_REGIONSERVER_UID"
  fi
fi

……

# 检查是否配置了 jdk，HBase-2.2.4 至少需要 1.8 以上的 JDK 版本，未配置则退出
# Now having JAVA_HOME defined is required 
if [ -z "$JAVA_HOME" ]; then
    cat 1>&2 <<EOF
……
fi
```

### hbase-daemons.sh

根据要启动的进程，生成好远程执行命令 remote_cmd 并调用其他脚本执行。

```shell
# 脚本用法
usage="Usage: hbase-daemons.sh [--config <hbase-confdir>] [--autostart-window-size <window size in hours>]\
      [--autostart-window-retry-limit <retry count limit for autostart>] \
      [--hosts regionserversfile] [autostart|autorestart|restart|start|stop] command args..."

# 如果没有指定参数，输出 usage
if [ $# -le 1 ]; then
  echo $usage
  exit 1
fi

# 获取当前路径
bin=`dirname "${BASH_SOURCE-$0}"`
bin=`cd "$bin">/dev/null; pwd`

# 默认的自动启动参数相关配置，一般前面脚本都是传递诸如 start 参数过来，可以忽略
AUTOSTART_WINDOW_SIZE=0
AUTOSTART_WINDOW_RETRY_LIMIT=0

# 加载 hbase-config.sh 脚本
. $bin/hbase-config.sh

……

# 调用 hbase-daemon.sh 并向其传递参数
remote_cmd="$bin/hbase-daemon.sh --config ${HBASE_CONF_DIR} ${autostart_args} $@"
# 将 $remote_cmd 作为参数继续包装到 args 中
args="--hosts ${HBASE_REGIONSERVERS} --config ${HBASE_CONF_DIR} $remote_cmd"

# 接受到的第二个参数的值，例如 start-hbase.sh 集群模式下传递了 zookeeper、regionserver 等，基于该参数分别调用相应的脚本执行，执行后退出
command=$2
case $command in
  (zookeeper)
    exec "$bin/zookeepers.sh" $args
    ;;
  (master-backup)
    exec "$bin/master-backup.sh" $args
    ;;
  (*)
    exec "$bin/regionservers.sh" $args
    ;;
esac
```

### hbase-daemon.sh

这个脚本很重要，前面的脚本都是做一些准备工作，它负责启动前的检查清理、日志滚动以及进程的启动等等。支持 7 种方式：start、autostart、autorestart、foreground_start、internal_autostart、stop、restart，其他参数则输出操作用法。

```shell
# 将 Hadoop hbase 命令作为守护程序执行
# 环境变量

# 配置文件目录，默认是 ${HBASE_HOME}/conf
# HBASE_CONF_DIR

# 日志存储目录，默认情况下为 pwd
# HBASE_LOG_DIR

# 进程号存放目录，默认是 /tmp
# HBASE_PID_DIR

# 代表当前 hadoop 实例的字符串，默认是当前点用户 $USER
# HBASE_IDENT_STRING

# 守护程序的调度优先级，默认是 0
# HBASE_NICENESS

# 在停止服务此时间之后，服务还未停止，将对其执行 kill -9 命令，默认 1200（单位是 s）
# HBASE_STOP_TIMEOUT

# 仿照了 $HADOOP_HOME/bin/hadoop-daemon.sh
usage="Usage: hbase-daemon.sh [--config <conf-dir>]\
 [--autostart-window-size <window size in hours>]\
 [--autostart-window-retry-limit <retry count limit for autostart>]\
 (start|stop|restart|autostart|autorestart|foreground_start) <hbase-command> \
 <args...>"

# 如果没有指定参数，输出 usage
if [ $# -le 1 ]; then
  echo $usage
  exit 1
fi

# 默认的自动启动配置参数
AUTOSTART_WINDOW_SIZE=0
AUTOSTART_WINDOW_RETRY_LIMIT=0

# 获取当前路径
bin=`dirname "${BASH_SOURCE-$0}"`
bin=`cd "$bin">/dev/null; pwd`

# 加载 hbase-config.sh 以及 hbase-common.sh
. "$bin"/hbase-config.sh
. "$bin"/hbase-common.sh

# 获取传递的参数，即 start 或 stop
startStop=$1
# 命令左移，shift 命令每执行一次，变量的个数($#)减一，而变量值提前一位
shift

command=$1
shift

# 日志滚动
hbase_rotate_log ()
{
    log=$1;
    num=5;
    if [ -n "$2" ]; then
    num=$2
    fi
    # 检查是否存在日志文件，若存在则进行日志滚动
    if [ -f "$log" ]; then # rotate logs
    while [ $num -gt 1 ]; do
        prev=`expr $num - 1`
        [ -f "$log.$prev" ] && mv -f "$log.$prev" "$log.$num"
        num=$prev
    done
    # 修改日志文件名
    mv -f "$log" "$log.$num";
    fi
}

# 当运行遇到问题时进行清理，在 foreground_start 方式中，接收到 SIGHUP（终端线路挂断） SIGINT（中断进程） SIGTERM（软件终止信号） EXIT（退出）等信号时，
# 使用 trap 命令对要处理的信号名采取相应的行动，即 kill 掉正在运行的进程，并通知 Zookeeper 删除节点
cleanAfterRun() {
  if [ -f ${HBASE_PID} ]; then
    # If the process is still running time to tear it down.
    kill -9 `cat ${HBASE_PID}` > /dev/null 2>&1
    rm -f ${HBASE_PID} > /dev/null 2>&1
  fi

  if [ -f ${HBASE_ZNODE_FILE} ]; then
    if [ "$command" = "master" ]; then
      HBASE_OPTS="$HBASE_OPTS $HBASE_MASTER_OPTS" $bin/hbase master clear > /dev/null 2>&1
    else
      # call ZK to delete the node
      ZNODE=`cat ${HBASE_ZNODE_FILE}`
      HBASE_OPTS="$HBASE_OPTS $HBASE_REGIONSERVER_OPTS" $bin/hbase zkcli delete ${ZNODE} > /dev/null 2>&1
    fi
    rm ${HBASE_ZNODE_FILE}
  fi
}

# 启动之间先检查进程是否存在，存在则输出警告信息
check_before_start(){
    #ckeck if the process is not running
    mkdir -p "$HBASE_PID_DIR"
    if [ -f $HBASE_PID ]; then
      # kill -0 pid 不发送任何信号，但是系统会进行错误检查，检查一个进程是否存在，存在返回 0；不存在返回 1
      if kill -0 `cat $HBASE_PID` > /dev/null 2>&1; then
        echo $command running as process `cat $HBASE_PID`.  Stop it first.
        exit 1
      fi
    fi
}

# 等待命令执行完成，超过 HBASE_SLAVE_TIMEOUT 即 300 之后则调用 kill -9 杀掉服务并输出警告信息
wait_until_done ()
{
    p=$1
    cnt=${HBASE_SLAVE_TIMEOUT:-300}
    origcnt=$cnt
    # 进程仍在运行，睡眠 1s 后重新判断，直到超过指定次数（时间）调用 kill -9 $pid
    while kill -0 $p > /dev/null 2>&1; do
      if [ $cnt -gt 1 ]; then
        cnt=`expr $cnt - 1`
        sleep 1
      else
        echo "Process did not complete after $origcnt seconds, killing."
        kill -9 $p
        exit 1
      fi
    done
    return 0
}

# 获取日志目录
if [ "$HBASE_LOG_DIR" = "" ]; then
  export HBASE_LOG_DIR="$HBASE_HOME/logs"
fi
mkdir -p "$HBASE_LOG_DIR"

# 如果没有配置 HBASE_PID_DIR 目录，则默认为 /tmp
if [ "$HBASE_PID_DIR" = "" ]; then
  HBASE_PID_DIR=/tmp
fi

if [ "$HBASE_IDENT_STRING" = "" ]; then
  export HBASE_IDENT_STRING="$USER"
fi

# 配置 JAVA_HOME
if [ "$JAVA_HOME" != "" ]; then
  #echo "run java in $JAVA_HOME"
  JAVA_HOME=$JAVA_HOME
fi
if [ "$JAVA_HOME" = "" ]; then
  echo "Error: JAVA_HOME is not set."
  exit 1
fi

JAVA=$JAVA_HOME/bin/java
# 日志前缀，如：hbase-root-master-node1
export HBASE_LOG_PREFIX=hbase-$HBASE_IDENT_STRING-$command-$HOSTNAME
export HBASE_LOGFILE=$HBASE_LOG_PREFIX.log

# 如果未配置 HBASE_ROOT_LOGGER 参数，则设置默认的日志级别
if [ -z "${HBASE_ROOT_LOGGER}" ]; then
export HBASE_ROOT_LOGGER=${HBASE_ROOT_LOGGER:-"INFO,RFA"}
fi

# 如果未配置 HBASE_SECURITY_LOGGER 参数，则设置默认的安全日志级别
if [ -z "${HBASE_SECURITY_LOGGER}" ]; then
export HBASE_SECURITY_LOGGER=${HBASE_SECURITY_LOGGER:-"INFO,RFAS"}
fi

# out 日志，即 System.out 输出信息
HBASE_LOGOUT=${HBASE_LOGOUT:-"$HBASE_LOG_DIR/$HBASE_LOG_PREFIX.out"}
HBASE_LOGGC=${HBASE_LOGGC:-"$HBASE_LOG_DIR/$HBASE_LOG_PREFIX.gc"}
HBASE_LOGLOG=${HBASE_LOGLOG:-"${HBASE_LOG_DIR}/${HBASE_LOGFILE}"}
# HBase 相关服务进程
HBASE_PID=$HBASE_PID_DIR/hbase-$HBASE_IDENT_STRING-$command.pid
# HBase 的 znode 文件
export HBASE_ZNODE_FILE=$HBASE_PID_DIR/hbase-$HBASE_IDENT_STRING-$command.znode
# HBase 的 autostart 文件
export HBASE_AUTOSTART_FILE=$HBASE_PID_DIR/hbase-$HBASE_IDENT_STRING-$command.autostart

# 如果配置了 SERVER_GC_OPTS、CLIENT_GC_OPTS 参数，则设置对应变量
if [ -n "$SERVER_GC_OPTS" ]; then
  export SERVER_GC_OPTS=${SERVER_GC_OPTS/"-Xloggc:<FILE-PATH>"/"-Xloggc:${HBASE_LOGGC}"}
fi
if [ -n "$CLIENT_GC_OPTS" ]; then
  export CLIENT_GC_OPTS=${CLIENT_GC_OPTS/"-Xloggc:<FILE-PATH>"/"-Xloggc:${HBASE_LOGGC}"}
fi

# 设置默认的程度调度优先级为 0
if [ "$HBASE_NICENESS" = "" ]; then
    export HBASE_NICENESS=0
fi

# 获取当前路径
thiscmd="$bin/$(basename ${BASH_SOURCE-$0})"
args=$@

case $startStop in

(start)
    check_before_start
    hbase_rotate_log $HBASE_LOGOUT
    hbase_rotate_log $HBASE_LOGGC
    # 输出启动的程序，以及日志输出目录，接着调用 foreground_start 
    echo running $command, logging to $HBASE_LOGOUT
    $thiscmd --config "${HBASE_CONF_DIR}" \
        foreground_start $command $args < /dev/null > ${HBASE_LOGOUT} 2>&1  &
    # 使正在运行的作业忽略 HUP 信号，避免当用户注销（logout）或者网络断开时，终端会收到 Linux HUP（hangup）信号从而关闭其所有子进程
    disown -h -r
    sleep 1; head "${HBASE_LOGOUT}"
  ;;

(autostart)
    check_before_start
    hbase_rotate_log $HBASE_LOGOUT
    hbase_rotate_log $HBASE_LOGGC
    echo running $command, logging to $HBASE_LOGOUT
    # 使用 nohup 挂起并执行自动启动程序，调用 internal_autostart 继续执行
    nohup $thiscmd --config "${HBASE_CONF_DIR}" --autostart-window-size ${AUTOSTART_WINDOW_SIZE} --autostart-window-retry-limit ${AUTOSTART_WINDOW_RETRY_LIMIT} \
        internal_autostart $command $args < /dev/null > ${HBASE_LOGOUT} 2>&1  &
  ;;

(autorestart)
    echo running $command, logging to $HBASE_LOGOUT
    # 先停止当前服务，并等待所有进程都停止
    $thiscmd --config "${HBASE_CONF_DIR}" stop $command $args &
    wait_until_done $!
    # 等待用户指定的睡眠周期
    sp=${HBASE_RESTART_SLEEP:-3}
    if [ $sp -gt 0 ]; then
      sleep $sp
    fi

    check_before_start
    hbase_rotate_log $HBASE_LOGOUT
    # 使用 nohup 挂起并执行自动启动程序，调用 internal_autostart 继续执行
    nohup $thiscmd --config "${HBASE_CONF_DIR}" --autostart-window-size ${AUTOSTART_WINDOW_SIZE} --autostart-window-retry-limit ${AUTOSTART_WINDOW_RETRY_LIMIT} \
        internal_autostart $command $args < /dev/null > ${HBASE_LOGOUT} 2>&1  &
  ;;

(foreground_start)
    trap cleanAfterRun SIGHUP SIGINT SIGTERM EXIT
    # 日志不重定向参数，一般都输出到日志中，这部分逻辑主要走 else
    if [ "$HBASE_NO_REDIRECT_LOG" != "" ]; then
        # NO REDIRECT
        echo "`date` Starting $command on `hostname`"
        echo "`ulimit -a`"
        # in case the parent shell gets the kill make sure to trap signals.
        # Only one will get called. Either the trap or the flow will go through.
        nice -n $HBASE_NICENESS "$HBASE_HOME"/bin/hbase \
            --config "${HBASE_CONF_DIR}" \
            $command "$@" start &
    else
        echo "`date` Starting $command on `hostname`" >> ${HBASE_LOGLOG}
        echo "`ulimit -a`" >> "$HBASE_LOGLOG" 2>&1
        # nice 以更改过的优先序来执行程序，调用 $HBASE_HOME/bin/hbase 传递参数继续执行
        nice -n $HBASE_NICENESS "$HBASE_HOME"/bin/hbase \
            --config "${HBASE_CONF_DIR}" \
            $command "$@" start >> ${HBASE_LOGOUT} 2>&1 &
    fi
    # 获取最后一个进程号，将其覆盖写入 $HBASE_PID 文件中，暂停当前进程并释放资源等待前面的线程执行
    hbase_pid=$!
    echo $hbase_pid > ${HBASE_PID}
    wait $hbase_pid
  ;;

(internal_autostart)
    ONE_HOUR_IN_SECS=3600
    # 自动启动的开始日期
    autostartWindowStartDate=`date +%s`
    autostartCount=0
    # 创建自动启动的文件
    touch "$HBASE_AUTOSTART_FILE"

    # 除非被要求停止，否则一直保持启动命令的状态，在崩溃时重新进入循环
    while true
    do
      hbase_rotate_log $HBASE_LOGGC
      if [ -f $HBASE_PID ] &&  kill -0 "$(cat "$HBASE_PID")" > /dev/null 2>&1 ; then
        wait "$(cat "$HBASE_PID")"
      else
        # 如果 $HBASE_AUTOSTART_FILE 不存在，说明服务可能不是通过 stop 命令停止的
        if [ ! -f "$HBASE_AUTOSTART_FILE" ]; then
          echo "`date` HBase might be stopped removing the autostart file. Exiting Autostart process" >> ${HBASE_LOGOUT}
          exit 1
        fi

        echo "`date` Autostarting hbase $command service. Attempt no: $(( $autostartCount + 1))" >> ${HBASE_LOGLOG}
        touch "$HBASE_AUTOSTART_FILE"
        $thiscmd --config "${HBASE_CONF_DIR}" foreground_start $command $args
        autostartCount=$(( $autostartCount + 1 ))

        # HBASE-6504 - 仅当输出详细gc时，才采用输出的第一行
        distMode=`$bin/hbase --config "$HBASE_CONF_DIR" org.apache.hadoop.hbase.util.HBaseConfTool hbase.cluster.distributed | head -n 1`

        if [ "$distMode" != 'false' ]; then
          # 如果集群正在被停止，不再重启
          zparent=`$bin/hbase org.apache.hadoop.hbase.util.HBaseConfTool zookeeper.znode.parent`
          # 创建对应的 znode 并设置服务运行状态
          if [ "$zparent" == "null" ]; then zparent="/hbase"; fi
          zkrunning=`$bin/hbase org.apache.hadoop.hbase.util.HBaseConfTool zookeeper.znode.state`
          if [ "$zkrunning" == "null" ]; then zkrunning="running"; fi
          zkFullRunning=$zparent/$zkrunning
          $bin/hbase zkcli stat $zkFullRunning 2>&1 | grep "Node does not exist"  1>/dev/null 2>&1

          # 如果发现上述指令的 grep 匹配到结果，则说明遇到了问题，显示警告信息，处理后退出
          if [ $? -eq 0 ]; then
            echo "`date` hbase znode does not exist. Exiting Autostart process" >> ${HBASE_LOGOUT}
            # 删除 $HBASE_AUTOSTART_FILE 文件
            rm -f "$HBASE_AUTOSTART_FILE"
            exit 1
          fi

          # 如果没有找到 Zookeeper 服务，就不重启并显示警告信息
          $bin/hbase zkcli stat $zkFullRunning 2>&1 | grep Exception | grep ConnectionLoss  1>/dev/null 2>&1
          if [ $? -eq 0 ]; then
            echo "`date` zookeeper not found. Exiting Autostart process" >> ${HBASE_LOGOUT}
            rm -f "$HBASE_AUTOSTART_FILE"
            exit 1
          fi
        fi
      fi

      // 当前日期
      curDate=`date +%s`
      // 是否重新设置自动启动窗口
      autostartWindowReset=false

      # 假如超过了自动启动的窗口大小，就重新设置一下
      if [ $AUTOSTART_WINDOW_SIZE -gt 0 ] && [ $(( $curDate - $autostartWindowStartDate )) -gt $(( $AUTOSTART_WINDOW_SIZE * $ONE_HOUR_IN_SECS )) ]; then
        echo "Resetting Autorestart window size: $autostartWindowStartDate" >> ${HBASE_LOGOUT}
        autostartWindowStartDate=$curDate
        autostartWindowReset=true
        autostartCount=0
      fi

      # 当重试次数超过了给定的窗口大小限制（窗口大小不是 0），就杀掉程序，处理后退出
      if ! $autostartWindowReset && [ $AUTOSTART_WINDOW_RETRY_LIMIT -gt 0 ] && [ $autostartCount -gt $AUTOSTART_WINDOW_RETRY_LIMIT ]; then
        echo "`date` Autostart window retry limit: $AUTOSTART_WINDOW_RETRY_LIMIT exceeded for given window size: $AUTOSTART_WINDOW_SIZE hours.. Exiting..." >> ${HBASE_LOGLOG}
        rm -f "$HBASE_AUTOSTART_FILE"
        exit 1
      fi

      # 等待关闭的钩子完成
      sleep 20
    done
  ;;

(stop)
    echo running $command, logging to $HBASE_LOGOUT
    rm -f "$HBASE_AUTOSTART_FILE"
    # 判断是否存在进程号的文件
    if [ -f $HBASE_PID ]; then
      pidToKill=`cat $HBASE_PID`
      # 执行 kill -0 以确认进程是否在运行，如果在运行则传递 kill 信号，调用 hbase-common.sh 的 waitForProcessEnd 函数等待执行
      if kill -0 $pidToKill > /dev/null 2>&1; then
        echo -n stopping $command
        echo "`date` Terminating $command" >> $HBASE_LOGLOG
        kill $pidToKill > /dev/null 2>&1
        waitForProcessEnd $pidToKill $command
      else
        retval=$?
        echo no $command to stop because kill -0 of pid $pidToKill failed with status $retval
      fi
    else
      echo no $command to stop because no pid file $HBASE_PID
    fi
    rm -f $HBASE_PID
  ;;

(restart)
    echo running $command, logging to $HBASE_LOGOUT
    # 停止服务
    $thiscmd --config "${HBASE_CONF_DIR}" stop $command $args &
    wait_until_done $!
    # 等待用户指定的睡眠周期
    sp=${HBASE_RESTART_SLEEP:-3}
    if [ $sp -gt 0 ]; then
      sleep $sp
    fi
    # 启动服务
    $thiscmd --config "${HBASE_CONF_DIR}" start $command $args &
    wait_until_done $!
  ;;

(*)
  echo $usage
  exit 1
  ;;
esac
```

### hbase-common.sh

仅有 waitForProcessEnd 方法，是个共享函数，用于等待进程结束，以 pid 和命令名称为参数。

```shell
waitForProcessEnd() {
  # 待停止的进程号
  pidKilled=$1
  # 服务名
  commandName=$2
  processedAt=`date +%s`
  # 判断进程是否仍在运行
  while kill -0 $pidKilled > /dev/null 2>&1;
   do
     echo -n "."
     sleep 1;
     # 如果进程持续的时间超过 $HBASE_STOP_TIMEOUT 即 1200s，不再等待，继续往下执行
     if [ $(( `date +%s` - $processedAt )) -gt ${HBASE_STOP_TIMEOUT:-1200} ]; then
       break;
     fi
   done
  # 如果进程仍在运行，执行 kill -9
  if kill -0 $pidKilled > /dev/null 2>&1; then
    echo -n force stopping $commandName with kill -9 $pidKilled
    $JAVA_HOME/bin/jstack -l $pidKilled > "$logout" 2>&1
    kill -9 $pidKilled > /dev/null 2>&1
  fi
  # Add a CR after we're done w/ dots.
  echo
}
```


### zookeepers.sh

接收 hbase-daemon.sh 中传递的参数，在所有的 Zookeeper 主机上执行命令。

```shell
# 环境变量

# 配置文件目录，默认是 ${HBASE_HOME}/conf
# HBASE_CONF_DIR

# 在生成远程命令的时候睡眠的时间，默认未设置
# HBASE_SLAVE_SLEEP

# 执行远程命令时，传递给 ssh 的选项
# HBASE_SSH_OPTS

# 仿照 $HADOOP_HOME/bin/slaves.sh

usage="Usage: zookeepers [--config <hbase-confdir>] command..."

# 如果没有指定参数，输出 usage
if [ $# -le 0 ]; then
  echo $usage
  exit 1
fi

# 获取当前路径
bin=`dirname "${BASH_SOURCE-$0}"`
bin=`cd "$bin">/dev/null; pwd`

# 加载 hbase-config.sh
. "$bin"/hbase-config.sh

# 如果 $HBASE_MANAGES_ZK 参数未配置，即 hbase-env.sh 中的 export HBASE_MANAGES_ZK=true 注释没打开，则将此参数设置为 true
if [ "$HBASE_MANAGES_ZK" = "" ]; then
  HBASE_MANAGES_ZK=true
fi

# 调用 $bin/hbase 脚本，运行 ZKServerTool 类获取 Zookeeper 所有的主机，通过 grep 和 sed 命令对结果进行处理，见附 3，$"${@// /\\ }"会将命令中将所有的 \ 替换成为空格
if [ "$HBASE_MANAGES_ZK" = "true" ]; then
  hosts=`"$bin"/hbase org.apache.hadoop.hbase.zookeeper.ZKServerTool | grep '^ZK host:' | sed 's,^ZK host:,,'`
  cmd=$"${@// /\\ }"
  # 在所有的主机上启动 Zookeeper 服务
  for zookeeper in $hosts; do
   ssh $HBASE_SSH_OPTS $zookeeper $cmd 2>&1 | sed "s/^/$zookeeper: /" &
   if [ "$HBASE_SLAVE_SLEEP" != "" ]; then
     sleep $HBASE_SLAVE_SLEEP
   fi
  done
fi

wait
```

#### 附 3

通过 $bin/hbase 运行此类。

```java
public final class ZKServerTool {

    private ZKServerTool() {
    }
    
    public static ServerName[] readZKNodes(Configuration conf) {
        List<ServerName> hosts = new LinkedList<>();
        // 从 conf 中获取 hbase.zookeeper.quorum 参数对应的值，默认为 localhost
        String quorum = conf.get(HConstants.ZOOKEEPER_QUORUM, HConstants.LOCALHOST);
        
        String[] values = quorum.split(",");
        for (String value : values) {
          String[] parts = value.split(":");
          String host = parts[0];
          // 默认端口 2181
          int port = HConstants.DEFAULT_ZOOKEEPER_CLIENT_PORT;
          if (parts.length > 1) {
            port = Integer.parseInt(parts[1]);
          }
          hosts.add(ServerName.valueOf(host, port, -1));
        }
        // 转换成数组输出
        return hosts.toArray(new ServerName[hosts.size()]);
    }
    
    public static void main(String[] args) {
        for(ServerName server: readZKNodes(HBaseConfiguration.create())) {
          // bin/zookeeper.sh 依赖于 "ZK host" 字符串进行 grep 操作，区分大小写
          System.out.println("ZK host: " + server.getHostname());
        }
    }
}
```

### master-backup.sh

接收 hbase-daemon.sh 中传递的参数，在所有的 backup master 主机上执行命令。

```shell
# 环境变量

# 远程主机文件命名，默认是 ${HBASE_CONF_DIR}/backup-masters
# HBASE_BACKUP_MASTERS

# Hadoop 配置文件路径，默认是 ${HADOOP_HOME}/conf
# HADOOP_CONF_DIR

# HBase 配置文件路径，默认是 ${HBASE_HOME}/conf
# HBASE_CONF_DIR

# 在生成远程命令的时候睡眠的时间，默认未设置
# HBASE_SLAVE_SLEEP

# 执行远程命令时，传递给 ssh 的选项
# HBASE_SSH_OPTS

# 仿照 $HADOOP_HOME/bin/slaves.sh

usage="Usage: $0 [--config <hbase-confdir>] command..."

# 如果没有指定参数，输出 usage
if [ $# -le 0 ]; then
  echo $usage
  exit 1
fi

# 获取当前路径
bin=`dirname "${BASH_SOURCE-$0}"`
bin=`cd "$bin">/dev/null; pwd`

# 加载 hbase-config.sh
. "$bin"/hbase-config.sh

# 如果在命令行中指定了 master backup 文件，那么优先级高于 hbase-env.sh 中的配置，此处进行保存
HOSTLIST=$HBASE_BACKUP_MASTERS

if [ "$HOSTLIST" = "" ]; then
  if [ "$HBASE_BACKUP_MASTERS" = "" ]; then
    export HOSTLIST="${HBASE_CONF_DIR}/backup-masters"
  else
    export HOSTLIST="${HBASE_BACKUP_MASTERS}"
  fi
fi

# $"${@// /\\ }"会将命令中将所有的 \ 替换成为空格
args=${@// /\\ }
args=${args/master-backup/master}

# 登陆到每个节点上，以 backup 的方式启动 master，启动后 Zookeeper 会自动选取一个 master 作为 active，其他的都是 backup
if [ -f $HOSTLIST ]; then
  for hmaster in `cat "$HOSTLIST"`; do
   ssh $HBASE_SSH_OPTS $hmaster $"$args --backup" \
     2>&1 | sed "s/^/$hmaster: /" &
   if [ "$HBASE_SLAVE_SLEEP" != "" ]; then
     sleep $HBASE_SLAVE_SLEEP
   fi
  done
fi 

wait
```

### regionservers.sh

接收 hbase-daemon.sh 中传递的参数，在所有的 RegionServer 主机上执行命令。

```shell
# 环境变量

# 远程主机文件命名，默认是 ${HADOOP_CONF_DIR}/regionservers
# HBASE_REGIONSERVERS

# Hadoop 配置文件路径，默认是 ${HADOOP_HOME}/conf
# HADOOP_CONF_DIR

# HBase 配置文件路径，默认是 ${HBASE_HOME}/conf
# HBASE_CONF_DIR

# 在生成远程命令的时候睡眠的时间，默认未设置
# HBASE_SLAVE_SLEEP

# 执行远程命令时，传递给 ssh 的选项
# HBASE_SSH_OPTS

# 仿照 $HADOOP_HOME/bin/slaves.sh

usage="Usage: regionservers [--config <hbase-confdir>] command..."

# 如果没有指定参数，输出 usage
if [ $# -le 0 ]; then
  echo $usage
  exit 1
fi

# 获取当前路径
bin=`dirname "${BASH_SOURCE-$0}"`
bin=`cd "$bin">/dev/null; pwd`

. "$bin"/hbase-config.sh

# 如果在命令行中指定了 regionservers 文件，那么优先级高于 hbase-env.sh 中的配置，此处进行保存
HOSTLIST=$HBASE_REGIONSERVERS

if [ "$HOSTLIST" = "" ]; then
  if [ "$HBASE_REGIONSERVERS" = "" ]; then
    export HOSTLIST="${HBASE_CONF_DIR}/regionservers"
  else
    export HOSTLIST="${HBASE_REGIONSERVERS}"
  fi
fi

regionservers=`cat "$HOSTLIST"`
# 如果 regionservers 是默认的 localhost，则会在本地启动 regionserver，集群模式则按顺序在各个节点上启动 RegionServer，$"${@// /\\ }"会将命令中将所有的 \ 替换成为空格
if [ "$regionservers" = "localhost" ]; then
  HBASE_REGIONSERVER_ARGS="\
    -Dhbase.regionserver.port=16020 \
    -Dhbase.regionserver.info.port=16030"

  $"${@// /\\ }" ${HBASE_REGIONSERVER_ARGS} \
        2>&1 | sed "s/^/$regionserver: /" &
else
  for regionserver in `cat "$HOSTLIST"`; do
    if ${HBASE_SLAVE_PARALLEL:-true}; then
      ssh $HBASE_SSH_OPTS $regionserver $"${@// /\\ }" \
        2>&1 | sed "s/^/$regionserver: /" &
    else # run each command serially
      ssh $HBASE_SSH_OPTS $regionserver $"${@// /\\ }" \
        2>&1 | sed "s/^/$regionserver: /"
    fi
    if [ "$HBASE_SLAVE_SLEEP" != "" ]; then
      sleep $HBASE_SLAVE_SLEEP
    fi
  done
fi

wait
```

### bin/hbase

hbase 命令脚本，基于 hadoop 命令脚本，它在 hadoop 脚本之前完成了相关配置。

```shell
# 环境变量

# 要使用的 java 实现，覆盖 JAVA_HOME
# JAVA_HOME

# 额外的 Java CLASSPATH
# HBASE_CLASSPATH

# 作为 system classpath 的额外 Java CLASSPATH 的前缀
# HBASE_CLASSPATH_PREFIX

# 使用的最大堆数量，默认未设置，并使用 JVM 的默认设置，通常是可用内存的 1/4
# HBASE_HEAPSIZE

# 对 JAVA_LIBRARY_PATH 的 HBase 添加，用于添加本机库
# HBASE_LIBRARY_PATH

# 额外的 Java 运行时选项
# HBASE_OPTS

# HBase 配置文件路径，默认是 ${HBASE_HOME}/conf
# HBASE_CONF_DIR

#日志追加器，默认是控制台 INFO 级别
# HBASE_ROOT_LOGGER

# JRuby路径：$JRUBY_HOME/lib/jruby.jar 应该存在，默认为 HBase 打包的jar
# JRUBY_HOME

# 额外的选项（例如'--1.9'）传递给了 hbase，默认为空
# JRUBY_OPTS

# 额外的传递给 hbase shell 的选项，默认为空
#   HBASE_SHELL_OPTS

# 获取当前目录
bin=`dirname "$0"`
bin=`cd "$bin">/dev/null; pwd`

# 加载 hbase-config.sh 获取配置
. "$bin"/hbase-config.sh

# 检测系统，是否使用 cygwin
cygwin=false
case "`uname`" in
CYGWIN*) cygwin=true;;
esac

# 检测当前是否在 HBase 的根目录中
in_dev_env=false
if [ -d "${HBASE_HOME}/target" ]; then
  in_dev_env=true
fi

# 检测是否在综合压缩包中
in_omnibus_tarball="false"
if [ -f "${HBASE_HOME}/bin/hbase-daemons.sh" ]; then
  in_omnibus_tarball="true"
fi

# read 当读到 '' 即结束，此处作为一部分用法进行输出
read -d '' options_string << EOF
Options:
  --config DIR         Configuration direction to use. Default: ./conf
  --hosts HOSTS        Override the list in 'regionservers' file
  --auth-as-server     Authenticate to ZooKeeper using servers configuration
  --internal-classpath Skip attempting to use client facing jars (WARNING: unstable results between versions)
EOF
# 如果没有指定参数，输出 usage，包含上面的内容
if [ $# = 0 ]; then
  echo "Usage: hbase [<options>] <command> [<args>]"
  echo "$options_string"
  echo ""
  echo "Commands:"
  echo "Some commands take arguments. Pass no args or -h for usage."
  echo "  shell            Run the HBase shell"
  echo "  hbck             Run the HBase 'fsck' tool. Defaults read-only hbck1."
  echo "                   Pass '-j /path/to/HBCK2.jar' to run hbase-2.x HBCK2."
  echo "  snapshot         Tool for managing snapshots"
  if [ "${in_omnibus_tarball}" = "true" ]; then
    echo "  wal              Write-ahead-log analyzer"
    echo "  hfile            Store file analyzer"
    echo "  zkcli            Run the ZooKeeper shell"
    echo "  master           Run an HBase HMaster node"
    echo "  regionserver     Run an HBase HRegionServer node"
    echo "  zookeeper        Run a ZooKeeper server"
    echo "  rest             Run an HBase REST server"
    echo "  thrift           Run the HBase Thrift server"
    echo "  thrift2          Run the HBase Thrift2 server"
    echo "  clean            Run the HBase clean up script"
  fi
  echo "  classpath        Dump hbase CLASSPATH"
  echo "  mapredcp         Dump CLASSPATH entries required by mapreduce"
  echo "  pe               Run PerformanceEvaluation"
  echo "  ltt              Run LoadTestTool"
  echo "  canary           Run the Canary tool"
  echo "  version          Print the version"
  echo "  completebulkload Run BulkLoadHFiles tool"
  echo "  regionsplitter   Run RegionSplitter tool"
  echo "  rowcounter       Run RowCounter tool"
  echo "  cellcounter      Run CellCounter tool"
  echo "  pre-upgrade      Run Pre-Upgrade validator tool"
  echo "  hbtop            Run HBTop tool"
  echo "  CLASSNAME        Run the class named CLASSNAME"
  exit 1
fi

# 获取传入的第一个参数
COMMAND=$1
# 命令左移
shift

JAVA=$JAVA_HOME/bin/java

# 覆盖此命令的默认设置（如果适用）
if [ -f "$HBASE_HOME/conf/hbase-env-$COMMAND.sh" ]; then
  . "$HBASE_HOME/conf/hbase-env-$COMMAND.sh"
fi

add_size_suffix() {
    # 如果参数缺少一个，则添加一个“m”后缀
    local val="$1"
    local lastchar=${val: -1}
    if [[ "mMgG" == *$lastchar* ]]; then
        echo $val
    else
        echo ${val}m
    fi
}

# 检测 HBASE_HEAPSIZE 是否设置
if [[ -n "$HBASE_HEAPSIZE" ]]; then
    JAVA_HEAP_MAX="-Xmx$(add_size_suffix $HBASE_HEAPSIZE)"
fi

# 检测 HBASE_OFFHEAPSIZE 是否设置
if [[ -n "$HBASE_OFFHEAPSIZE" ]]; then
    JAVA_OFFHEAP_MAX="-XX:MaxDirectMemorySize=$(add_size_suffix $HBASE_OFFHEAPSIZE)"
fi

# 这样在下面的循环中可以正确处理带空格的文件名，设置 IFS
ORIG_IFS=$IFS
IFS=

# CLASSPATH 初始化包含 HBASE_CONF_DIR
CLASSPATH="${HBASE_CONF_DIR}"
CLASSPATH=${CLASSPATH}:$JAVA_HOME/lib/tools.jar

# 如果传入的文件存在，则加入 CLASSPATH
add_to_cp_if_exists() {
  if [ -d "$@" ]; then
    CLASSPATH=${CLASSPATH}:"$@"
  fi
}

# 对于发行版，将 hbase 和 webapp 添加到 CLASSPATH 中 Webapp 必须首先出现，否则会使 Jetty 混乱
if [ -d "$HBASE_HOME/hbase-webapps" ]; then
  add_to_cp_if_exists "${HBASE_HOME}"
fi
# 如果在开发环境中则添加
if [ -d "$HBASE_HOME/hbase-server/target/hbase-webapps" ]; then
  if [ "$COMMAND" = "thrift" ] ; then
    add_to_cp_if_exists "${HBASE_HOME}/hbase-thrift/target"
  elif [ "$COMMAND" = "thrift2" ] ; then
    add_to_cp_if_exists "${HBASE_HOME}/hbase-thrift/target"
  elif [ "$COMMAND" = "rest" ] ; then
    add_to_cp_if_exists "${HBASE_HOME}/hbase-rest/target"
  else
    add_to_cp_if_exists "${HBASE_HOME}/hbase-server/target"
    # 需要下面的 GetJavaProperty 检查
    add_to_cp_if_exists "${HBASE_HOME}/hbase-server/target/classes"
  fi
fi

# 如果可用，将 Hadoop 添加到 CLASSPATH 和 JAVA_LIBRARY_PATH，允许禁用此功能
if [ "$HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP" != "true" ] ; then
  HADOOP_IN_PATH=$(PATH="${HADOOP_HOME:-${HADOOP_PREFIX}}/bin:$PATH" which hadoop 2>/dev/null)
fi

# 声明 shaded_jar，将 libs 添加到 CLASSPATH
declare shaded_jar

if [ "${INTERNAL_CLASSPATH}" != "true" ]; then
  # find our shaded jars
  declare shaded_client
  declare shaded_client_byo_hadoop
  declare shaded_mapreduce
  for f in "${HBASE_HOME}"/lib/shaded-clients/hbase-shaded-client*.jar; do
    if [[ "${f}" =~ byo-hadoop ]]; then
      shaded_client_byo_hadoop="${f}"
    else
      shaded_client="${f}"
    fi
  done
  for f in "${HBASE_HOME}"/lib/shaded-clients/hbase-shaded-mapreduce*.jar; do
    shaded_mapreduce="${f}"
  done

  # 如果命令可以使用 shaded client，使用它
  declare -a commands_in_client_jar=("classpath" "version" "hbtop")
  for c in "${commands_in_client_jar[@]}"; do
    if [ "${COMMAND}" = "${c}" ]; then
      if [ -n "${HADOOP_IN_PATH}" ] && [ -f "${HADOOP_IN_PATH}" ]; then
        # 如果上面没有找到一个 jar，它将为空，然后下面的检查将默认返回内部类路径
        shaded_jar="${shaded_client_byo_hadoop}"
      else
        # 如果上面没有找到一个jar，它将为空，然后下面的检查将默认返回内部类路径
        shaded_jar="${shaded_client}"
      fi
      break
    fi
  done

  # 如果命令需要 shaded mapreduce，使用它
  # 此处不包含 N.B “mapredcp”，因为在 shaded 情况下，它会跳过我们构建的类路径
  declare -a commands_in_mr_jar=("hbck" "snapshot" "canary" "regionsplitter" "pre-upgrade")
  for c in "${commands_in_mr_jar[@]}"; do
    if [ "${COMMAND}" = "${c}" ]; then
      # 如果上面没有找到一个jar，它将为空，然后下面的检查将默认返回内部类路径
      shaded_jar="${shaded_mapreduce}"
      break
    fi
  done

  # 当我们在运行时获得完整的 hadoop 类路径时，某些命令专门只能使用 shaded mapreduce
  if [ -n "${HADOOP_IN_PATH}" ] && [ -f "${HADOOP_IN_PATH}" ]; then
    declare -a commands_in_mr_need_hadoop=("backup" "restore" "rowcounter" "cellcounter")
    for c in "${commands_in_mr_need_hadoop[@]}"; do
      if [ "${COMMAND}" = "${c}" ]; then
        # 如果上面没有找到一个jar，它将为空，然后下面的检查将默认返回内部类路径
        shaded_jar="${shaded_mapreduce}"
        break
      fi
    done
  fi
fi

# 加载相关 jar 包
if [ -n "${shaded_jar}" ] && [ -f "${shaded_jar}" ]; then
  CLASSPATH="${CLASSPATH}:${shaded_jar}"
# fall through to grabbing all the lib jars and hope we're in the omnibus tarball
#
# N.B. shell specifically can't rely on the shaded artifacts because RSGroups is only
# available as non-shaded
#
# N.B. pe and ltt can't easily rely on shaded artifacts because they live in hbase-mapreduce:test-jar
# and need some other jars that haven't been relocated. Currently enumerating that list
# is too hard to be worth it.
#
else
  for f in $HBASE_HOME/lib/*.jar; do
    CLASSPATH=${CLASSPATH}:$f;
  done
  # make it easier to check for shaded/not later on.
  shaded_jar=""
fi
for f in "${HBASE_HOME}"/lib/client-facing-thirdparty/*.jar; do
  if [[ ! "${f}" =~ ^.*/htrace-core-3.*\.jar$ ]] && \
     [ "${f}" != "htrace-core.jar$" ] && \
     [[ ! "${f}" =~ ^.*/slf4j-log4j.*$ ]]; then
    CLASSPATH="${CLASSPATH}:${f}"
  fi
done

# 默认的日志文件目录
if [ "$HBASE_LOG_DIR" = "" ]; then
  HBASE_LOG_DIR="$HBASE_HOME/logs"
fi
# 默认的日志名
if [ "$HBASE_LOGFILE" = "" ]; then
  HBASE_LOGFILE='hbase.log'
fi

# 组装 jar
function append_path() {
  if [ -z "$1" ]; then
    echo "$2"
  else
    echo "$1:$2"
  fi
}

JAVA_PLATFORM=""

# 如果定义了 HBASE_LIBRARY_PATH，则将其用作第一个或第二个选项
if [ "$HBASE_LIBRARY_PATH" != "" ]; then
  JAVA_LIBRARY_PATH=$(append_path "$JAVA_LIBRARY_PATH" "$HBASE_LIBRARY_PATH")
fi

# 如果已配置并且可用，则将 Hadoop 添加到 CLASSPATH 和 JAVA_LIBRARY_PATH
if [ -n "${HADOOP_IN_PATH}" ] && [ -f "${HADOOP_IN_PATH}" ]; then
  # 如果构建了 hbase，则将 hbase-server.jar 临时添加到 GetJavaProperty 的类路径中
  # 排除 hbase-server*-tests.jar
  temporary_cp=
  for f in "${HBASE_HOME}"/lib/hbase-server*.jar; do
    if [[ ! "${f}" =~ ^.*\-tests\.jar$ ]]; then
      temporary_cp=":$f"
    fi
  done
  HADOOP_JAVA_LIBRARY_PATH=$(HADOOP_CLASSPATH="$CLASSPATH${temporary_cp}" "${HADOOP_IN_PATH}" \
                             org.apache.hadoop.hbase.util.GetJavaProperty java.library.path)
  if [ -n "$HADOOP_JAVA_LIBRARY_PATH" ]; then
    JAVA_LIBRARY_PATH=$(append_path "${JAVA_LIBRARY_PATH}" "$HADOOP_JAVA_LIBRARY_PATH")
  fi
  CLASSPATH=$(append_path "${CLASSPATH}" "$(${HADOOP_IN_PATH} classpath 2>/dev/null)")
else
  # 否则，如果我们提供的是 Hadoop，我们还需要使用它的版本构建，则应包括 htrace 3
  for f in "${HBASE_HOME}"/lib/client-facing-thirdparty/htrace-core-3*.jar "${HBASE_HOME}"/lib/client-facing-thirdparty/htrace-core.jar; do
    if [ -f "${f}" ]; then
      CLASSPATH="${CLASSPATH}:${f}"
      break
    fi
  done
  # 使用 shaded jars 时，某些命令需要特殊处理。对于这些情况，我们依赖于 hbase-shaded-mapreduce 而不是 hbase-shaded-client*，因为我们利用了一些 IA.Private 类，这些私有类不再后者中。但是我们不使用"hadoop jar"来调用它们，因此当我们不执行运行时 hadoop 类路径查找时，我们需要确保有一些 Hadoop 类可用。
 # 我们需要的一组类是打包在 shaded-client 中的那些类
  for c in "${commands_in_mr_jar[@]}"; do
    if [ "${COMMAND}" = "${c}" ] && [ -n "${shaded_jar}" ]; then
      CLASSPATH="${CLASSPATH}:${shaded_client:?We couldn\'t find the shaded client jar even though we did find the shaded MR jar. for command ${COMMAND} we need both. please use --internal-classpath as a workaround.}"
      break
    fi
  done
fi

# 最后添加用户指定的 CLASSPATH
if [ "$HBASE_CLASSPATH" != "" ]; then
  CLASSPATH=${CLASSPATH}:${HBASE_CLASSPATH}
fi

# 首先添加用户指定的 CLASSPATH 前缀
if [ "$HBASE_CLASSPATH_PREFIX" != "" ]; then
  CLASSPATH=${HBASE_CLASSPATH_PREFIX}:${CLASSPATH}
fi

# cygwin 路径转换
if $cygwin; then
  CLASSPATH=`cygpath -p -w "$CLASSPATH"`
  HBASE_HOME=`cygpath -d "$HBASE_HOME"`
  HBASE_LOG_DIR=`cygpath -d "$HBASE_LOG_DIR"`
fi

if [ -d "${HBASE_HOME}/build/native" -o -d "${HBASE_HOME}/lib/native" ]; then
  if [ -z $JAVA_PLATFORM ]; then
    JAVA_PLATFORM=`CLASSPATH=${CLASSPATH} ${JAVA} org.apache.hadoop.util.PlatformName | sed -e "s/ /_/g"`
  fi
  if [ -d "$HBASE_HOME/build/native" ]; then
    JAVA_LIBRARY_PATH=$(append_path "$JAVA_LIBRARY_PATH" "${HBASE_HOME}/build/native/${JAVA_PLATFORM}/lib")
  fi

  if [ -d "${HBASE_HOME}/lib/native" ]; then
    JAVA_LIBRARY_PATH=$(append_path "$JAVA_LIBRARY_PATH" "${HBASE_HOME}/lib/native/${JAVA_PLATFORM}")
  fi
fi

# cygwin 路径转换
if $cygwin; then
  JAVA_LIBRARY_PATH=`cygpath -p "$JAVA_LIBRARY_PATH"`
fi

# 清除循环中的 IFS
unset IFS

# 根据我们所运行的设置正确的 GC 选项
declare -a server_cmds=("master" "regionserver" "thrift" "thrift2" "rest" "avro" "zookeeper")
for cmd in ${server_cmds[@]}; do
	if [[ $cmd == $COMMAND ]]; then
		server=true
		break
	fi
done

if [[ $server ]]; then
	HBASE_OPTS="$HBASE_OPTS $SERVER_GC_OPTS"
else
	HBASE_OPTS="$HBASE_OPTS $CLIENT_GC_OPTS"
fi

if [ "$AUTH_AS_SERVER" == "true" ] || [ "$COMMAND" = "hbck" ]; then
   if [ -n "$HBASE_SERVER_JAAS_OPTS" ]; then
     HBASE_OPTS="$HBASE_OPTS $HBASE_SERVER_JAAS_OPTS"
   else
     HBASE_OPTS="$HBASE_OPTS $HBASE_REGIONSERVER_OPTS"
   fi
fi

# 检测命令是否需要 jline
declare -a jline_cmds=("zkcli" "org.apache.hadoop.hbase.zookeeper.ZKMainServer")
for cmd in "${jline_cmds[@]}"; do
  if [[ $cmd == "$COMMAND" ]]; then
    jline_needed=true
    break
  fi
done

# for jruby
# (1) for the commands which need jruby (see jruby_cmds defined below)
#     A. when JRUBY_HOME is specified explicitly, eg. export JRUBY_HOME=/usr/local/share/jruby
#        CLASSPATH and HBASE_OPTS are updated according to JRUBY_HOME specified
#     B. when JRUBY_HOME is not specified explicitly
#        add jruby packaged with HBase to CLASSPATH
# (2) for other commands, do nothing

# 检测命令是否需要 jruby
declare -a jruby_cmds=("shell" "org.jruby.Main")
for cmd in "${jruby_cmds[@]}"; do
  if [[ $cmd == "$COMMAND" ]]; then
    jruby_needed=true
    break
  fi
done

add_maven_deps_to_classpath() {
  f="${HBASE_HOME}/hbase-build-configuration/target/$1"

  if [ ! -f "${f}" ]; then
      echo "As this is a development environment, we need ${f} to be generated from maven (command: mvn install -DskipTests)"
      exit 1
  fi
  CLASSPATH=${CLASSPATH}:$(cat "${f}")
}

# 添加开发环境类路径的东西
if $in_dev_env; then
  add_maven_deps_to_classpath "cached_classpath.txt"

  if [[ $jline_needed ]]; then
    add_maven_deps_to_classpath "cached_classpath_jline.txt"
  elif [[ $jruby_needed ]]; then
    add_maven_deps_to_classpath "cached_classpath_jruby.txt"
  fi
fi

# 命令需要 jruby
if [[ $jruby_needed ]]; then
  if [ "$JRUBY_HOME" != "" ]; then  # JRUBY_HOME is specified explicitly, eg. export JRUBY_HOME=/usr/local/share/jruby
    # add jruby.jar into CLASSPATH
    CLASSPATH="$JRUBY_HOME/lib/jruby.jar:$CLASSPATH"

    # add jruby to HBASE_OPTS
    HBASE_OPTS="$HBASE_OPTS -Djruby.home=$JRUBY_HOME -Djruby.lib=$JRUBY_HOME/lib"

  else  # JRUBY_HOME is not specified explicitly
    if ! $in_dev_env; then  # not in dev environment
      # add jruby packaged with HBase to CLASSPATH
      JRUBY_PACKAGED_WITH_HBASE="$HBASE_HOME/lib/ruby/*.jar"
      for jruby_jar in $JRUBY_PACKAGED_WITH_HBASE; do
        CLASSPATH=$jruby_jar:$CLASSPATH;
      done
    fi
  fi
fi

# 找出要运行的 class，该脚本可用于直接运行 Java 类
if [ "$COMMAND" = "shell" ] ; then
	#find the hbase ruby sources
  if [ -d "$HBASE_HOME/lib/ruby" ]; then
    HBASE_OPTS="$HBASE_OPTS -Dhbase.ruby.sources=$HBASE_HOME/lib/ruby"
  else
    HBASE_OPTS="$HBASE_OPTS -Dhbase.ruby.sources=$HBASE_HOME/hbase-shell/src/main/ruby"
  fi
  HBASE_OPTS="$HBASE_OPTS $HBASE_SHELL_OPTS"
  CLASS="org.jruby.Main -X+O ${JRUBY_OPTS} ${HBASE_HOME}/bin/hirb.rb"
elif [ "$COMMAND" = "hbck" ] ; then
  # Look for the -j /path/to/HBCK2.jar parameter. Else pass through to hbck.
  case "${1}" in
    -j)
    # Found -j parameter. Add arg to CLASSPATH and set CLASS to HBCK2.
    shift
    JAR="${1}"
    if [ ! -f "${JAR}" ]; then
      echo "${JAR} file not found!"
      echo "Usage: hbase [<options>] hbck -jar /path/to/HBCK2.jar [<args>]"
      exit 1
    fi
    CLASSPATH="${JAR}:${CLASSPATH}";
    CLASS="org.apache.hbase.HBCK2"
    shift # past argument=value
    ;;
    *)
    CLASS='org.apache.hadoop.hbase.util.HBaseFsck'
    ;;
  esac
elif [ "$COMMAND" = "wal" ] ; then
  CLASS='org.apache.hadoop.hbase.wal.WALPrettyPrinter'
elif [ "$COMMAND" = "hfile" ] ; then
  CLASS='org.apache.hadoop.hbase.io.hfile.HFilePrettyPrinter'
elif [ "$COMMAND" = "zkcli" ] ; then
  CLASS="org.apache.hadoop.hbase.zookeeper.ZKMainServer"
  for f in $HBASE_HOME/lib/zkcli/*.jar; do
    CLASSPATH="${CLASSPATH}:$f";
  done
elif [ "$COMMAND" = "upgrade" ] ; then
  echo "This command was used to upgrade to HBase 0.96, it was removed in HBase 2.0.0."
  echo "Please follow the documentation at http://hbase.apache.org/book.html#upgrading."
  exit 1
elif [ "$COMMAND" = "snapshot" ] ; then
  SUBCOMMAND=$1
  shift
  if [ "$SUBCOMMAND" = "create" ] ; then
    CLASS="org.apache.hadoop.hbase.snapshot.CreateSnapshot"
  elif [ "$SUBCOMMAND" = "info" ] ; then
    CLASS="org.apache.hadoop.hbase.snapshot.SnapshotInfo"
  elif [ "$SUBCOMMAND" = "export" ] ; then
    CLASS="org.apache.hadoop.hbase.snapshot.ExportSnapshot"
  else
    echo "Usage: hbase [<options>] snapshot <subcommand> [<args>]"
    echo "$options_string"
    echo ""
    echo "Subcommands:"
    echo "  create          Create a new snapshot of a table"
    echo "  info            Tool for dumping snapshot information"
    echo "  export          Export an existing snapshot"
    exit 1
  fi
elif [ "$COMMAND" = "master" ] ; then
  CLASS='org.apache.hadoop.hbase.master.HMaster'
  if [ "$1" != "stop" ] && [ "$1" != "clear" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_MASTER_OPTS"
  fi
elif [ "$COMMAND" = "regionserver" ] ; then
  CLASS='org.apache.hadoop.hbase.regionserver.HRegionServer'
  if [ "$1" != "stop" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_REGIONSERVER_OPTS"
  fi
elif [ "$COMMAND" = "thrift" ] ; then
  CLASS='org.apache.hadoop.hbase.thrift.ThriftServer'
  if [ "$1" != "stop" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_THRIFT_OPTS"
  fi
elif [ "$COMMAND" = "thrift2" ] ; then
  CLASS='org.apache.hadoop.hbase.thrift2.ThriftServer'
  if [ "$1" != "stop" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_THRIFT_OPTS"
  fi
elif [ "$COMMAND" = "rest" ] ; then
  CLASS='org.apache.hadoop.hbase.rest.RESTServer'
  if [ "$1" != "stop" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_REST_OPTS"
  fi
elif [ "$COMMAND" = "zookeeper" ] ; then
  CLASS='org.apache.hadoop.hbase.zookeeper.HQuorumPeer'
  if [ "$1" != "stop" ] ; then
    HBASE_OPTS="$HBASE_OPTS $HBASE_ZOOKEEPER_OPTS"
  fi
elif [ "$COMMAND" = "clean" ] ; then
  case $1 in
    --cleanZk|--cleanHdfs|--cleanAll)
      matches="yes" ;;
    *) ;;
  esac
  if [ $# -ne 1 -o "$matches" = "" ]; then
    echo "Usage: hbase clean (--cleanZk|--cleanHdfs|--cleanAll)"
    echo "Options: "
    echo "        --cleanZk   cleans hbase related data from zookeeper."
    echo "        --cleanHdfs cleans hbase related data from hdfs."
    echo "        --cleanAll  cleans hbase related data from both zookeeper and hdfs."
    exit 1;
  fi
  "$bin"/hbase-cleanup.sh --config ${HBASE_CONF_DIR} $@
  exit $?
elif [ "$COMMAND" = "mapredcp" ] ; then
  # If we didn't find a jar above, this will just be blank and the
  # check below will then default back to the internal classpath.
  shaded_jar="${shaded_mapreduce}"
  if [ "${INTERNAL_CLASSPATH}" != "true" ] && [ -f "${shaded_jar}" ]; then
    echo -n "${shaded_jar}"
    for f in "${HBASE_HOME}"/lib/client-facing-thirdparty/*.jar; do
      if [[ ! "${f}" =~ ^.*/htrace-core-3.*\.jar$ ]] && \
         [ "${f}" != "htrace-core.jar$" ] && \
         [[ ! "${f}" =~ ^.*/slf4j-log4j.*$ ]]; then
        echo -n ":${f}"
      fi
    done
    echo ""
    exit 0
  fi
  CLASS='org.apache.hadoop.hbase.util.MapreduceDependencyClasspathTool'
elif [ "$COMMAND" = "classpath" ] ; then
  echo "$CLASSPATH"
  exit 0
elif [ "$COMMAND" = "pe" ] ; then
  CLASS='org.apache.hadoop.hbase.PerformanceEvaluation'
  HBASE_OPTS="$HBASE_OPTS $HBASE_PE_OPTS"
elif [ "$COMMAND" = "ltt" ] ; then
  CLASS='org.apache.hadoop.hbase.util.LoadTestTool'
  HBASE_OPTS="$HBASE_OPTS $HBASE_LTT_OPTS"
elif [ "$COMMAND" = "canary" ] ; then
  CLASS='org.apache.hadoop.hbase.tool.CanaryTool'
  HBASE_OPTS="$HBASE_OPTS $HBASE_CANARY_OPTS"
elif [ "$COMMAND" = "version" ] ; then
  CLASS='org.apache.hadoop.hbase.util.VersionInfo'
elif [ "$COMMAND" = "regionsplitter" ] ; then
  CLASS='org.apache.hadoop.hbase.util.RegionSplitter'
elif [ "$COMMAND" = "rowcounter" ] ; then
  CLASS='org.apache.hadoop.hbase.mapreduce.RowCounter'
elif [ "$COMMAND" = "cellcounter" ] ; then
  CLASS='org.apache.hadoop.hbase.mapreduce.CellCounter'
elif [ "$COMMAND" = "pre-upgrade" ] ; then
  CLASS='org.apache.hadoop.hbase.tool.PreUpgradeValidator'
elif [ "$COMMAND" = "completebulkload" ] ; then
  CLASS='org.apache.hadoop.hbase.tool.BulkLoadHFilesTool'
elif [ "$COMMAND" = "hbtop" ] ; then
  CLASS='org.apache.hadoop.hbase.hbtop.HBTop'
  if [ -n "${shaded_jar}" ] ; then
    for f in "${HBASE_HOME}"/lib/hbase-hbtop*.jar; do
      if [ -f "${f}" ]; then
        CLASSPATH="${CLASSPATH}:${f}"
        break
      fi
    done
    for f in "${HBASE_HOME}"/lib/commons-lang3*.jar; do
      if [ -f "${f}" ]; then
        CLASSPATH="${CLASSPATH}:${f}"
        break
      fi
    done
  fi

  if [ -f "${HBASE_HOME}/conf/log4j-hbtop.properties" ] ; then
    HBASE_HBTOP_OPTS="${HBASE_HBTOP_OPTS} -Dlog4j.configuration=file:${HBASE_HOME}/conf/log4j-hbtop.properties"
  fi
  HBASE_OPTS="${HBASE_OPTS} ${HBASE_HBTOP_OPTS}"
else
  CLASS=$COMMAND
fi

# Have JVM dump heap if we run out of memory.  Files will be 'launch directory'
# and are named like the following: java_pid21612.hprof. Apparently it doesn't
# 'cost' to have this flag enabled. Its a 1.6 flag only. See:
# http://blogs.sun.com/alanb/entry/outofmemoryerror_looks_a_bit_better
HBASE_OPTS="$HBASE_OPTS -Dhbase.log.dir=$HBASE_LOG_DIR"
HBASE_OPTS="$HBASE_OPTS -Dhbase.log.file=$HBASE_LOGFILE"
HBASE_OPTS="$HBASE_OPTS -Dhbase.home.dir=$HBASE_HOME"
HBASE_OPTS="$HBASE_OPTS -Dhbase.id.str=$HBASE_IDENT_STRING"
HBASE_OPTS="$HBASE_OPTS -Dhbase.root.logger=${HBASE_ROOT_LOGGER:-INFO,console}"
if [ "x$JAVA_LIBRARY_PATH" != "x" ]; then
  HBASE_OPTS="$HBASE_OPTS -Djava.library.path=$JAVA_LIBRARY_PATH"
  export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$JAVA_LIBRARY_PATH"
fi

# 仅在 master 和 regionserver 上启用安全日志记录
if [ "$COMMAND" = "master" ] || [ "$COMMAND" = "regionserver" ]; then
  HBASE_OPTS="$HBASE_OPTS -Dhbase.security.logger=${HBASE_SECURITY_LOGGER:-INFO,RFAS}"
else
  HBASE_OPTS="$HBASE_OPTS -Dhbase.security.logger=${HBASE_SECURITY_LOGGER:-INFO,NullAppender}"
fi

HEAP_SETTINGS="$JAVA_HEAP_MAX $JAVA_OFFHEAP_MAX"
# 现在，如果我们正在运行命令，则意味着我们需要记录
for f in ${HBASE_HOME}/lib/client-facing-thirdparty/slf4j-log4j*.jar; do
  if [ -f "${f}" ]; then
    CLASSPATH="${CLASSPATH}:${f}"
    break
  fi
done

# 除非设置了 HBASE_NOEXEC，否则执行
export CLASSPATH
if [ "${DEBUG}" = "true" ]; then
  echo "classpath=${CLASSPATH}" >&2
  HBASE_OPTS="${HBASE_OPTS} -Xdiag"
fi

if [ "${HBASE_NOEXEC}" != "" ]; then
  "$JAVA" -Dproc_$COMMAND -XX:OnOutOfMemoryError="kill -9 %p" $HEAP_SETTINGS $HBASE_OPTS $CLASS "$@"
else
  export JVM_PID="$$"
  exec "$JAVA" -Dproc_$COMMAND -XX:OnOutOfMemoryError="kill -9 %p" $HEAP_SETTINGS $HBASE_OPTS $CLASS "$@"
fi
```

-------


## 停止

### stop-hbase.sh

停止 hadoop hbase 守护程序，在主节点上运行以停止整个 HBase 服务。

```shell
# 仿照 $HADOOP_HOME/bin/stop-hbase.sh.

bin=`dirname "${BASH_SOURCE-$0}"`
bin=`cd "$bin">/dev/null; pwd`

# 加载环境变量和参数
. "$bin"/hbase-config.sh
. "$bin"/hbase-common.sh

# 停止命令需要的一些参数
if [ "$HBASE_LOG_DIR" = "" ]; then
  export HBASE_LOG_DIR="$HBASE_HOME/logs"
fi
mkdir -p "$HBASE_LOG_DIR"

if [ "$HBASE_IDENT_STRING" = "" ]; then
  export HBASE_IDENT_STRING="$USER"
fi

export HBASE_LOG_PREFIX=hbase-$HBASE_IDENT_STRING-master-$HOSTNAME
export HBASE_LOGFILE=$HBASE_LOG_PREFIX.log
logout=$HBASE_LOG_DIR/$HBASE_LOG_PREFIX.out  
loglog="${HBASE_LOG_DIR}/${HBASE_LOGFILE}"
pid=${HBASE_PID_DIR:-/tmp}/hbase-$HBASE_IDENT_STRING-master.pid

# 如果 HBase 的相关进程号文件存在，则调用 "$HBASE_HOME"/bin/hbase 停止服务，并记录日志，停止后删除进程号文件，见附 4
if [[ -e $pid ]]; then
  echo -n stopping hbase
  echo "`date` Stopping hbase (via master)" >> $loglog

  nohup nice -n ${HBASE_NICENESS:-0} "$HBASE_HOME"/bin/hbase \
     --config "${HBASE_CONF_DIR}" \
     master stop "$@" > "$logout" 2>&1 < /dev/null &

  waitForProcessEnd `cat $pid` 'stop-master-command'

  rm -f $pid
else
  echo no hbase master found
fi

# 单机模式下停止由 HBase 管理的 Zookeeper 服务，即 HQuorumPeer 进程
distMode=`$bin/hbase --config "$HBASE_CONF_DIR" org.apache.hadoop.hbase.util.HBaseConfTool hbase.cluster.distributed | head -n 1`
if [ "$distMode" == 'true' ] 
then
  "$bin"/hbase-daemons.sh --config "${HBASE_CONF_DIR}" stop zookeeper
fi
```

#### 附 4

`$bin/hbase`接收到 master stop 参数，并经过脚本识别后调用 HMaster 类，进行停止。省略了从 HMaster 到 HMasterCommandLine 的传参过程，前文已经描述过，这里直接从 HMasterCommandLine 中的 stopMaster 方法开始分析。

```java
public class HMasterCommandLine extends ServerCommandLine {
    public int run(String args[]) throws Exception {
        ……
        
        if ("start".equals(command)) {
          return startMaster();
        } else if ("stop".equals(command)) {
          // 匹配到 stop 的指令
          return stopMaster();
        } else if ("clear".equals(command)) {
          return (ZNodeClearer.clear(getConf()) ? 0 : 1);
        } else {
          usage("Invalid command: " + command);
          return 1;
        }
    }
    
    private int stopMaster() {
        // 获取配置文件
        Configuration conf = getConf();
        // 客户端请求失败不再重试
        conf.setInt(HConstants.HBASE_CLIENT_RETRIES_NUMBER, 0);
        // 此处 createConnection 方法通过反射获取一个新的 connection 实例
        try (Connection connection = ConnectionFactory.createConnection(conf)) {
          // 再经过 connection 获得 Admin 实例，Admin 是 HBase 用来管理的 API
          try (Admin admin = connection.getAdmin()) {
            admin.shutdown();
          } catch (Throwable t) {
            LOG.error("Failed to stop master", t);
            return 1;
          }
        } catch (MasterNotRunningException e) {
          LOG.error("Master not running");
          return 1;
        } catch (ZooKeeperConnectionException e) {
          LOG.error("ZooKeeper not available");
          return 1;
        } catch (IOException e) {
          LOG.error("Got IOException: " +e.getMessage(), e);
          return 1;
        }
        // 只有当正确停止后，返回 0
        return 0;
    }
}
```

源码中看到 shutdown 方法和 ShutdownRequest 类等等都是报红的，这是因为 HBase 的某些类和方法是由 protobuf 之类的工具生成的。变量 master 是接口 MasterKeepAliveConnection 的实例，该接口有两个实现类：在 ConnectionImplementation 类中 getKeepAliveMasterService 方法直接返回的内部类 MasterKeepAliveConnection 以及 ShortCircuitMasterConnection。ShortCircuitMasterConnection 是与本地主机通信时可以绕过RPC层（串行化，反序列化，网络等）的短路连接类。

```java
public class HBaseAdmin implements Admin {
    protected MasterKeepAliveConnection master;
    ……
    
    public synchronized void shutdown() throws IOException {
        executeCallable(new MasterCallable<Void>(getConnection(), getRpcControllerFactory()) {
          @Override
          protected Void rpcCall() throws Exception {
            // 设置请求的优先级为高优先级
            setPriority(HConstants.HIGH_QOS);
            master.shutdown(getRpcController(), ShutdownRequest.newBuilder().build());
            return null;
          }
        });
      }
}
```

这里调用分析的是 getKeepAliveMasterService 方法返回的内部类，ShortCircuitMasterConnection 类中的 shutdown 方法也是类似的，通过 MasterProtos 最终调用至实现了 MasterService.BlockingInterface 接口的 MasterRpcServices 类。

```java
class ConnectionImplementation implements ClusterConnection, Closeable {
    ……
    
    private MasterKeepAliveConnection getKeepAliveMasterService() throws IOException {
        ……
        // Ugly delegation just so we can add in a Close method.
        final MasterProtos.MasterService.BlockingInterface stub = this.masterServiceState.stub;
        return new MasterKeepAliveConnection() {
          MasterServiceState mss = masterServiceState;
          ……
          
          @Override
          public MasterProtos.ShutdownResponse shutdown(RpcController controller,
              MasterProtos.ShutdownRequest request) throws ServiceException {
            return stub.shutdown(controller, request);
          }
        }  
    }
}
```

可以看到，在 MasterRpcServices 中，通过实例化的 HMaster 对象，调用的是 shutdown 方法来进行停止。

```java
public class MasterRpcServices extends RSRpcServices
      implements MasterService.BlockingInterface, RegionServerStatusService.BlockingInterface,
        LockService.BlockingInterface, HbckService.BlockingInterface {
    private final HMaster master;
    ……
    
    @Override
    public ShutdownResponse shutdown(RpcController controller,
          ShutdownRequest request) throws ServiceException {
        LOG.info(master.getClientIdAuditPrefix() + " shutdown");
        try {
          master.shutdown();
        } catch (IOException e) {
          LOG.error("Exception occurred in HMaster.shutdown()", e);
          throw new ServiceException(e);
        }
        return ShutdownResponse.newBuilder().build();
      }
}
```

HMaster 会先停止所有的 HRegionServer 服务，然后再停止自身。将 ServerManager 的状态设置为关闭后，RegionServer 将注意到状态的变化，并开始自行关闭，等最后一个 RegionServer 退出后，HMaster 即可关闭。

```java
public class HMaster extends HRegionServer implements MasterServices {
    ……
    
    public void shutdown() throws IOException {
        if (cpHost != null) {
          cpHost.preShutdown();
        }
    
        // 告知 serverManager 关闭集群，serverManager 是用于管理 RegionServer 的
        if (this.serverManager != null) {
          this.serverManager.shutdownCluster();
        }
        // clusterStatusTracker 是用于在 Zookeeper 中对集群设置进行追踪的，这里通过删除 znode 来达到关闭集群的目的
        if (this.clusterStatusTracker != null) {
          try {
            this.clusterStatusTracker.setClusterDown();
          } catch (KeeperException e) {
            LOG.error("ZooKeeper exception trying to set cluster as down in ZK", e);
          }
        }
        // Stop the procedure executor. Will stop any ongoing assign, unassign, server crash etc.,
        // processing so we can go down.
        if (this.procedureExecutor != null) {
          this.procedureExecutor.stop();
        }
        // 关闭集群联机，将杀死可能正在运行的 RPC，如果不关闭连接，将不得不等待 RPC 超时
        if (this.clusterConnection != null) {
          this.clusterConnection.close();
        }
    }

    @Override
    public void stop(String msg) {
        // isStopped 方法继承自 HRegionServer，在其停止后会设置为 false
        if (!isStopped()) {
          // 调用父类 HRegionServer 的 stop 方法挨个进行停止
          super.stop(msg);
          if (this.activeMasterManager != null) {
            this.activeMasterManager.stop();
          }
        }
    }
}
```

接上文，在 ServerManager 中调用 shutdownCluster 方法后又回到 HMaster 中，调用其自身的 stop 方法进行停止。

```java
public class ServerManager {
    private final MasterServices master;
    ……
    
    public void shutdownCluster() {
        String statusStr = "Cluster shutdown requested of master=" + this.master.getServerName();
        LOG.info(statusStr);
        // 设置集群关闭状态
        this.clusterShutdown.set(true);
        if (onlineServers.isEmpty()) {
          // 这里没有使用同步方法可能会导致停止两次，但这没啥问题
          master.stop("OnlineServer=0 right after cluster shutdown set");
        }
      }
}
```

HRegionServer 在接收到子类 HMaster 的 stop 方法调用后，开始停止服务。其 run 方法在开始运行时一直处于自旋状态，将 stopped 变量改为 true 后，会运行后面部分的代码，即停止相关服务。

```java
public class HRegionServer extends HasThread implements
    RegionServerServices, LastSequenceId, ConfigurationObserver {
    
    private volatile boolean stopped = false;
    ……
    
    @Override
    public void stop(final String msg) {
        stop(msg, false, RpcServer.getRequestUser().orElse(null));
    }

    public void stop(final String msg, final boolean force, final User user) {
        if (!this.stopped) {
          LOG.info("***** STOPPING region server '" + this + "' *****");
          if (this.rsHost != null) {
            // when forced via abort don't allow CPs to override
            try {
              this.rsHost.preStop(msg, user);
            } catch (IOException ioe) {
              if (!force) {
                LOG.warn("The region server did not stop", ioe);
                return;
              }
              LOG.warn("Skipping coprocessor exception on preStop() due to forced shutdown", ioe);
            }
          }
          this.stopped = true;
          LOG.info("STOPPED: " + msg);
          // Wakes run() if it is sleeping
          sleeper.skipSleepCycle();
        }
      }
}
```

省略了后续相关服务停止以及 Zookeeper 清理等部分，至此，整个 HMaster 集群已经完全关闭。

-------


## 配置文件

### hbase-env.sh

前面的一些脚本中有加载 hbase-env.sh 中的环境变量，这些变量都是给用户提供的可配置项。
它设置了 HBase 运行中的一些重要 JVM 参数，在对 HBase 进行调优时可能会用到。

文件格式是以`export 环境变量名=变量值`这种形式组织的

* `JAVA_HOME` - JDK 路径，Java 1.8+
  
* `HBASE_CLASSPATH` - 额外的 Java CLASSPATH，可选项

* `HBASE_HEAPSIZE` - 使用的最大堆数量，默认为 JVM 默认值

* `HBASE_OFFHEAPSIZE` - 堆外内存

* `HBASE_OPTS` - 额外的 Java 运行时参数，默认为"-XX:+UseConcMarkSweepGC"，使用 CMS 收集器对年老代进行垃圾收集，CMS 收集器通过多线程并发进行垃圾回收，尽量减少垃圾收集造成的停顿

* `SERVER_GC_OPTS` - 可以为服务器端进程启用 Java 垃圾回收日志记录

* `CLIENT_GC_OPTS` - 为客户端进程启用Java垃圾回收日志记录

* 额外的运行时选项配置，包含 JMX 导出、启用主要 HBase 进程的远程 JDWP 调试等
    *  `HBASE_JMX_BASE`
    *  `HBASE_MASTER_OPTS`
    *  `HBASE_REGIONSERVER_OPTS`
    *  `HBASE_THRIFT_OPTS`
    *  `HBASE_ZOOKEEPER_OPTS`

* `HBASE_REGIONSERVERS` - RegionServer 服务运行节点

* `HBASE_REGIONSERVER_MLOCK` - 是否使所有区域服务器页面都映射为驻留在内存中

* `HBASE_REGIONSERVER_UID` - RegionServer 的用户 ID

* `HBASE_BACKUP_MASTERS` - 备用 Master 节点

* `HBASE_SSH_OPTS` - 额外的 ssh 选项

* `HBASE_LOG_DIR` - HBase 日志存储路径

* `HBASE_IDENT_STRING` - 标识 HBase 实例的字符串，默认为当前用户

* `HBASE_NICENESS` - 守护进程的调度优先级

* `HBASE_PID_DIR` - PID 文件的存储路径，默认是 /tmp，最好换个稳定的路径

* `HBASE_SLAVE_SLEEP` - 在从属命令之间休眠的秒数，默认情况下未设置

* `HBASE_MANAGES_ZK` - 是否启动 HBase 内嵌的 Zookeeper，一般使用集群的 Zookeeper

* `HBASE_ROOT_LOGGER` - HBase 日志级别

* `HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP` - HBase 启动时是否应包含 Hadoop 的库，默认值为 false，表示包含 Hadoop 的库

### hbase-site.xml

该文件配置项较多，此处仅列举一些常见集群配置项，更多参数请移步[官方文档](https://hbase.apache.org/book.html#config.files)。
文件格式是以下面这种形式组织的：
```
<property>
    <name>参数名称</name>
    <value>参数值</value>
</property>
```

* `hbase.tmp.dir` - 本地文件系统上的临时目录，默认为 /tmp/hbase-${user.name}，最好更改此路径为一个更稳定的，否则数据容易丢失。

* `hbase.rootdir` - RegionServers 共享目录，也是 HBase 持久化存储的目录，支持 HDFS 的存储。默认情况下写到 ${hbase.tmp.dir}/hbase 目录，所以最好更改此目录，否则机器重新启动后，所有的数据将丢失。

* `hbase.cluster.distributed` - 集群启动模式，单机模式为 false（默认值），集群模式为 true。如果为 false，将在同一个 JVM 中运行 HBase 以及 Zookeeper 的进程。

* `hbase.zookeeper.quorum` - Zookeeper 服务器列表，以逗号分隔，默认是 127.0.0.1。如果在 hbase-env.sh 中配置了`export HBASE_MANAGES_ZK=true`，那么该 Zookeeper服务将由 HBase 进行管理，作为 HBase 启动/停止的一部分，最好是部署独立的 Zookeeper 集群。

* `hbase.zookeeper.property.dataDir` - Zookeeper 配置文件 zoo.cfg 中的属性，也是快照存储的目录，只有在使用外置的 Zookeeper 集群服务时有效。

* `hbase.master.port` - Master 的内部端口号，默认是 16000。

* `hbase.master.info.port` - Master 的 Web UI 端口号，默认是 16010，如果不想运行 UI 实例，设置为 -1 即可。

* `hbase.regionserver.port` - RegionServer 的内部端口号，默认是 16020。

* `hbase.regionserver.info.port` - RegionServer 的 Web UI 端口号，默认是 16030，如果不想运行 UI 实例，设置为 -1 即可。

* `hbase.regionserver.handler.count` - 在 RegionServer 上的 RPC 监听器实例计数，Master 也使用相同的属性，太多的 handlers 可能会适得其反。将其设置为 CPU 的倍数，如果大多数情况下是只读的，那么接近 CPU 数更好，从 CPU 数的两倍开始进行调整，默认为 30。

* `hbase.regionserver.global.memstore.size` - 在阻止新的更新并强制刷新之前，RegionServer 中所有内存的最大值，默认为堆的 0.4。更新被阻塞并强制刷新，知道一个 RegionServer 中所有内存的大小达到 hbase.regionserver.global.memstore.size.lower.limit，配置中的默认值保留为空。

* `hbase.regionserver.global.memstore.size.lower.limit` - 默认是 hbase.regionserver.global.memstore.size 的 95%，配置中的默认值保留为空。

* `zookeeper.znode.parent` - Zookeeper 中 HBase 的根 Znode 节点，默认是 /hbase。

* `dfs.client.read.shortcircuit` - 设置为 true，则启用本地短路读，默认是 false。

* `hbase.column.max.version` - 新的列簇将使用此值作为默认的版本数，默认是 1。

* `hbase.coprocessor.master.classes` - 以逗号分隔的协处理器列表，在 HMaster 上加载的 MasterObserver 协处理器，指定完整的类名。

* `hbase.coprocessor.region.classes` - 以逗号分隔的协处理器列表，在所有的表上加载，指定完整的类名，也可通过 HTableDescriptor 或 HBase shell 按需加载。

* `hbase.coprocessor.user.region.classes` - 从配置中加载用户表的系统默认协处理器，用户可以继承 HBase 的 RegionCoprocessor 实现自己需要的逻辑部分，指定完整的类名。

* `hbase.coprocessor.user.enabled` - 启用/禁用加载用户的协处理器加载，默认为 true。

* `hbase.coprocessor.enabled` - 启用/禁用加载所有的协处理器加载，默认为 true。