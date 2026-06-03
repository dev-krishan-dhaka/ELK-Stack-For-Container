# ELK Stack — Docker Container Log Monitoring

Automatically collects and visualizes logs from **all Docker containers** on your server. Zero application changes needed.

---

## Requirements

- Docker + Docker Compose installed
- Ports **9200**, **5601**, **5044** open
- Minimum **4GB RAM** on your server

---

## Start the Stack

```bash
git clone https://github.com/your-repo/elk-stack.git
cd elk-stack
sudo docker-compose up -d
```

Wait ~60 seconds for all services to initialize.

---

## Verify

```bash
# Check all 4 containers are running
sudo docker-compose ps

# Check Elasticsearch is healthy
curl -X GET "localhost:9200"

# Check logs are flowing (should show docker-logs-* index)
curl -s "localhost:9200/_cat/indices?v" | grep docker
```

---

## Open Kibana

Go to `http://YOUR_SERVER_IP:5601`

**First time setup:**
1. Go to **Stack Management → Data Views → Create data view**
2. Index pattern: `docker-logs-*`
3. Timestamp field: `@timestamp`
4. Click **Save**
5. Go to **Discover** → select `docker-logs-*` → set time to **Last 7 days**

**Filter logs by container:**
```
container_name: frontend
container_name: backend
container_name: postgres
```

---

## Log Retention (15 days auto-delete)

Run once after starting the stack:

```bash
# Create delete policy
curl -X PUT "localhost:9200/_ilm/policy/docker-logs-policy" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot":    { "min_age": "0ms", "actions": {} },
        "delete": { "min_age": "15d", "actions": { "delete": {} } }
      }
    }
  }'

# Apply to all docker-logs-* indices
curl -X PUT "localhost:9200/_index_template/docker-logs-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["docker-logs-*"],
    "template": {
      "settings": { "index.lifecycle.name": "docker-logs-policy" }
    }
  }'

# Apply to existing indices
curl -X PUT "localhost:9200/docker-logs-*/_settings" \
  -H "Content-Type: application/json" \
  -d '{ "index.lifecycle.name": "docker-logs-policy" }'
```

To change retention days, re-run the first command with a different value e.g. `"30d"` or `"7d"`.

---

## Timezone

Logs are stored in UTC. To view in your local timezone:

1. Kibana → **Stack Management → Advanced Settings**
2. Search **"timezone"**
3. Set your zone e.g. `Asia/Kolkata` or `Europe/Amsterdam`
4. Click **Save**

---

## Stop / Start

```bash
sudo docker-compose down   # stop
sudo docker-compose up -d  # start
```

## Import Dashboard

After starting the stack, import the pre-built dashboard:

### Option A — Command line
```bash
curl -X POST "localhost:5601/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" \
  -F file=@kibana/dashboard.ndjson
```

### Option B — Kibana UI
1. Go to Stack Management → Saved Objects → Import
2. Select kibana/dashboard.ndjson
3. Click Import
4. Go to Dashboards → Docker logs

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Kibana not loading | Wait 60s — it takes time to initialize |
| No `docker-logs-*` index | Wait 60s then check again |
| `container_name: unknown` | Check Logstash has `user: root` in compose file |
| No logs in Discover | Change time range to **Last 7 days** |
| Disk filling up | Apply log retention policy above |
