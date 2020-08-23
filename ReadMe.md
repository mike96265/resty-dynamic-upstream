### 从 nginx 到 openresty + consul, 服务注册与发现

背景: 在 Application 逐渐复杂的过程中, 将 Application 切分为 粒度更细的 Application，利用 nginx 代理到不同的服务。但是管理的复杂度逐渐增大，各个服务之间的互相调用包括 nginx upstream 的维护变得比较困难。

- consul 作为服务注册中心，各个 Application 启动时自动将自己注册到 consul，退出时注销，consul 有内建的健康检查，所以在 Application 出现异常时，也能够及时的注销服务。
- openresty 作为流量入口，因为 openresty 与 nginx 完全兼容，所以不需要修改 nginx 代理， 负载均衡的逻辑，而通过 consul 的 watch api，可以实现我们在服务部署，服务扩容时的热更新。

#### consul

consul 是一个功能比较完备的服务注册中心，更多详细的描述可以自行搜索

如果想根据本文的思路搭建一个简单的demo，你可以通过 docker，或者 podman 搭建集群使用，可以参考 https://hub.docker.com/_/consul

```bash
# server，如果是 podman则需要用必须使用 sudo
sudo docker run -d --name=dev-consul -e CONSUL_BIND_INTERFACE=eth0 consul

# 加入两个节点，ip地址你可以通过获得
sudo docker inspect dev-consul | grep IP
sudo docker run -d -e CONSUL_BIND_INTERFACE=eth0 consul agent -dev -join=10.88.0.4
sudo docker run -d -e CONSUL_BIND_INTERFACE=eth0 consul agent -dev -join=10.88.0.4
sudo docker run -d --net=host -e 'CONSUL_LOCAL_CONFIG={"leave_on_terminate": true}' consul agent -bind={绑定的ip} -retry-join=10.88.0.4

```

#### openresty 

openresty 需要加入第三方模块来支持动态修改 upstream

- https://openresty.org/download/openresty-1.17.8.2.tar.gz
- https://github.com/ZigzagAK/ngx_dynamic_upstream.git 
- https://github.com/ZigzagAK/ngx_dynamic_upstream_lua.git
- https://github.com/ZigzagAK/ngx_dynamic_healthcheck.git

```bash
tar -xvzf openresty-1.17.8.2.tgz
cd openresty-1.17.8.2
mkdir third_party
cd third_party
git clone https://github.com/ZigzagAK/ngx_dynamic_upstream.git
git clone https://github.com/ZigzagAK/ngx_dynamic_upstream_lua.git
git clone https://github.com/ZigzagAK/ngx_dynamic_healthcheck.git
cd ..
```

> 需要给 openresty 和 ngx_dynamic_upstream_lua 打上补丁，相关的 issue 可以在这里看 https://github.com/openresty/stream-lua-nginx-module/issues/180，补丁文件在仓库，你可以自行下载到对应的目录

```bash
# 在 openresty 解压之后的目录
patch -p0 < ngx_stream_lua_api.patch
# 在 third_party/ngx_dynamic_upstream_lua 目录
git am -s < 0001-uninclude-ngx_stream_lua_request.h.patch
```



如果你的项目结构和我一样，你可以用下面的命令编译 openresty，你可以用 -j 指定 make 时的线程数，--prefix 指定安装的目标目录

```bash
./configure -jN --prefix=/home/luvjoey/Projects/openresty-bin --add-module=./third_party/ngx_dynamic_upstream --add-module=ngx_dynamic_upstream_lua --add-module=ngx_dynamic_healthcheck
make -jN
make install 
```

如果你修改了 prefix，你可能还需要修改 PATH 环境变量，进入 prefix 指定目录下的 bin 目录。执行

```bash
export PATH=`pwd`:$PATH:
```

你现在应该已经有了一个 consul 集群，和 编译好的 openresty

你可以用 python-consul2，和 flask 搭建一个简单的服务，注册到 consul 上，你可以修改 port 来让多个服务注册上去

```python
from flask import Flask
from consul import Consul, Check

consul = Consul(
    host='127.0.0.1',
    port=8500
)
# 服务的 ip 和 端口
ip = '10.88.0.1'
port = 3000

consul.agent.service.register(
    name='star',
    service_id='star-{}'.format(port),
    address=ip,
    port=port,
    check=Check.http(
        url='http://ip:{}/health-check'.format(ip, port),
        interval='1s',
        timeout='5s',
        deregister='5s'
    )
)

app = Flask('star')


@app.route('/hello-world')
def hello_world():
    return "hello world from {}:{}".format(ip, port)


@app.route('/health-check')
def health_check():
    return "alive"


if __name__ == '__main__':
    app.run(host=ip, port=port)
```

当你运行起来之后，你可以在浏览器打开 consul 的数据面板，服务的 ip 地址是 你执行 `docker inspect dev-consul | grep IP` 得到的地址， 端口默认是 8500，你会看到大概这样一个界面:

