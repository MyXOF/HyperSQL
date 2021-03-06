# HyperSQL

[![Build Status](https://travis-ci.org/MyXOF/HyperSQL.svg?branch=master)](https://travis-ci.org/MyXOF/HyperSQL)

包含了HyperSQL项目的源码，已经转化为Maven项目，可以用Eclipse或者IDEA直接导入。


## 代码分析

### Server

类**org.hssqldb.server.Server**是启动的入口，运行这个类在控制台里会出现下面的输出:

```shell
[Server@28d93b30]: Startup sequence initiated from main() method
[Server@28d93b30]: Could not load properties from file
[Server@28d93b30]: Using cli/default properties only
[Server@28d93b30]: Initiating startup sequence...
[Server@28d93b30]: Server socket opened successfully in 6 ms.
[Server@28d93b30]: Database [index=0, id=0, db=file:test, alias=] opened successfully in 3233 ms.
[Server@28d93b30]: Startup sequence completed in 3241 ms.
[Server@28d93b30]: 2018-01-31 11:19:44.077 HSQLDB server 2.3.5 is online on port 9001
[Server@28d93b30]: To close normally, connect and execute SHUTDOWN SQL
[Server@28d93b30]: From command line, use [Ctrl]+[C] to abort abruptly
```

输出信息中比较有用的是`[Server@28d93b30]: 2018-01-31 11:19:44.077 HSQLDB server 2.3.5 is online on port 9001` 这一行，这表明服务器现在监听9001端口，之后如果要用JDBC去连接，端口就是9001。

程序内部将启动一个后台线程serverThread，名字是“HSQLDB Server”

```java
public int start() {
  //ignore some codes..

  serverThread = new ServerThread("HSQLDB Server ");

  if (isDaemon) {
    serverThread.setDaemon(true);
  }

  serverThread.start();

  //ignore some codes..

  return previousState;
}
```

在serverThread启动的过程中会打开socket连接，用于处理外部的请求。对于每到来的一个请求，都会创建一个线程进行处理。

```java
private void run() {

  //ignore some codes...

  try {
    // Faster init first:
    // It is huge waste to fully open the databases, only
    // to find that the socket address is already in use
    openServerSocket();
  } catch (Exception e) {
    setServerError(e);
    printError("run()/openServerSocket(): ");
    printStackTrace(e);
    shutdown(true);
    return;
  }
  
  //ignore some codes...

  try {
    /*
     * This loop is necessary for UNIX w/ Sun Java 1.3 because
     * in that case the socket.close() elsewhere will not
     * interrupt this accept().
     */
    while (socket != null) {
      try {
        handleConnection(socket.accept());
      } catch (java.io.InterruptedIOException iioe) {}
    }
  } catch (IOException ioe) {
    if (getState() == ServerConstants.SERVER_STATE_ONLINE) {
      setServerError(ioe);
      printError(this + ".run()/handleConnection(): ");
      printStackTrace(ioe);
    }
  } catch (Throwable t) {
    printWithThread(t.toString());
  } finally {
    shutdown(false);    // or maybe getServerError() != null?
  }
}

public void handleConnection(Socket s) {
 
  Thread   t;
  Runnable r;
  String   ctn;

  //ignore some codes...

  if (serverProtocol == ServerConstants.SC_PROTOCOL_HSQL) {
    r   = new ServerConnection(s, this);
    ctn = ((ServerConnection) r).getConnectionThreadName();
  } else {
    r   = new WebServerConnection(s, (WebServer) this);
    ctn = ((WebServerConnection) r).getConnectionThreadName();
  }

  t = new Thread(serverConnectionThreadGroup, r, ctn);

  t.start();
  printWithThread("handleConnection() exited");
}
```

在ServerConnection类中，最终转到了odbcExecDirect函数里面的session.execute(r)方法对SQL语句进行处理。

```Java
private void odbcExecDirect(String inStatement)  throws RecoverableOdbcFailure, IOException {
  //ignore some codes...

  Result r = Result.newExecuteDirectRequest();

  r.setPrepareOrExecuteProperties(
    statement, 0, 0, StatementTypes.RETURN_COUNT, 0,
    ResultProperties.defaultPropsValue,
    ResultConstants.RETURN_NO_GENERATED_KEYS, null, null);

  Result rOut = session.execute(r);

  //ignore some codes...
}
```

在Session的execute方法里面EXECDIRECT和BATCHEXECDIRECT分别代表单条执行SQL语句和批量执行SQL语句所对应的逻辑。executeXXXStatement方法负责解析SQL语句并执行，performPostExecute方法负责将结果返回。

```Java
case ResultConstants.EXECDIRECT : {
  Result result = executeDirectStatement(cmd);

  result = performPostExecute(cmd, result);
  return result;
}
case ResultConstants.BATCHEXECDIRECT : {
  isBatch = true;

  Result result = executeDirectBatchStatement(cmd);

  result = performPostExecute(cmd, result);
  return result;
}
```

对于SQL解析部分，HyperSQL里面没有采用流行的开源工具antlr而是自己实现了解析，这里不做过多分析。

### JDBC

如果想通过JDBC连接HyperSQL，首先需要在pom文件中加入下面的依赖：

```xml
<!-- https://mvnrepository.com/artifact/org.hsqldb/hsqldb -->
<dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <version>2.4.0</version>
</dependency>
```

之后可以通过标准的SQL语句对HyperSQL进行操作。

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class HyperSQLTest {
	
	public static void main(String[] args) throws SQLException {
		Connection conn = null;
        String sql;

        String url = "jdbc:hsqldb:hsql://127.0.0.1:9001";
        try {

        	Class.forName("org.hsqldb.jdbcDriver");
            System.out.println("成功加载HyperSQL驱动程序");
            conn = DriverManager.getConnection(url);
            Statement stmt = conn.createStatement();
            
            sql = "create table student(NO char(20),name varchar(20),primary key(NO))";
            int result = stmt.executeUpdate(sql);// executeUpdate语句会返回一个受影响的行数，如果返回-1就没有成功
            if (result != -1) {
                System.out.println("创建数据表成功");
                sql = "insert into student(NO,name) values('2012001','123')";
                result = stmt.executeUpdate(sql);
                sql = "insert into student(NO,name) values('2012002','123')";
                result = stmt.executeUpdate(sql);
                sql = "select * from student";
                ResultSet rs = stmt.executeQuery(sql);// executeQuery会返回结果的集合，否则返回空值
                System.out.println("学号\t姓名");
                while (rs.next()) {
                    System.out.println(rs.getString(1) + "\t" + rs.getString(2));// 入如果返回的是int类型可以用getInt()
                }
            }
        } catch (SQLException e) {
            System.out.println("HyperSQL操作错误");
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            conn.close();
        }
	}
}

```













