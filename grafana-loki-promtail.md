นี่คือตัวอย่างไฟล์ `docker-compose.yml` ที่ใช้งาน **Grafana Loki**, **Promtail**, และ **Grafana** ด้วยกันอย่างครบชุด เหมาะสำหรับการทดลองหรือใช้งานเบื้องต้น:

---

## 🐳 `docker-compose.yml` สำหรับ Loki + Promtail + Grafana

```yaml
version: '3'

services:
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      - /var/log:/var/log
      - /etc/hostname:/etc/hostname
    command: -config.file=/etc/promtail/config.yaml
    depends_on:
      - loki

  grafana:
    image: grafana/grafana:10.2.3
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    depends_on:
      - loki

volumes:
  grafana-storage:
```

---

## 📝 ตัวอย่าง `loki-config.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  max_chunk_age: 1h
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/boltdb-cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: true
  retention_period: 168h
```

---

## 📝 ตัวอย่าง `promtail-config.yaml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
```

---

## 🚀 วิธีใช้งาน

1. สร้างไฟล์: `docker-compose.yml`, `loki-config.yaml`, `promtail-config.yaml`

2. รันคำสั่ง:

   ```bash
   docker-compose up -d
   ```

3. เปิด Grafana ผ่าน: [http://localhost:3000](http://localhost:3000)

   * User: `admin`
   * Password: `admin` (หรือระบบอาจให้ตั้งตอนแรก)

4. เพิ่ม **Loki Data Source** ใน Grafana:

   * URL: `http://loki:3100`

5. สร้าง dashboard เพื่อดู log ได้ทันที

---

หากคุณต้องการให้ Promtail เก็บ log จาก container อื่นด้วย แจ้งผมได้เลย ผมสามารถช่วยปรับ config ให้เหมาะกับการเก็บ log จาก Docker container โดยตรง.
