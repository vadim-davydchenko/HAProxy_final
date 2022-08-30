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
  - Work in http mode
  - Contain a named acl named `is_cached` that checks using path_end that the path request ends in .js, .php. or .css
leastconn balancing mode
  - Use rebrain_cache to get cache files and add new ones if acl `is_cached` conditions are met
  - Redirect traffic to 2 servers named `rebrain_01_81` and `rebrain_02_81` at `127.0.0.1:81`. Activate health_check with a check interval of 4 seconds.
  - Maximum number of connections for `rebrain_02_81` 80

#### 9.Setup backend [rebrain_back](https://github.com/vadim-davydchenko/HAProxy_final/blob/8ec2ff518ba32d81471634147d9f4a3e2537bda5/haproxy.cfg#L60)
  - Work in http mode
  - Balancer source
  - Prefix PHPSESSID cookies with s1 for `rebrain_01_82` server, s2 for `rebrain_02_82` server, and s3 for `rebrain_03_82` server. Disable caching for cookies.
  - Redirect traffic to 3 servers `rebrain_01_82`, `rebrain_02_82` and `rebrain_03_82` at `127.0.0.1:82`. Activate health check. Explicitly specify a health check on port 82. The check interval is 8 seconds. The maximum number of connections is 1100 for each server.

#### 10.Setup backend [rebrain_sql](https://github.com/vadim-davydchenko/HAProxy_final/blob/8e3ff03d4deeca72dd6a76c7c777871e2559e4e0/haproxy.cfg#L68)
  - Roundrobin balancing mode.
  - Activate mysql health-check for user haproxy.
  - Redirect traffic to `rebrain_db_1` and `rebrain_db_2` servers at `127.0.0.1:3306`. Activate health-check, explicitly specify health check on port 3306, interval between checks is 2 seconds, assign server inactive after 2 failed checks, assign active after 1 successful check, maximum number of connections is 100.

#### 11.Add a [listen stat](https://github.com/vadim-davydchenko/HAProxy_final/blob/0d294e95dd3460b95a1ded5db61a19a473d0fc3d/haproxy.cfg#L74) that satisfies the following conditions:
  - Listen on port 777
  - Work in http mode
  - Activate statistics
  - Activate show-legends
  - Update every 30 seconds
  - Login:password for statistics admin:admin
  - Disable haproxy version display
  - Assign authorization realm Haproxy Statistics

#### 12.Add [logging](https://github.com/vadim-davydchenko/HAProxy_final/blob/09a0d4dfed52e00b2e0207d795b134f05a27d6da/haproxy.cfg#L19) format to the defaults block
#### 13.Setting `MySQL`, add new user database with name `haproxy` for use in heelth-check
```
sudo apt install mariadb-server -y
CREATE USER haproxy;
```

#### 14.Setting `keepalived`
Monitor the haproxy process on both servers and if it is disabled on the master server, the virtual ip must be assigned to the backup server, and when enabled, assigned back to the master server.
  - Setting network in `sysctl` and install `keepalived`
 ```
 net.ipv6.conf.all.disable_ipv6 = 1
 net.ipv6.conf.default.disable_ipv6 = 1
 net.ipv4.ip_nonlocal_bind = 1
 net.ipv4.conf.all.arp_ignore = 1
 net.ipv4.conf.all.arp_announce = 1
 net.ipv4.conf.all.arp_filter = 0
 net.ipv4.conf.eth0.arp_filter = 1
 
 sudo sysctl -p
 sudo apt-get install keepalived -y
 ```
 
  - Setting `/etc/keepalived/keepalived.conf` for [MASTER](https://github.com/vadim-davydchenko/HAProxy_final/blob/master/keepalived_master.conf) and [BACKUP](https://github.com/vadim-davydchenko/HAProxy_final/blob/master/keepalived_backup.conf)
 
