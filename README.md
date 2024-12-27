# High Availability Setup with Ansible

This project automates the configuration of a high-availability environment with the following components:

- Two virtual machines: `MASTER` and `BACKUP`, configured with `keepalived`.
- Three web servers on ports 80, 81, and 82 running in Docker containers on each machine.
- MySQL database for each machine.
- HAProxy for load balancing and caching.
- Prometheus and Grafana for monitoring.

## Project Overview

The environment includes:

- **Web Server Code**: Stored in the `src` folder and served from paths `/lk.php`, `/api.php`, and `/`.
- **SSL/TLS**: A self-signed certificate `rebrain.pem` for secure communication.
- **HAProxy**: Frontends and backends configured with caching, load balancing, and health checks.
- **Keepalived**: Manages virtual IP failover between MASTER and BACKUP.
- **Prometheus and Grafana**: For monitoring system health and metrics.

## Roles

The project is organized into the following Ansible roles:

1. **haproxy**: Configures HAProxy with SSL, caching, and load balancing.
2. **mysql**: Installs MariaDB and creates a user for HAProxy health checks.
3. **keepalived**: Configures MASTER and BACKUP nodes for IP failover.
4. **monitoring**: Sets up Prometheus, HAProxy Exporter, and Grafana for monitoring.

```yaml
all:
  hosts:
    master:
      ansible_host: 192.168.0.10
      ansible_user: root
    backup:
      ansible_host: 192.168.0.11
      ansible_user: root
```

1. **Prepare `host_vars`**

   Define specific configurations for `master` and `backup` in the `host_vars` directory:

   - `host_vars/master.yml`
   - `host_vars/backup.yml`

2. **Run the Playbook**

   ```bash
   ansible-playbook -i inventory.yaml site.yml
   ```

---

## Role Details

### 1. **HAProxy**

- Configures `rebrain_front` and `front_sql` frontends.
- Sets up `rebrain_api`, `rebrain_lk`, `rebrain_back`, and `rebrain_sql` backends.
- Enables caching (`rebrain_cache`) with parameters:
  - Max size: 4095 MB
  - Max object size: 10,000 bytes
  - Max lifetime: 30 seconds
- Deploys a self-signed SSL certificate.

### 2. **MySQL**

- Installs MariaDB.
- Creates a `haproxy` user for health checks.

### 3. **Keepalived**

- Configures IP failover between `MASTER` and `BACKUP`.
- Network settings applied via `sysctl`.
- Monitors HAProxy and assigns a virtual IP (`51.250.27.79`).

### 4. **Monitoring**

- **Prometheus**:
  - Scrapes HAProxy Exporter metrics.
  - Configuration file located at `/etc/prometheus/prometheus.yml`.
- **HAProxy Exporter**:
  - Scrape URI: `http://admin:admin@localhost:777/stats;csv`.
  - Runs on port `9091`.
- **Grafana** (on MASTER only):
  - Installed from the official Grafana repository.

---

## Outputs

After execution:

- MASTER and BACKUP are synchronized and ready for failover.
- HAProxy is configured with caching, SSL, and load balancing.
- Prometheus and Grafana provide system and application monitoring dashboards.
