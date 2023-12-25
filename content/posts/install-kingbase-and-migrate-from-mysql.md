---
title: "Install Kingbase and Migrate From Mysql"
date: 2023-12-25T22:29:01+08:00
draft: false
---

人大金仓的安装配置和数据迁移

# 一、人大金仓的安装和配置

人大金仓是基于 2016.9 月发布的 PostgreSQL 9.6 开发的，很多问题要用这个版本号来查。

部署使用的是官方 docker 镜像，最新版本为 KingbaseES V8。

```sh
# 下载 docker 镜像，最新的地址可通过访问官网获取：https://www.kingbase.com.cn/xzzx/index.htm
# wget https://kingbase.oss-cn-beijing.aliyuncs.com/KESV8R3/V008R006C008B0014-docker/x86_64/kdb_x86_64_v008r006c008b0014.tar

# 官方安装参考文档
# https://help.kingbase.com.cn/v8/install-updata/install-docker/install-docker-3.html

# 加载镜像
docker load -i kdb_x86_64_v008r006c008b0014.tar

# 启动镜像，为了保留数据，将数据挂载到镜像的外面
mkdir -p /mnt/kingbase/data
# 镜像内部使用了 id 为 1000 的用户，这里如果不改的话会有权限问题
chown 1000:1000 /mnt/kingbase/data
docker rm -f kingbase
# 54321 是默认端口
docker run -tid --name kingbase --user 1000:1000 -p 54321:54321 \
    -v /mnt/kingbase/data:/home/kingbase/userdata \
    -e DB_USER=root \
    -e DB_PASSWORD=xxxx \
    kingbase:v1
# 项目启动后，需要手动启动一下，不然无法连接    
docker exec -it kingbase /home/kingbase/install/kingbase/bin/sys_ctl -D /home/kingbase/userdata/data -l logfile start
docker logs -f kingbase
```

现在通过第三方客户端如 navicat，新建 postgres 连接，就可以连上了。

## 安装 GIS 插件

金仓的 GIS 插件需要联系客服，加入 QQ 群获取😶。 也可以用下面网盘里的地址。

```sh
# https://pan.baidu.com/s/11p8n-aFs5Th6hfa6GGuxBA?pwd=axig
# 先下载 GIS 插件压缩包，解压后复制到容器内
docker cp share/ kingbase:/home/kingbase/install/kingbase
docker cp lib/ kingbase:/home/kingbase/install/kingbase
docker cp bin/ kingbase:/home/kingbase/install/kingbase
```

启用 GIS 插件。

```postgresql
-- 启用 GIS 插件，先新建一个数据库，然后打开一个查询窗口
create extension postgis;
create extension postgis_raster;
create extension postgis_sfcgal;
create extension fuzzystrmatch;
create extension postgis_tiger_geocoder;
set exclude_reserved_words = 'level';
create extension postgis_topology;
create extension address_standardizer;
create extension address_standardizer_data_us;
```

# 二、从 MySQL 迁移数据

人大金仓自带了 KDTS 数据迁移工具，不是很多用，有很多错误。最终使用的是 pgloader，自带的类型转换基本上够用了。

```sh
# 为了批量导入多个数据库，这里通过脚本的方式执行，同时方便自定义数据类型转换，不需要的可以把这个 CAST 去掉
# 密码中如果有 @，则要用两个 @ 来表示
# 这里的 CURRENT_DB 是一个变量，由下面那个 sh 脚本在执行的时候传过来
vim all.load 

LOAD DATABASE
        FROM    mysql://root:password@10.100.0.128:3306/'{{CURRENT_DB}}'
        INTO    pgsql://root:password@10.100.0.161:3306/uniseas
        CAST    
                type geometry   to geometry
                ,type point   	to geometry
                ,type polygon   to geometry
;
```

转换脚本

```sh
vim migrate-mysql-postgresql.sh 
#!/bin/bash

export MYSQL_URL='mysql://root:root@10.100.0.128:3306'

# 这里是要批量迁移的数据库列表
database_list=(
    app_upgrade
    nacos_config
    xxl_job
)

rm -f /tmp/pg.log
for i in "${database_list[@]}"; do
    echo process $i
    docker run --rm --name pgloader -it \
        -e CURRENT_DB=$i \
        -v $PWD:/data ghcr.io/dimitri/pgloader:latest \
        pgloader /data/all.load >> /tmp/pg.log
done

echo done
```

# 三、整合 geometry 类型到 mybatis

因为库中保存的是 WKB 格式的数据，所以在保存和查询的时候，都要转换一下。这里主要是通过集成 BaseTypeHandler 类来实现的。

```java
@Component
public class GeometryTypeHandler<T extends Geometry> extends BaseTypeHandler<T> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
        // 这里仅作示例，实际上 wktWritter 需要用 ThreadLocal 包装一下
        WKTWriter wktWriter = new WKTWriter();
        String wkt = wktWriter.write(parameter);
        ps.setObject(i, wkt);
    }

    @SneakyThrows
    @Override
    public T getNullableResult(ResultSet rs, String columnName) throws SQLException {
            KBobject kBobject = (KBobject) rs.getObject(columnName);
            String value = kBobject.getValue();
            WKBReader reader = new WKBReader();
            return (T) reader.read(WKBReader.hexToBytes(value));
    }

    @SneakyThrows
    @Override
    public T getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        KBobject kBobject = (KBobject) rs.getObject(columnIndex);
        String value = kBobject.getValue();
        WKBReader reader = new WKBReader();
        return (T) reader.read(WKBReader.hexToBytes(value));
    }

    @SneakyThrows
    @Override
    public T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        KBobject kBobject = (KBobject) cs.getObject(columnIndex);
        String value = kBobject.getValue();
        WKBReader reader = new WKBReader();
        return (T) reader.read(WKBReader.hexToBytes(value));
    }

}
```

