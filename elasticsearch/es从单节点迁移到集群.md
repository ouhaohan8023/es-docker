## elasticsearch 从单节点迁移到集群

### 前提
1. 使用docker
2. es00: 单节点
3. es-cluster: es集群，其中包括 es01, es02, es03

### 步骤
#### 1. 先配置es01
> 目前es01即是master节点，又是data节点
> docker-compose up
> 启动docker，会自动创建 esnet 网络，并启动es
> 先确保es01能成功启动
```yaml
version: '3'
services:
  es01:
    image: elasticsearch:6.8.11
    container_name: es01
    environment:
      - cluster.name=es-cluster
      - node.name=es01
      - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3
      - discovery.zen.minimum_master_nodes=1 #设置集群中的最小主节点数
      - node.master=true
      - node.data=true
      - gateway.recover_after_nodes=1 #用于设置在多少个节点启动之后，Elasticsearch 会开始自动恢复数据。
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "xpack.security.enabled=false"   # 开启xpack安全认证
      - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:                                  # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "./es01/data:/usr/share/elasticsearch/data"
      - "./es01/logs:/usr/share/elasticsearch/logs"
      - "./es01/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      esnet:
        ipv4_address: 192.168.3.1
networks:
  esnet:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.3.0/24
          gateway: 192.168.3.10
```

#### 2. 配置es02，es03
> 将es00的data数据复制到es01对应目录下，es02，es03保持默认设置（如果把data复制到es02，es03，会导致无法启动） 
> 三个节点都启动后，访问 /_cat/nodes?v&pretty 查看是否构成集群
> 
```yaml
version: '3'
services:
  es01:
    image: elasticsearch:6.8.11
    container_name: es01
    environment:
      - cluster.name=es-cluster
      - node.name=es01
      - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3
      - discovery.zen.minimum_master_nodes=1
      - node.master=true
      - node.data=true
      - gateway.recover_after_nodes=1
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "xpack.security.enabled=false"   # 开启xpack安全认证
      - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:                                  # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "./es01/data:/usr/share/elasticsearch/data"
      - "./es01/logs:/usr/share/elasticsearch/logs"
      - "./es01/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      esnet:
        ipv4_address: 192.168.3.1
    
  es02:
    image: elasticsearch:6.8.11
    container_name: es02
    environment:
      - cluster.name=es-cluster
      - node.name=es02
      - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3
      - discovery.zen.minimum_master_nodes=1
      - node.master=false
      - node.data=true
      - gateway.recover_after_nodes=1
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "xpack.security.enabled=false"   # 开启xpack安全认证
      - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - "./es02/data:/usr/share/elasticsearch/data"
      - "./es02/logs:/usr/share/elasticsearch/logs"
      - "./es02/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
    ports:
      - 9201:9200
      - 9301:9300
    networks:
      esnet:
        ipv4_address: 192.168.3.2

  es03:
    image: elasticsearch:6.8.11
    container_name: es03
    environment:
      - cluster.name=es-cluster
      - node.name=es03
      - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3
      - discovery.zen.minimum_master_nodes=1
      - node.master=false
      - node.data=true
      - gateway.recover_after_nodes=1
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "xpack.security.enabled=false"   # 开启xpack安全认证
      - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - "./es03/data:/usr/share/elasticsearch/data"
      - "./es03/logs:/usr/share/elasticsearch/logs"
      - "./es03/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
    ports:
      - 9202:9200
      - 9302:9300
    networks:
      esnet:
        ipv4_address: 192.168.3.3

networks:
  esnet:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.3.0/24
          gateway: 192.168.3.10
```

#### 3. 将es01中的数据完全复制到es02，es03
> 使用elasticsearch-head可视化查看
> 也可使用 /_cat/shards?v 查看分片是否复制
```
PUT es01:9200/_settings
{
    "index" : {
        "number_of_replicas" : 2
    }
}
```

#### 4. 关停节点，将es01设置为单master，删除data数据
> 这一步做完，在启动后，es01中无分片，之前es01的分片变为 Unassigned 状态

#### 5. 重启集群，将 number_of_replicas 重新设置为0或1
> 这一步做完，集群状态会重新回归 green
```
PUT es01:9200/_settings
{
    "index" : {
        "number_of_replicas" : 0
    }
}
```

