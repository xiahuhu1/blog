
版本信息：
>postgresql: 13
> 
> 
Dockerfile文件：
```dockerfile

FROM postgres:13
RUN apt-get update && apt-get install --no-install-recommends -y wget unzip make gcc postgresql-server-dev-13 ca-certificates libhiredis-dev
RUN wget https://github.com/nahanni/rw_redis_fdw/archive/refs/tags/v1.1.zip -O /opt/redis_fdw.zip && \
    cd /opt && unzip redis_fdw && \
    cd /opt/rw_redis_fdw-1.1 && make USE_PGXS=1 && make install USE_PGXS=1
RUN apt-get remove -y wget unzip make gcc postgresql-server-dev-13
RUN rm -rf /var/lib/apt/lists/*
```

生成docker镜像：
```shell
docker build -t 172.16.61.68:8080/library/postgres13-redis_fwd:latest -f Dockerfile .
```
