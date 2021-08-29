---
title: Livy
tags:
  - Livy
categories: Livy
encrypt: 
enc_pwd: 
abbrlink: 3617
date: 2020-05-31 21:47:19
summary_img:
---

# Livy

# 一 使用背景

Apache Spark作为当前最为流行的开源大数据计算框架，广泛应用于数据处理和分析应用，它提供了两种方式来处理数据：一是交互式处理，比如用户使用spark-shell或是pyspark脚本启动Spark应用程序，伴随应用程序启动的同时Spark会在当前终端启动REPL(Read–Eval–Print Loop)来接收用户的代码输入，并将其编译成Spark作业提交到集群上去执行;二是批处理，批处理的程序逻辑由用户实现并编译打包成jar包，spark-submit脚本启动Spark应用程序来执行用户所编写的逻辑，与交互式处理不同的是批处理程序在执行过程中用户没有与Spark进行任何的交互。

两种处理交互方式虽然看起来完全不一样，但是都需要用户登录到Gateway节点上通过脚本启动Spark进程。这样的方式会有什么问题吗?

**首先**将资源的使用和故障发生的可能性集中到了这些Gateway节点。由于所有的Spark进程都是在Gateway节点上启动的，这势必会增加Gateway节点的资源使用负担和故障发生的可能性，同时Gateway节点的故障会带来单点问题，造成Spark程序的失败。

**其次**难以管理、审计以及与已有的权限管理工具的集成。由于Spark采用脚本的方式启动应用程序，因此相比于Web方式少了许多管理、审计的便利性，同时也难以与已有的工具结合，如Apache Knox。

**同时**也将Gateway节点上的部署细节以及配置不可避免地暴露给了登陆用户。

为了避免上述这些问题，同时提供原生Spark已有的处理交互方式，并且为Spark带来其所缺乏的企业级管理、部署和审计功能，本文将介绍一个新的基于Spark的REST服务：Livy。

# 二 介绍

Livy是一个基于Spark的开源REST服务，它能够通过REST的方式将代码片段或是序列化的二进制代码提交到Spark集群中去执行。它提供了以下这些基本功能：

提交Scala、Python或是R代码片段到远端的Spark集群上执行;

提交Java、Scala、Python所编写的Spark作业到远端的Spark集群上执行;

提交批处理应用在集群中运行。

从Livy所提供的基本功能可以看到Livy涵盖了原生Spark所提供的两种处理交互方式。与原生Spark不同的是，所有操作都是通过REST的方式提交到Livy服务端上，再由Livy服务端发送到不同的Spark集群上去执行.

# 三 架构

Livy是一个典型的REST服务架构，它一方面接受并解析用户的REST请求，转换成相应的操作;另一方面它管理着用户所启动的所有Spark集群

用户可以以REST请求的方式通过Livy启动一个新的Spark集群，Livy将每一个启动的Spark集群称之为一个会话(session)，一个会话是由一个完整的Spark集群所构成的，并且通过RPC协议在Spark集群和Livy服务端之间进行通信。根据处理交互方式的不同，Livy将会话分成了两种类型：

交互式会话(interactive session)，这与Spark中的交互式处理相同，交互式会话在其启动后可以接收用户所提交的代码片段，在远端的Spark集群上编译并执行;

批处理会话(batch session)，用户可以通过Livy以批处理的方式启动Spark应用，这样的一个方式在Livy中称之为批处理会话，这与Spark中的批处理是相同的。

可以看到，Livy所提供的核心功能与原生Spark是相同的，它提供了两种不同的会话类型来代替Spark中两类不同的处理交互方式。

# 四 使用postman测试使用

**交互式使用**

## 1 spark

### 1.1 创建session和查看session

使用交互式会话的前提是需要先创建会话。当我们提交请求创建交互式会话时，我们需要指定会话的类型（“kind”），比如“spark”，Livy会根据我们所指定的类型来启动相应的REPL，当前Livy可支持spark、pyspark,sparkr或sql四种不同的交互式会话类型以满足不同语言的需求。

工具采用postman，首先新建一个session（其实就是开启一个spark application）：请求方式post，请求URL为 livy-ip:8998/sessions
请求体类似如下格式：

```json
{
    "kind": "spark",
    "executorMemory":"1G",
	"proxyUser":"livy",
	"queue":"livy",
    "conf" : {
        "spark.cores.max" : 4,
        "spark.dynamicAllocation.enabled":"true",
		"spark.shuffle.service.enabled":"true",
		"spark.dynamicAllocation.minExecutors":0 ,
		"spark.dynamicAllocation.maxExecutors":8 ,
		"spark.dynamicAllocation.executorIdleTimeout":"30s",
		"spark.dynamicAllocation.executorAllocationRatio":0.8
    }
}
```

当创建完会话后，Livy会返回给我们一个JSON格式的数据结构表示当前会话的所有信息：

**ID代表了此会话,所有基于该会话的操作都需要指明其id。**

