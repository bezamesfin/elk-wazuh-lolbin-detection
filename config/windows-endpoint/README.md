# Windows endpoint configs

Deploy on the Windows Server 2025 endpoint.

| File | Deploy to | Purpose |
|------|-----------|---------|
| `winlogbeat.yml` | Winlogbeat install dir | Collects Sysmon + Windows Security/Application/System channels, outputs to Logstash |
| `ossec.conf` | `C:\Program Files (x86)\ossec-agent\ossec.conf` | Wazuh agent config; also forwards the Sysmon channel to the Wazuh Manager |
| `sysmonconfig.xml` | applied via `Sysmon64.exe -accepteula -i sysmonconfig.xml` | Sysmon event configuration (based on Olaf Hartong's sysmon-modular) |

Replace `<ELK_SERVER_IP>` before use.