#### 6. 配置ssl证书
```shell
# 在ES的根目录生成CA证书。 中间需要设置密码，直接回车可以不设置（慎重考虑）
1. elasticsearch-certutil ca
# 使用第一步生成的证书，产生p12密钥
2. elasticsearch-certutil cert --ca elastic-stack-ca.p12
3. 在config目录创建certs目录
4. 拷贝p12文件至certs目录
5. 修改elasticsearch.yml
6. 每个节点的elasticsearch.yml都需要配置
7. 需要在docker-compose中配置文件映射
8. 要注意certs文件夹及目录下文件的权限,如果权限不满足,会出现异常 failed to load plugin class [org.elasticsearch.xpack.core.XPackPlugin]
```

```yaml
# 开启安全控制
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: ./certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: ./certs/elastic-certificates.p12
```

#### 最终版 docker-compose.yml
```yaml
version: '3'
services:
  es01:
    image: elasticsearch:6.8.11
    container_name: es01
    environment:
      - cluster.name=es-cluster
      - node.name=es01
      - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3,192.168.3.4
      - discovery.zen.minimum_master_nodes=1
      - node.master=true
      - node.data=false
      - gateway.recover_after_nodes=1
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:                                  # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "./es01/data:/usr/share/elasticsearch/data"
      - "./es01/logs:/usr/share/elasticsearch/logs"
      - "./es01/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
      - "./es01/config/certs:/usr/share/elasticsearch/config/certs"
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      esnet:
        ipv4_address: 192.168.3.1
    
  es02:
    image: elasticsearch:6.8.11
    container_name: es02
    environment:
      - cluster.name=es-cluster
      - node.name=es02
      - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3,192.168.3.4
      - discovery.zen.minimum_master_nodes=1
      - node.master=false
      - node.data=true
      - gateway.recover_after_nodes=1
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - "./es02/data:/usr/share/elasticsearch/data"
      - "./es02/logs:/usr/share/elasticsearch/logs"
      - "./es02/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
      - "./es02/config/certs:/usr/share/elasticsearch/config/certs"
    ports:
      - 9201:9200
      - 9301:9300
    networks:
      esnet:
        ipv4_address: 192.168.3.2

  es03:
    image: elasticsearch:6.8.11
    container_name: es03
    environment:
      - cluster.name=es-cluster
      - node.name=es03
      - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3,192.168.3.4
      - discovery.zen.minimum_master_nodes=1
      - node.master=false
      - node.data=true
      - gateway.recover_after_nodes=1
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - "./es03/data:/usr/share/elasticsearch/data"
      - "./es03/logs:/usr/share/elasticsearch/logs"
      - "./es03/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
      - "./es03/config/certs:/usr/share/elasticsearch/config/certs"
    ports:
      - 9202:9200
      - 9302:9300
    networks:
      esnet:
        ipv4_address: 192.168.3.3
        
#   es04:
#     image: elasticsearch:6.8.11
#     container_name: es04
#     environment:
#       - cluster.name=es-cluster
#       - node.name=es04
#       - discovery.zen.ping.unicast.hosts=192.168.3.1,192.168.3.2,192.168.3.3,192.168.3.4
#       - discovery.zen.minimum_master_nodes=1
#       - node.master=false
#       - node.data=true
#       - gateway.recover_after_nodes=1
#       - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
#       - "xpack.security.enabled=false"   # 开启xpack安全认证
#       - "ELASTIC_PASSWORD=PASSWORD" # 设置密码
#     ulimits:
#       memlock:
#         soft: -1
#         hard: -1
#     volumes:
#       - "./es04/data:/usr/share/elasticsearch/data"
#       - "./es04/logs:/usr/share/elasticsearch/logs"
#       - "./es04/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml"
#     ports:
#       - 9203:9200
#       - 9303:9300
#     networks:
#       esnet:
#         ipv4_address: 192.168.3.4

networks:
  esnet:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.3.0/24
          gateway: 192.168.3.10
```

#### 最终es01/config/elasticsearch.yml
```yaml
cluster.name: "es-cluster"
network.host: 192.168.3.1
http.port: 9200
# 开启es跨域
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization
# 开启安全控制
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: ./certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: ./certs/elastic-certificates.p12
```

#### 其他
1. 查看集群内各节点存储情况
> http://ip:9200/_cat/allocation?v&pretty
2. 查看分片分布情况
> http://ip:9201/_cat/shards?v
3. 