```json
{
    "id": 47,
    "name": null,
    "appId": null,
    "owner": null,
    "proxyUser": "livy",
    "state": "starting",
    "kind": "spark",
    "appInfo": {
        "driverLogUrl": null,
        "sparkUiUrl": null
    },
    "log": [
        "stdout: ",
        "\nstderr: ",
        "\nYARN Diagnostics: "
    ]
}
```

可以看到上面的"state"为"starting" 从这里看不出这个集群是否完全开启,所以可以使用一下请求查看session的状态

```json
 1  请求方式GET 请求URL为 livy-ip:8998/sessions   可以查看所有的sesssion
返回:
{
    "from": 0,
    "total": 1,
    "sessions": [
        {
            "id": 48,
            "name": null,
            "appId": "application_1584600121035_92196",
            "owner": null,
            "proxyUser": "livy",
            "state": "starting",
            "kind": "spark",
            "appInfo": {
                xxxxx
            },
            "log": [
               xxxxxxx
            ]
        }
    ]
}

2   请求方式GET 请求URL为 livy-ip:8998/sessions/48  (id)  可以查看id 为 47 的session

{
    "id": 48,
    "name": null,
    "appId": "application_1584600121035_92196",
    "owner": null,
    "proxyUser": "livy",
    "state": "idle",
    "kind": "spark",
    "appInfo": {
					xxxxxxxxxxxx
    },
    "log": [
      xxxxxxxxx
    ]
}
```

```
Session State(状态)
状态	               说明
not_started	        Session has not been started
starting	          Session is starting
idle	              Session is waiting for input
busy	              Session is executing a statement
shutting_down	      Session is shutting down
error	              Session errored out
dead	              Session has exited
killed	            Session has been killed
success	            Session is successfully stopped
```

### 1.2 杀死session

请求DELETE   请求URL为 livy-ip:8998/sessions/47   杀死id为47的session

```json
//返回
{
    "msg": "deleted"
}
```

### 1.3 执行语句 和得到结果

```
请求方式post，请求URL为 livy-ip:8998/sessions/{session id}/statements
```

```json
请求体:
{"code":"100+100"}
```

```json
返回值:
{
    "id": 1,
    "code": "100+100",
    "state": "waiting",
    "output": null,
    "progress": 0.0,
    "started": 0,
    "completed": 0
}
```

这个返回的不全,可以通过一下 查看执行结果;

```
GET   请求URL为 livy-ip:8998/sessions/{session id}/statements/{statementId}
```

```JSON
返回结果
{
    "id": 1,
    "code": "100+100",
    "state": "available",
    "output": {
        "status": "ok",
        "execution_count": 1,
        "data": {
            "text/plain": "res0: Int = 200\n"
        }
    },
    "progress": 1.0,
    "started": 1591171274886,
    "completed": 1591171275064
}
```

### 总结

创建session的时候可以指定kind为sql ,在我们进行sql测试中执行,kind为spark时会报错,用的(spark.sql("sql"))是设置参数的时候报错(set xxxx=xx),所以使用的是sql.

# 五 基于Java编写服务

**maven**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>**</artifactId>
        <groupId>**</groupId>
        <version>***</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <properties>
        <scala.version>2.11.8</scala.version>
        <spark.version>2.4.3</spark.version>
        <slf4j.version>1.7.16</slf4j.version>
        <log4j.version>1.2.17</log4j.version>
        <spring.version>4.2.3.RELEASE</spring.version>
    </properties>
    <artifactId>***</artifactId>
    <repositories>
        <repository>
            <id>cloudera</id>
            <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
        </repository>
        <repository>
            <id>maven</id>
            <url>http://central.maven.org/maven2/</url>
        </repository>
        <repository>
            <id>spring.io</id>
            <url>http://repo.spring.io/plugins-release/</url>
        </repository>
    </repositories>


    <dependencies>


        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-jdbc</artifactId>
            <version>1.2.1</version>
            <exclusions>
                <exclusion>
                    <artifactId>slf4j-log4j12</artifactId>
                    <groupId>org.slf4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>18.0</version>
        </dependency>

  

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.41</version>
        </dependency>

        <!-- 日志系统 -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.25</version>
        </dependency>

        <!--slf4j 委托 slf4j2实现-->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-slf4j-impl</artifactId>
            <version>2.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.6.2</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>1.7.25</version>
        </dependency>

        <dependency>
            <groupId>commons-httpclient</groupId>
            <artifactId>commons-httpclient</artifactId>
            <version>3.1</version>
        </dependency>
        <dependency>
            <groupId>com.googlecode.json-simple</groupId>
            <artifactId>json-simple</artifactId>
            <version>1.1</version>
        </dependency>
        <dependency>
            <groupId>commons-cli</groupId>
            <artifactId>commons-cli</artifactId>
            <version>1.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.6.0</version>
        </dependency>
    </dependencies>
    <build>

        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <archive>
                        <manifest>
                            <!--这里要替换成jar包main方法所在类 -->
                            <mainClass>com.lx.dw.job.Runner</mainClass>
                        </manifest>
                        <manifestEntries>
                            <Class-Path>.</Class-Path>
                        </manifestEntries>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id> <!-- this is used for inheritance merges -->
                        <phase>package</phase> <!-- 指定在打包节点执行jar包合并操作 -->
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.4.2</version>
                <configuration>
                    <skipTests>true</skipTests>
                </configuration>
            </plugin>
        </plugins>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
                <filtering>true</filtering>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
                <includes>
                    <include>**/*.xml</include>
                    <include>**/*.properties</include>
                </includes>
                <filtering>true</filtering>
            </resource>
        </resources>
    </build>

