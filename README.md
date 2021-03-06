# ngdns-server

 * 基于 OpenResty [lua-resty-dns-server](https://github.com/vislee/lua-resty-dns-server) dns server

 * 支持 A, AAAA, CNAME, NS, TXT, MX, SRV [DNS记录类型列表](https://zh.wikipedia.org/wiki/DNS%E8%AE%B0%E5%BD%95%E7%B1%BB%E5%9E%8B%E5%88%97%E8%A1%A8)

 * 支持 区域解析 [ngx_stream_ipdb_module](https://github.com/vislee/ngx_stream_ipdb_module), IP地址库: [qqwry.ipdb](https://github.com/metowolf/qqwry.ipdb)

## install openresty

```
# git clone https://github.com/vislee/ngx_stream_ipdb_module.git
# cd ngx_stream_ipdb_module
# git checkout add-lua-api
# cd ..
# wget https://openresty.org/download/openresty-1.15.8.2.tar.gz
# tar zxvf openresty-1.15.8.2.tar.gz
# cd openresty-1.15.8.2
# ./configure --prefix=/usr/local/openresty-dns/ --with-stream --add-module=../ngx_stream_ipdb_module/ --with-cc-opt="-I $PWD/build/ngx_stream_lua*/src"
# gmake -j
# gmake install
```

## install lua-resty-dns-server

https://github.com/vislee/lua-resty-dns-server

```
/usr/local/openresty-dns/bin/opm get vislee/lua-resty-dns-server
```

## install lua-resty-mlcache

https://github.com/thibaultcha/lua-resty-mlcache

```
/usr/local/openresty-dns/bin/opm get thibaultcha/lua-resty-mlcache
```

## install lua-resty-logger-socket

https://github.com/p0pr0ck5/lua-resty-logger-socket

```
/usr/local/openresty-dns/bin/opm get p0pr0ck5/lua-resty-logger-socket
```

## install openresty-dns

```
wget https://raw.githubusercontent.com/selboo/ngdns-server/master/53.lua \
 -O /usr/local/openresty-dns/nginx/conf/53.lua
```

## install qqwry.ipdb

```
wget https://cdn.jsdelivr.net/npm/qqwry.ipdb/qqwry.ipdb \
 -O /usr/local/openresty-dns/nginx/conf/qqwry.ipdb
```

## install redis

```
# yum install redis -y
```

## config nginx.conf

```
#user  nobody;
worker_processes auto;

error_log  logs/error.log ;
pid        logs/nginx.pid;

events {
    worker_connections  1024;
}


stream {

    lua_package_path '/usr/local/openresty-dns/lualib/?.lua;/usr/local/openresty-dns/nginx/conf/?.lua;;';
    lua_package_cpath '/usr/local/openresty-dns/lualib/?.so;;';

    lua_shared_dict QUERYCACHE 32m;

    ipdb /usr/local/openresty-dns/nginx/conf/qqwry.ipdb;
    ipdb_language "CN";

    init_by_lua_block {
        local mlcache = require "resty.mlcache"

        local cache, err = mlcache.new("my_cache", "QUERYCACHE", {
            lru_size = 100000,
            ttl      = 10,
            neg_ttl  = 10,
        })

        _G.cache = cache

        local redis_host = "127.0.0.1"
        local redis_port = 6379

        _G.redis_host = redis_host
        _G.redis_port = redis_port

        local tld = {
            "aikaiyuan.com",
            "selboo.com",
            "abc.com"
        }
        local tlds = table.concat(tld, "|")

        local zone = {
            "(",
            tlds,
            ")$"
        }
        _G.zone = table.concat(zone)

        local DNSTYPES = {}

        DNSTYPES[1]   = "A"
        DNSTYPES[2]   = "NS"
        DNSTYPES[5]   = "CNAME"
        DNSTYPES[6]   = "SOA"
        DNSTYPES[12]  = "PTR"
        DNSTYPES[15]  = "MX"
        DNSTYPES[16]  = "TXT"
        DNSTYPES[28]  = "AAAA"
        DNSTYPES[33]  = "SRV"
        DNSTYPES[99]  = "SPF"
        DNSTYPES[255] = "ANY"

        _G.DNSTYPES = DNSTYPES

        local views = {
            电信    = "DX",
            联通    = "LT",
            移动    = "YD",
            中华电信 = "DX",
            鹏博士   = "PBS",
            教育网   = "JY",
            远传电信 = "DX",
            广电网   = "DX",
            亚太电信 = "DX",
            长城    = "PBS"
        }

        _G.VIEWS = views
    }

    server {
        listen 53 udp ;

        content_by_lua_file conf/53.lua;

        log_by_lua_block {

            local logger = require "resty.logger.socket"
            if not logger.initted() then
                local ok, err = logger.init{
                    host = '127.0.0.1',
                    port = 514,
                    sock_type = "udp",
                    flush_limit = 1,
                    drop_limit = 99999,
                }
                if not ok then
                    ngx.log(ngx.ERR, "failed to initialize the logger: ", err)
                    return
                end
            end

            logger.log(ngx.ctx.log)

        }

    }


}
```

## start redis openresty-dns

```
# systemctl restart redis
# /usr/local/openresty-dns/nginx/sbin/nginx
```

## dns query log

```
2019-06-25 15:59:51 127.0.0.1 8.8.8.8 lb.aikaiyuan.com aikaiyuan.com lb * A aikaiyuan.com/lb/*/A
2019-06-25 15:59:51 127.0.0.1 127.0.0.1 lb.aikaiyuan.com aikaiyuan.com lb * A aikaiyuan.com/lb/*/A
```

> time client_ip subnet_ip domain tld sub view qtype redis_key

## dns view type

View | Code | Region |
---- | ---- | ------ |
电信    | DX | 中国    |
远传电信 | DX | 中国    |
广电网   | DX | 中国    |
亚太电信 | DX | 中国    |
中华电信 | DX | 中国    |
联通    | LT | 中国    |
移动    | YD | 中国    |
教育网   | JY | 中国    |
鹏博士   | PBS | 中国   |
长城    | PBS | 中国    |
海外    |  HW | 海外    |
局域网   | JYW |       |

数据来源: [http://www.cz88.net/ip/](http://www.cz88.net/ip/)

## dns type

#### A

```
## tld/sub/view/type   value/ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com/lb/*/A 220.181.1.1/3600 220.181.2.2/3600 # 默认区域
OK
127.0.0.1:6379> sadd aikaiyuan.com/lb/LT/A 123.125.3.3/3600 # 联通区域
OK
# dig @127.0.0.1 lb.aikaiyuan.com
```

#### CNAME

```
## tld/sub/view/type   value/ttl    set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com/www/*/CNAME   aikaiyuan.appchizi.com./3600  # 默认区域
OK
127.0.0.1:6379> sadd aikaiyuan.com/www/DX/CNAME   dx.appchizi.com./60  # 电信区域
OK
# dig @127.0.0.1 www.aikaiyuan.com CNAME
```

#### AAAA

```
## tld/sub/view/type   value/ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com/ipv6/*/AAAA 240c::6666/60 240c::8888/60
OK
# dig @127.0.0.1 ipv6.aikaiyuan.com AAAA
```

#### NS

```
## tld/sub/view/type   value/ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com/ns/*/NS ns10.aikaiyuan.com./86400
OK
# dig @127.0.0.1 ns.aikaiyuan.com NS
```

#### TXT

```
## tld/sub/view/type   value/ttl   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com/txt/*/TXT txt.aikaiyuan.com/1200
OK
# dig @127.0.0.1 txt.aikaiyuan.com txt
```

#### MX

```
## tld/sub/view/type   value/ttl/preference   set
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com/@/*/MX smtp1.qq.com./720/10 smtp2.qq.com./720/10
OK
# dig @127.0.0.1 aikaiyuan.com MX
```

#### SRV

```
## tld/sub/view/type   priority/weight/port/value/ttl
# redis-cli
127.0.0.1:6379> sadd aikaiyuan.com/srv/*/SRV 1/100/800/www.aikaiyuan.com/120
OK
# dig @127.0.0.1 srv.aikaiyuan.com SRV
```

# TODO

 * [x] edns-client-subnet
 * [ ] ngdns-api

# Thanks

 * https://github.com/vislee/lua-resty-dns-server
 * https://github.com/vislee/ngx_stream_ipdb_module
 * https://github.com/metowolf/qqwry.ipdb
 * https://github.com/thibaultcha/lua-resty-mlcache
 * https://github.com/cloudflare/lua-resty-logger-socket

