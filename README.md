
# Integrating ELK with Wazuh EDR for Low-Visibility Attack Detection

Research artefact for an MSc Cybersecurity thesis investigating whether
integrating the **ELK stack** (Elasticsearch, Logstash, Kibana) with the
**Wazuh EDR** platform improves detection of low-visibility attack techniques —
specifically **LOLBins** and **fileless execution**.

This repository contains everything needed to rebuild the detection lab and
replicate the experiment. 

> **Research question** — How effectively can integrating the ELK stack with an
> EDR platform improve detection of low-visibility malware such as LOLBins and
> fileless techniques?

---

## Key finding

Integration did **not** raise the raw detection rate, a technique seen by one
platform was generally seen by the other. What integration changed was
everything *after* detection: **ordered-sequence correlation, cross-field lineage
joins, structured record output, and cross-source correlation**. In other words,
the contribution is architectural — it is about how completely an incident can be
*reconstructed*, not how many techniques are *detected*.

This is measured with a custom **process-tree completeness** metric.

---

## Experimental design

A controlled experiment runs the same simulations under three conditions,
toggling only the Winlogbeat and Wazuh agent services:

| Condition | Setup                               | What generates alerts            |
|-----------|-------------------------------------|----------------------------------|
| **A** — Wazuh EDR only  | Wazuh agent running, Winlogbeat stopped | Wazuh custom rules          |
| **B** — ELK SIEM only   | Winlogbeat running, Wazuh agent stopped | EQL rules (`winlogbeat-*` only) |
| **C** — Integrated      | Both running                         | Both platforms simultaneously   |

**Workload:** 9 individual techniques (A1–A9) and 6 chained scenarios (B1–B6),
all mapped to MITRE ATT&CK — see [`docs/attack-mapping.md`](docs/attack-mapping.md).

**Evaluation metrics:**

1. **Detection rate** — was the technique detected at all
2. **Detection latency** — time from execution to alert is generated
3. **False positive rate**
4. **Process-tree completeness** *(custom)* — how much of the expected attack
   process tree can be populated and linked

### Process-tree completeness

A node-and-edge score adapted from provenance-graph theory:

```
completeness = (nodes populated + edges resolved) / (nodes expected + edges expected)
```

- **Base** completeness is the platform's output from its own alerts.
- **Reconstructed** completeness is  recovered by linking related
  events which is done automatically by EQL sequence rules in Conditions B and C, but
  requiring manual analyst joining in Condition A.

This is the key point: Wazuh creates separate alerts for each attack phase
so an analyst must connect them together, whereas
the sequence engine assembles the tree for you.

---

## Architecture

![System architecture](docs/images/architecture.png)

Two VirtualBox VMs on an isolated host-only network:

- **Linux analysis server (Kali)** — Elasticsearch, Kibana, Logstash, Wazuh
  Manager, Filebeat
- **Windows endpoint (Windows Server 2025)** — Sysmon, Winlogbeat, Wazuh agent

Telemetry travels two paths: Sysmon and Windows channels go through Winlogbeat →
Logstash → Elasticsearch, the Wazuh agent sends events to the Wazuh Manager
whose alerts Filebeat forwards into the `wazuh-alerts-*` index. Full build steps
are in the [configuration manual](docs/configuration-manual.md).

---

## Repository layout

```
.
├── README.md
├── docs/
│   ├── configuration-manual.md     # Full lab build guide (with screenshots)
│   ├── attack-mapping.md           # Simulation ↔ Wazuh rule ↔ EQL rule ↔ ATT&CK
│   └── images/                     # Diagrams and setup screenshots
├── config/
│   ├── linux-server/
│   │   ├── logstash-pipeline.conf  # Beats input, ECS normalisation, ES output
│   │   └── filebeat.yml            # Forwards Wazuh alerts to Elasticsearch
│   └── windows-endpoint/
│       ├── winlogbeat.yml          # Sysmon + Windows channels → Logstash
│       ├── ossec.conf              # Wazuh agent config
│       └── sysmonconfig.xml        # Sysmon config (Olaf Hartong sysmon-modular)
├── rules/
│   ├── wazuh/
│   │   └── local_rules.xml         # Custom Wazuh rules 100001–100030
│   └── eql/
│       └── eql_rules.md            # Individual + chained EQL rules for Kibana
├── simulations/
│   └── attack-simulations.md       # Adversary emulation (LAB USE ONLY)
└── results/
    └── thesis_result_spreadsheet.xlsx   # Raw per-condition scoring
```

---

## Reproducing the experiment

1. **Build the lab** — follow [`docs/configuration-manual.md`](docs/configuration-manual.md)
   to prepare both VMs and all components.
2. **Deploy detection rules:**
   - Add [`rules/wazuh/local_rules.xml`](rules/wazuh/local_rules.xml) to
     `/var/ossec/etc/rules/`, validate with `wazuh-logtest -t`, restart the manager.
   - Create each rule in [`rules/eql/eql_rules.md`](rules/eql/eql_rules.md) in the
     Kibana detection engine as an Event Correlation rule.
3. **Set a condition** by starting/stopping the Winlogbeat and Wazuh agent
   services as per the table above.
4. **Run the simulations** from [`simulations/`](simulations/attack-simulations.md)
   on the Windows endpoint, in an **isolated lab **.
5. **Score** each alert against the four metrics; raw figures are in
   [`results/`](results/).

Replace the placeholders with your own values:

- `<ELASTIC_PASSWORD>` — your Elasticsearch `elastic` user password
- `<ELK_SERVER_IP>` — the IP of your Linux analysis server

---

##  Disclaimer

The contents of `simulations/` are adversary-emulation techniques for a 
self-owned lab. Running them against systems you do not own or control may be
illegal. They are added only to document and allow reproduction of this
research.

---

## Credits & licence

- Sysmon configuration is based on **Olaf Hartong's**
  [sysmon-modular](https://github.com/olafhartong/sysmon-modular) was used under its own licence.
- Several simulation steps use **Red Canary's**
  [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team).


