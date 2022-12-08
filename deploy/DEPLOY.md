### deploy

1.decentraland matrix 服务器部署脚本：https://github.com/decentraland/matrix-playbook

2.```git clone https://github.com/decentraland/matrix-playbook.git```

3.修改脚本配置
```shell
cp .env.example .env
vim env
```

  ```
  SSH_USER=ubuntu  // 域名解析绑定的服务器的登录用户名
  SUBDOMAIN=synapse  // 二级域名的域
  MATRIX_DOMAIN=decentraland.org  // 顶级域名
  ENV=dev
  GENERIC_SECRET_KEY=someKey  // 用于基础加密的密钥
  ENABLE_OPEN_REGISTRATION=true
  USE_PRESENCE=false
  USE_DECENTRALAND_PASSWORD_PROVIDER=true
  USE_EXTERNAL_DB=false
  POSTGRES_USER=postgres_user          // postgres数据库 user
  POSTGRES_HOST=postgres_host          // postgres数据库 host
  POSTGRES_PASSWORD=postgres_password  // postgres数据库 密码
  POSTGRES_DATABASE=postgres_database  // postgres数据库 指定数据库
  METRICS_BEARER_TOKEN=bearer-token
  USE_PROMETHEUS_POSTGRES_EXPORTER=false
  PROMETHEUS_EXPORTER_POSTGRES_USER=prometheus_postgres_user   // prometheus postgres数据库 user
  PROMETHEUS_EXPORTER_POSTGRES_HOST=prometheus_postgres_host   // prometheus postgres数据库 host
  NGINX_PROXY_WORKER_CONNECTIONS=20480
  POSTGRES_MAX_CONNECTIONS=300
  USE_WORKER_AFFINITY=true
  USE_ELASTI_CACHE=false
  REDIS_HOST=redis_host   // redis host
  REDIS_PORT=redis_port   // redis 端口
  GENERIC_WORKERS_COUNT=3
  DOCKER_USERNAME=someUser      // docker hub 用户名
  DOCKER_PASSWORD=somePassword  // docker hub 用户密码
  ```

4.需要jinja2环境，安装j2cli，需要在docker 起ansible
```
pip install jinja2
pip install j2cli （本地编辑好脚本后，服务器可不需要j2cli以及jinja2环境）
```

```
j2 -v
j2cli 0.3.10, Jinja2 2.11.3
```

```
docker info
Server Version: 20.10.14
```

5.配置二级域名以及域名解析绑定一台公网服务器
服务器需要密钥登录和密码登录
密钥登录配置路径
```
~/.ssh/

.
├── authorized_keys
├── id_rsa
└── id_rsa.pub
```

6.修改脚本
```
templates/hosts.j2  25
删除 ansible_connection=local

templates/main.yml.j2 198
修改 trusted_servers 配置
trusted_servers:
- https://peer.decentraland.org/lambdas
- https://peer-ec1.decentraland.org/lambdas
- https://peer-wc1.decentraland.org/lambdas
- https://peer-eu1.decentraland.org/lambdas
- https://peer-ap1.decentraland.org/lambdas
- https://interconnected.online/lambdas
修改为私有lambdas服务
```

```
cd matrix-playbook
./generate-inventory.sh
```

7.服务器运行脚本
```
cd matrix-playbook

docker run -it --rm -w /work -v `pwd`:/work -v $HOME/.ssh/id_rsa:/root/.ssh/id_rsa:ro --entrypoint=/bin/sh docker.io/devture/ansible:2.11.6-r1

ansible-playbook -i inventory/hosts setup.yml --tags=setup-all

ansible-playbook -i inventory/hosts setup.yml --tags=start
```

8.停止服务
```
ansible-playbook -i inventory/hosts setup.yml --tags=stop
```

9. kernel 代码修改
```
packages/shared/friends/utils.ts getMatrixIdFromUser 29
matrix的服务器域名需更换：xxxx.com

packages/shared/meta/sagas.ts fetchMetaConfiguration 165
https://synapse.decentraland.org、https://synapse.decentraland.zone
均需要修改为部署的二级域名：chat.xxxx.com

packages/shared/meta/selectors.ts getSynapseUrl 92
store.meta.config.synapseUrl ?? 'https://synapse.decentraland.zone'
修改为部署的二级域名：chat.xxxx.com
```

10.错误解决
```
error:
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: ModuleNotFoundError: No module named 'docker'
fatal: [chat.xxxx.com]: FAILED! => changed=false
msg: 'Failed to import the required Python library (Docker SDK for Python: docker above 5.0.0 (Python >= 3.6) or docker before 5.0.0 (Python 2.7) or docker-py (Python 2.6)) on decentraland''s Python /usr/bin/python3. Please read the module documentation and install it in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter, for example via `pip install docker` (Python >= 3.6) or `pip install docker==4.4.4` (Python 2.7) or `pip install docker-py` (Python 2.6). The error was: No module named ''docker'''

solve:
pip3 install docker-py
```

```
error:
failed: [chat.xraremeta.com] (item=matrix-nginx-proxy.service) => changed=false
  ansible_loop_var: item
  item: matrix-nginx-proxy.service
  msg: matrix-nginx-proxy.service was not detected to be running. It's possible that there's a configuration problem or another service on your server interferes with it (uses the same ports, etc.). Try running `systemctl status matrix-nginx-proxy.service` and `journalctl -fu matrix-nginx-proxy.service` on the server to investigate. If you're on a slow or overloaded server, it may be that services take a longer time to start and that this error is a false-positive. You can consider raising the value of the `matrix_common_after_systemd_service_start_wait_for_timeout_seconds` variable. See `roles/matrix-common-after/defaults/main.yml` for more details about that.

solve:
80端口被占用，需要修改/etc/nginx/nginx.conf以及conf.d配置文件的80端口。
journalctl -fu matrix-nginx-proxy.service
systemctl status matrix-nginx-proxy.service
```