</project>
```

代码:

```java
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;


import java.util.Map;
ip
public  class LivyServerUtil {

    private static final String HOST = "ip";
    private static final String PORT = "port";
    private static final String SESSIONS = "sessions";
    private static final String STATEMENTS = "statements";

    private static RestTemplate client = new RestTemplate();
    private static HttpHeaders headers = new HttpHeaders();

    private static final String ERROR_GETRESULT = "读取结果时出错";
    private static final String ERROR_CREATE_SESSION = "创建session时出错";
    private static final String ERROR_CODE = "返回code不是201 or 201";

    /**
     * 向目的URL发送post请求
     * @param host       SPARK_HOME:port
     * @param code    excute Code
     * @return  AdToutiaoJsonTokenData
     */
    public static String sendPost(String host,String port, String code){

        headers.setContentType(MediaType.APPLICATION_JSON);

        String sessionId = getSession(host,port);

        String finalURL = host +":" + port + "/" + SESSIONS +"/" + sessionId +"/"+STATEMENTS;

        HttpEntity<String> entity = new HttpEntity<>(code,headers);
        ResponseEntity<Map> response = client.exchange(finalURL, HttpMethod.POST, entity, Map.class);
        if(response.getStatusCode().toString().equals("201") ||response.getStatusCode().toString().equals("202")){

            String resultBody = response.getBody().toString();
            String curentIndex = resultBody.substring(resultBody.indexOf("=")+1,resultBody.indexOf(","));
            //读取结果
            return getResult(host,port,sessionId,curentIndex);
        }
        return ERROR_CODE;
    }

    private static String getResult(String host,String port,String sessionId,String index){
        int i = 1;

        while( i< 10000){

            String finalURL = host+":"+port+"/"+SESSIONS+"/"+sessionId+"/"+STATEMENTS +"/"+ index;
            ResponseEntity<Map> resp = client.exchange(finalURL, HttpMethod.GET, null, Map.class);

            if(resp.getStatusCode().toString().equals("201")||resp.getStatusCode().toString().equals("200")){
                String resultBody = resp.getBody().toString();
                if(resultBody.toString().contains("state=available")){

                    System.out.println("===============Index : "+i+"====================");
                    return resultBody;
                }
            }else{

                return ERROR_CODE;
            }
            i++;
        }
        return ERROR_GETRESULT;
    }

    private static String getSession(String host,String port){
        ResponseEntity<Map> resp = client.exchange(host+":"+port+SESSIONS+"/", HttpMethod.GET, null, Map.class);
        System.out.println(resp.getBody().toString()+"getSession()======");
        String resultBody = resp.getBody().toString();
        if(resultBody.contains("[]")){
            try {
                if(createSparkSession(HOST,PORT)){
                    ResponseEntity<Map> resp2 = client.exchange(host+":"+port+SESSIONS+"/", HttpMethod.GET, null, Map.class);
                    String resultBody2 = resp2.getBody().toString();
                    String sessionId = resultBody2.substring(resultBody2.indexOf("sessions=[{id=")+14,resultBody2.indexOf(", appId"));
                    return sessionId;
                }else{
                    System.out.println(ERROR_CREATE_SESSION+"======");
                }
            } catch (Exception e) {
                System.out.println(ERROR_CREATE_SESSION);
                e.printStackTrace();

            }
        }
        return resultBody.substring(resultBody.indexOf("[{id=")+5,resultBody.indexOf("[{id=")+6);
    }

    private static boolean createSparkSession(String host,String port) throws Exception{
        HttpEntity<String> entity = new HttpEntity<String>("{\"kind\": \"spark\"}",headers);
        ResponseEntity<Map> resp = client.exchange(host+":"+port+SESSIONS+"/", HttpMethod.POST, entity, Map.class);
        if(resp.getStatusCode().toString().equals("201")||resp.getStatusCode().toString().equals("200"))
            return true;
        return false;
    }

    public static void main(String args[]){
        System.out.println(sendPost(HOST,PORT,"{\"code\":\"100+100\"}"));

    }

}
//原文链接：https://blog.csdn.net/RONE321/article/details/96859232
```



