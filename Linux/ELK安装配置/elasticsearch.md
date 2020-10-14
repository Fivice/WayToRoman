# 使docker compose安装elasticsearch

## docker 以及docker compose安装准备

参考 `Docker` 官网完成安装

## elasticsearch 6.8.12 使用的docker-compose.yml文件内容

```yml
version: '2.2'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.8.12
    container_name: elasticsearch
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - http.cors.enabled=true
      - http.cors.allow-origin="*"
      - http.cors.allow-methods=OPTIONS, HEAD, GET, POST, PUT, DELETE
      - http.cors.allow-headers="X-Requested-With, Content-Type, Content-Length, X-User"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  elasticsearch2:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.8.12
    container_name: elasticsearch2
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elasticsearch"
      - http.cors.enabled=true
      - http.cors.allow-origin="*"
      - http.cors.allow-methods=OPTIONS, HEAD, GET, POST, PUT, DELETE
      - http.cors.allow-headers="X-Requested-With, Content-Type, Content-Length, X-User"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata2:/usr/share/elasticsearch/data
    networks:
      - esnet
volumes:
  esdata1:
    driver: local
  esdata2:
    driver: local

networks:
  esnet:
    driver: bridge
```

使用docker compose 安装elasticsearch的ik分词器.

```bash
docker-compose exec elasticsearch elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.8.12/elasticsearch-analysis-ik-6.8.12.zip
```

安装后重启容器

```bash
docker-compose restart elasticsearch
```