![2020-08-23 12-37-26屏幕截图.png](https://i.loli.net/2020/08/23/N6P4y75RfAublFH.png)

然后启动 新建一个 openresty 项目

```bash
mkdir service-auto-load
cd service-auto-load
opm --cwd install hamishforbes/lua-resty-consul
# 新建 nginx.conf 文件，粘贴文末的配置
mkdir logs
openresty -p `pwd` -c nginx.conf
```

关于配置文件

- 定义了一个 叫做 dynamic_service 的 upstream，
- 创建定时器，利用 consul 的 watch api，`passing = true` 过滤得到健康节点，index 是 consul 提供的 blocking query 的 概念，也即 watch。然后更新本地 uupstream 的 peers，来实现动态更新的功能。
- curl localhost:8080/api，curl localhost:8080/consul

一些想法

- **balancer_by_lua** vs **dynamic upstream** ?

我觉得目前的 balancer_by_lua 有点名不符实，因为跟负载均衡的关系不大。虽然已经有用 lua 实现的轮询和一致性哈希，感觉在健壮性上，还比不上 nginx 提供的负载均衡算法，所以我选择 dynamic upstream 而不是 balancer_by_lua

- `if ngx.worker.id() == 0`

因为 dynamic upstream 只支持 配置了 zone 指令的 upstream 块，也就意味着这一块内存是 worker 共享的，所以只需要一个 worker 在更新就好了。

- 为什么有一个 set_by_lua

在 balancer_by_lua 的 情况下，可以更灵活的选择应用服务器，在应用 dynamic upstream 时，也可以尝试，在配置文件中针对业务需要的灰度发布，预定义多个 upstream，可以 watch tag 不同的 server group，然后通过 set_by_lua，实现更灵活的 load balancer

- server 0.0.0.1 down

占位符

- lua_package_path "resty_modules/lualib/?.lua;;";

在使用 opm 包管理器的时候，使用 --cwd 可以避免权限问题，或者污染全局环境

- ngx.timer.at(1, refresh_service)

在服务更新时，比如三个 Application 一起启动，有可能会短时间内触发三次事件，所以适当的 sleep，等待批量更新完成。

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

worker_processes 1;

http {

    lua_package_path "resty_modules/lualib/?.lua;;";

    upstream dynamic_service {
        keepalive 20;
        zone service_zone 1m;
        server 0.0.0.1 down;
    }

    init_worker_by_lua_block {
        local du = require "ngx.dynamic_upstream"
        local resty_consul = require "resty.consul"
        local json = require "cjson"

        if ngx.worker.id() == 0 then
            local index = {}
            consul = resty_consul:new({
                host="127.0.0.1",
                port=8500,
                read_timeout=(60*1000)
            })

            local function hash_diff(t1, t2)

                if type(t1) ~= 'table' or type(t2) ~= 'table' then
                    return false, nil, nil, "param must be lua table."
                end

                local res1 = {}
                local res2 = {}

                for key, _ in pairs(t1) do
                    if t2[key] == nil then
                        res1[key] = true
                    end
                end

                for key, _ in pairs(t2) do
                    if t1[key] == nil then
                        res2[key] = true
                    end
                end

                return true, res1, res2, ''
            end

            local function refresh_service()
                ngx.log(ngx.ERR, 'start refresh service')
                local exists = {}
                local realtime = {}

                -- get primary peers from nginx, store in exists set
                local ok, peers, err = du.get_primary_peers('dynamic_service')

                if ok then
                    for _, peer in ipairs(peers) do
                        exists[peer['server']] = true
                    end
                end

                -- get active peers from consul, store in realtime set
                local idx = index['dynamic_service'] or 0
                local res, err = consul:get("/health/service/star", { ["passing"] = true, ["index"] = idx })

                if not res or res.status ~= 200 then
                    ngx.log(ngx.ERR, err)
                    ngx.timer.at(1, refresh_service)
                    return
                end
                index['dynamic_service'] = res.headers['X-Consul-Index']

                local services = res.body

                for _, service in ipairs(services) do
                    local peer = service['Service']['Address'] .. ":" .. service['Service']['Port']
                    realtime[peer] = true
                end

                -- diff exists between realtime
                -- in exists but not in realtime is invalid
                -- in realtime but not in exists is new
                local ok, invalid, new, err = hash_diff(exists, realtime)

                ngx.log(ngx.ERR, "invalid: " .. json.encode(invalid))
                ngx.log(ngx.ERR, "new: " .. json.encode(new))

                if ok then
                    for peer, _ in pairs(new) do
                        local ok, err = du.add_primary_peer('dynamic_service', peer)
                    end

                    for peer, _ in pairs(invalid) do
                        if peer ~= "0.0.0.1" then
                            local ok, err = du.remove_peer('dynamic_service', peer)
                        end
                    end
                end

                ngx.timer.at(1, refresh_service)
            end
            ngx.timer.at(0, refresh_service)
        end
    }

    server {
        listen 8080;
        location / {
            set_by_lua_block $service {
                return "dynamic_service"
            }
            proxy_pass http://$service;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }

        location /api {
            content_by_lua_block {
                local json = require "cjson"
                local du = require "ngx.dynamic_upstream"
                local ok, peers, e = du.get_primary_peers('dynamic_service')
                if ok then
                    ngx.say(json.encode(peers))
                else
                    ngx.say(json.encode("error while query peers: " .. e))
                end
            }
        }

        location /consul {
            content_by_lua_block {
                local resty_consul = require "resty.consul"
                local json = require "cjson"
                consul = resty_consul:new({
                        host="127.0.0.1",
                        port=8500,
                })
                local res, err = consul:get("/health/service/star")
                -- ngx.log(ngx.ERR, json.encode(err))
                if not res then
                    ngx.say("error while query services: " .. err)
                else
                    ngx.say(json.encode(res.body))
                end
            }
        }
    }
}
```

