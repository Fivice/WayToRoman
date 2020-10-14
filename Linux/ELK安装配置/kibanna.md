# 使docker compose安装kibana

## docker 以及docker compose安装准备

参考 `Docker` 官网完成安装

## kibana:6.8.12 使用的docker-compose.yml文件内容

kibana版本需要与我们安装的elasticsearch版本一致

```yml
version: '2'
services:
  kibana:
    image: docker.elastic.co/kibana/kibana:6.8.12
    environment:
      SERVER_NAME: kibana
      SERVER_HOST: "0.0.0.0"
      ELASTICSEARCH_HOSTS: http://223.247.135.249:9200
      I18N_LOCALE: zh-CN
    ports:
      - 5601:5601
```
