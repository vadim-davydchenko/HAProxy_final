### Install 2 virtual machines: one MASTER, another BACKUP, establish interaction between them using `keepalived`. Each machine has 3 servers installed on ports 80, 81 and 82 and a MySQL database. The src folder contains the web server code. Are processed by the case along the paths /lk.php, /api.php and /
#### 1.Create a self-signed certificate `rebrain.pem containing the certificate and private key and place it in `/etc/haproxy/rebrain.pem`

```
openssl genrsa -out rebrain.key 2048
openssl req -x509 -new -nodes -key rebrain.key -sha256 -days 1024 -out rebrain.pem
cat rebrain.pem rebrain.key > /etc/haproxy/rebrain.pem
```
#### 2.The config should contain *cache* [rebrain_cache](https://github.com/vadim-davydchenko/HAProxy_final/blob/c9b08f6306c900d2ea9d967d0af762d0a4f3a105/haproxy.cfg#L20) which should be configured like this:
  - Maximum memory size 4095
  - Maximum object size 10000
  - Maximum lifetime 30
#### 3.The config must contain 2 frontends `rebrain_front` and `front_sql`
#### 4.Setting frontend [rebrain_front](https://github.com/vadim-davydchenko/HAProxy_final/blob/95e209bcc3cfba37070ed131aec6196e81a4d998/haproxy.cfg#L25)
  - Listen on port 443 using `rebrain.pem` certificate
  - Work in http mode
  - Add client ip address to all incoming packets in `X-Forwarded-For` header
  - Use cache `rebrain_cache` for incoming requests
  - Use cache `rebrain_cache` for server responses
  - Use named `acl` `url_api` checking that request starts with api using path_beg
  - Use backend `rebrain_ap`i if `acl` `url_api` is running
  - Use named `acl` `url_lk` checking that request starts with `lk` using path_beg
  - Use backend `rebrain_lk` if `acl` `url_lk` is running
  - Default backend `rebrain_back`
#### 5.Setting frontend [front_sql](https://github.com/vadim-davydchenko/HAProxy_final/blob/7f3206ecfb7d6592120ff5e141e56ef8b4b12c40/haproxy.cfg#L37)
#### 6.Setup 3 backends `rebrain_api`, `rebrain_lk` and `rebrain_back`
#### 7.Setup backend [rebrain_api](https://github.com/vadim-davydchenko/HAProxy_final/blob/5f504f3d4921a7bbea867974eedabc6d2c30e705/haproxy.cfg#L43)
  - Work in http mode
  - Use roundrobin balancing
  - Use the `prefer-last-server` option
  - Add REBRAIN cookie with value `rebrain_01_80` for `rebrain_01_80` server and `rebrain_02_80` for `rebrain_02_80` server using cookie insert. Do not assign new links if the   request already contains them. Don't store cookies in cache.
  - Redirect traffic to 2 servers named `rebrain_01_80`, `rebrain_02_80`, respectively, at `127.0.0.1:80`. Activate health check for each server.

#### 8.Setup backend [rebrain_lk](https://github.com/vadim-davydchenko/HAProxy_final/blob/c24b2a58aeca1f521bc22a487932d107863f4293/haproxy.cfg#L51)

