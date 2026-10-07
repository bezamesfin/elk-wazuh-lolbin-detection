# Linux analysis server configs

Deploy on the Kali analysis server.

| File | Deploy to | Purpose |
|------|-----------|---------|
| `logstash-pipeline.conf` | `/etc/logstash/conf.d/` | Beats input (port 5044), ECS normalisation of Winlogbeat telemetry, Elasticsearch output |
| `filebeat.yml` | `/etc/filebeat/filebeat.yml` | Forwards Wazuh alerts (`/var/ossec/logs/alerts/alerts.json`) to the `wazuh-alerts-*` index |

Replace `<ELASTIC_PASSWORD>` and `<ELK_SERVER_IP>` before use. Restart the
service after editing (`systemctl restart logstash` / `systemctl restart filebeat`).
