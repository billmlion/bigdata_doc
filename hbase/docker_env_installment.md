# 1.Docker 安装 HBase

## 1）搜索 HBase 镜像

```
docker search hbase
```

![image](DA808ED7FDD449848558B3F90EAA4DCB)

## 2）下载 HBase 镜像

```
docker pull harisekhon/hbase:1.3
```

## 3）下载好之后查看镜像信息

```
docker images
```
![image](E3635DC6DC8A42F9987C3452492EFEDB)

## 4）运行 hbase 镜像


设置主机名

```
hostname hbase-docker
```
![image](11FEEF28FDF34768ACB7CB92955E479F)

```
docker run -d -h hbase-docker -p 2181:2181 -p 9090:9090 -p 9095:9095 -p 16000:16000  -p 16020:16020  -p 16010:16010 -p 16201:16201 -p 16301:16301 --name hbase1.3 harisekhon/hbase:1.3
```

> **Note**
>
> "-p 16020:16020" 需要暴露此端口，容器指定該端口為hbase.regionserver.port,如果沒有，會報訪問拒絕問題
**==（该问题折腾了一会，哈哈，天下文章一般抄，如果把该端口删除映射，可复现此问题）==**

## 5）登录 HBase web页面

![image](BED3EB2276A64151BECFBDAD6BC3826C)





# 2.操作 HBase

> 上面我们通过 docker 安装好了 HBase 服务

## 1）进入 HBase 镜像

```
docker exec -it hbase1.3 bash
```
![image](4730C46B8C2A4AB28A93D6C40532D726)

##  2）进入 HBase 的 bin 目录
![image](1EE218868D3742F3A9A265F5E7247DF1)

## 3）进入 HBase 客户端
![image](799A12E697C844239A6679E430BB4BB6)


# 3.intellij idea下 java api访问

## 1）配置maven依赖
```
 <dependencies>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.3.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-server</artifactId>
            <version>1.3.2</version>
        </dependency>
```

## 2）Java代码大集合

```
package com.atguigu.test;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.*;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;


public class HbaseTest {

    private static Admin admin;

    private static final String COLUMNS_FAMILY_1 = "cf1";
    private static final String COLUMNS_FAMILY_2 = "cf2";

    public static Connection initHbase() throws IOException {
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hbase-docker");
        configuration.set("hbase.zookeeper.property.clientPort", "2181");
//        configuration.set("hbase.master", "127.0.0.1:60000");
        configuration.set("hbase.master", "hbase-docker:16000");
        configuration.setInt("hbase.regionserver.port", 16020);
        Connection connection = ConnectionFactory.createConnection(configuration);
        return connection;
    }

    //创建表 create
    public static void createTable(TableName tableName, String[] cols) throws IOException {
        admin = initHbase().getAdmin();
        if (admin.tableExists(tableName)) {
            System.out.println("Table Already Exists！");
        } else {
            HTableDescriptor hTableDescriptor = new HTableDescriptor(tableName);
            for (String col : cols) {
                HColumnDescriptor hColumnDescriptor = new HColumnDescriptor(col);
                hTableDescriptor.addFamily(hColumnDescriptor);
            }
            admin.createTable(hTableDescriptor);
            System.out.println("Table Create Successful");
        }
    }

    public static TableName getTbName(String tableName) {
        return TableName.valueOf(tableName);
    }

    // 删除表 drop
    public static void deleteTable(TableName tableName) throws IOException {
        admin = initHbase().getAdmin();
        if (admin.tableExists(tableName)) {
            admin.disableTable(tableName);
            admin.deleteTable(tableName);
            System.out.println("Table Delete Successful");
        } else {
            System.out.println("Table does not exist!");
        }

    }

    //put 插入数据
    public static void insertData(TableName tableName, Student student) throws IOException {
        Put put = new Put(Bytes.toBytes(student.getId()));
        put.addColumn(Bytes.toBytes(COLUMNS_FAMILY_1), Bytes.toBytes("name"), Bytes.toBytes(student.getName()));
        put.addColumn(Bytes.toBytes(COLUMNS_FAMILY_1), Bytes.toBytes("age"), Bytes.toBytes(student.getAge()));
        initHbase().getTable(tableName).put(put);
        System.out.println("Data insert success:" + student.toString());
    }

    // delete 删除数据
    public static void deleteData(TableName tableName, String rowKey) throws IOException {
        Delete delete = new Delete(Bytes.toBytes(rowKey));      // 指定rowKey
//        delete = delete.addColumn(Bytes.toBytes(COLUMNS_FAMILY_1), Bytes.toBytes("name"));  // 指定column，也可以不指定，删除该rowKey的所有column
        initHbase().getTable(tableName).delete(delete);
        System.out.println("Delete Success");
    }

    // scan数据
    public static List<Student> allScan(TableName tableName) throws IOException {
        ResultScanner results = initHbase().getTable(tableName).getScanner(new Scan().addFamily(Bytes.toBytes("cf1")));
        List<String> list = new ArrayList<>();
        for (Result result : results) {
            Student student = new Student();
            for (Cell cell : result.rawCells()) {
                String colName = Bytes.toString(cell.getQualifierArray(), cell.getQualifierOffset(), cell.getQualifierLength());
                String value = Bytes.toString(cell.getValueArray(), cell.getValueOffset(), cell.getValueLength());
            }

        }
        return null;
    }

    // 根据rowkey get数据
    public static Student singleGet(TableName tableName, String rowKey) throws IOException {
        Student student = new Student();
        student.setId(rowKey);
        Get get = new Get(Bytes.toBytes(rowKey));
        if (!get.isCheckExistenceOnly()) {
            Result result = initHbase().getTable(tableName).get(get);
            for (Cell cell : result.rawCells()) {
                String colName = Bytes.toString(cell.getQualifierArray(), cell.getQualifierOffset(), cell.getQualifierLength());
                String value = Bytes.toString(cell.getValueArray(), cell.getValueOffset(), cell.getValueLength());
                switch (colName) {
                    case "name":
                        student.setName(value);
                        break;
                    case "age":
                        student.setAge(value);
                        break;
                    default:
                        System.out.println("unknown columns");
                }

            }

        }

        System.out.println(student.toString());
        return student;
    }

    // 查询指定Cell数据
    public static String getCell(TableName tableName, String rowKey, String cf, String column) throws IOException {
        Get get = new Get(Bytes.toBytes(rowKey));
        String rst = null;
        if (!get.isCheckExistenceOnly()) {
            get = get.addColumn(Bytes.toBytes(cf), Bytes.toBytes(column));
            try {

                Result result = initHbase().getTable(tableName).get(get);
                byte[] resByte = result.getValue(Bytes.toBytes(cf), Bytes.toBytes(column));
                rst = Bytes.toString(resByte);
            } catch (Exception exception) {
                System.out.printf("columnFamily or column does not exists");
            }

        }
        System.out.println("Value is: " + rst);
        return rst;
    }


    public static void main(String[] args) throws IOException {
        Student student = new Student();
        student.setId("1");
        student.setName("Arvin");
        student.setAge("18");
        String table = "student";

//        createTable(getTbName(table), new String[]{COLUMNS_FAMILY_1, COLUMNS_FAMILY_2});
//        deleteTable(getTbName(table));
//        insertData(getTbName(table), student);
//        deleteData(getTbName(table), "1");
//        singleGet(getTbName(table), "2");
        getCell(getTbName(table), "2", "cf1", "name");
    }

}

```




```
package com.atguigu.test;

public class Student {
    private String id;
    private String name;
    private String age;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Student{" +
                "id='" + id + '\'' +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }

}

```


# 4.issue
## 4.1）本地hadoop lib 导入问题
![image](1D99E0A920E248D3A677FF85CFF79601)

测试发现winutils在环境变量设置不成功
![image](89D73BB9382C4D559CE98460D0528568)

设置hadoop环境变量
![image](853B616CFF0D48BA8B983B73DAF0EA6E)

设置idea terminal环境
![image](636B5D8D7C5543B980347B6401E4EE49)

如果设置hadoop环境变量后，cmd测试可以，但是idea任然导入不成功，此时需要使用 管理员身份打开idea