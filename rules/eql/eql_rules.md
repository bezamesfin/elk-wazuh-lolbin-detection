# EQL Detection Rules (Kibana Detection Engine)

These are the Event Correlation (EQL) rules created in the Kibana detection engine, published exactly as they were used in the experiment.

**Index patterns**

- Individual technique rules (A-series): `winlogbeat-*`
- File integrity rule (A8): `wazuh-alerts-*`
- Chained rules (B-series): `winlogbeat-*` and `wazuh-alerts-*` together, so that sequences can span both data sources

Deployment steps are in the [configuration manual](../../docs/configuration-manual.md#eql-rules-in-kibana).

---

## Individual technique rules

### A1 · PowerShell Encoded Command (T1059.001)

```eql
process where process.name == "powershell.exe" and
  (
    process.command_line like~ "*-enc *" or
    process.command_line like~ "*-encodedcommand *" or
    process.command_line like~ "*-e *" or
    process.command_line like~ "*-version 2*"
  )
```

### A2 · Mshta Remote Execution (T1218.005)

```eql
process where process.name == "mshta.exe" and
  (
    process.command_line like~ "*http://*" or
    process.command_line like~ "*https://*" or
    process.command_line like~ "*javascript:*" or
    process.command_line like~ "*vbscript:*"
  )
```

### A3 · WMI Child Process Execution (T1047)

```eql
process where process.parent.name == "WmiPrvSE.exe" and
  process.name in ("cmd.exe", "powershell.exe", "wscript.exe",
                   "cscript.exe", "mshta.exe", "regsvr32.exe")
```

### A4 · Remote Thread Injection (T1055.001)

```eql
any where winlog.event_id == "8" and
  winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
  not process.name in ("MsMpEng.exe", "svchost.exe",
                       "csrss.exe", "lsass.exe")
```

### A5 · Regsvr32 Squiblydoo Execution (T1218.010)

```eql
process where process.name == "regsvr32.exe" and
  (
    process.command_line like~ "*/i:http*" or
    process.command_line like~ "*/i:ftp*" or
    (
      process.command_line like~ "*scrobj*" and
      process.command_line like~ "*/u*"
    )
  )
```

### A6 · WMI Event Subscription Persistence (T1546.003)

```eql
any where winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
  winlog.event_id in ("19", "20", "21")
```

### A7 · Alternate Data Stream Creation (T1564.004)

```eql
any where winlog.event_id == "15" and
  winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
  not file.name like~ "*Zone.Identifier*"
```

### A8 · File Integrity: Suspicious File in Staging Directory (T1105)

```eql
any where
  (
    syscheck.event == "added" or
    syscheck.event == "modified"
  ) and
  (
    syscheck.path like~ "*\\Windows\\Temp\\*" or
    syscheck.path like~ "*\\AppData\\Roaming\\*" or
    syscheck.path like~ "*\\ProgramData\\*"
  ) and
  (
    syscheck.path like~ "*.exe" or
    syscheck.path like~ "*.dll" or
    syscheck.path like~ "*.ps1" or
    syscheck.path like~ "*.bat" or
    syscheck.path like~ "*.vbs" or
    syscheck.path like~ "*.hta"
  )
```

### A9 · Security Policy Disabled via Registry Write (T1562.002)

```eql
any where winlog.event_id == "13" and
  winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
  (
    registry.path like~
      "*\\SOFTWARE\\Policies\\Microsoft\\Windows\\PowerShell*" or
    registry.path like~
      "*\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Policies*" or
    registry.path like~
      "*\\SYSTEM\\CurrentControlSet\\Services\\EventLog*"
  ) and
  (
    registry.data.strings like~ "*0*" or
    registry.data.strings like~ "*disabled*"
  )
```

---

## Chained scenario rules

### B1 · Defence Evasion → Encoded PowerShell → Process Injection

```eql
sequence by agent.name with maxspan=10m
  /* Phase A, Script Block Logging disabled (Sysmon EID 13) */
  [any where winlog.event_id == 13 and
   registry.path like~ "*ScriptBlockLogging*" and
   registry.data.strings like~ "*0*"]
  /* Phase B, encoded PowerShell, anchored on the Wazuh alert */
  [any where rule.level >= 7 and rule.mitre.id == "T1059.001"]
  /* Phase C, remote-thread injection (Sysmon EID 8) */
  [any where winlog.event_id == 8 and
   winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
   not process.name in ("MsMpEng.exe", "svchost.exe")]
```

### B2 · Encoded PowerShell → WMI Persistence → Log Clearing

```eql
sequence by host.name with maxspan=15m
  /* Phase A, encoded PowerShell */
  [process where process.name == "powershell.exe" and
   process.command_line like~ "*-enc*"]
  /* Phase B, WMI persistence installation */
  [any where winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
   winlog.event_id in ("19", "20", "21")]
  /* Phase C, log clearing using wevtutil */
  [process where process.name == "wevtutil.exe" and
   process.command_line like~ "*wevtutil*cl *"]
```

### B3 · LOLBin Execution Chain

```eql
sequence by host.name with maxspan=15m
  /* mshta LOLBin execution */
  [process where process.name == "mshta.exe" and
   (process.command_line like~ "*http*" or process.command_line like~ "*javascript*")]
  /* WMI spawns a child command interpreter */
  [process where process.parent.name == "WmiPrvSE.exe" and
   process.name in ("cmd.exe", "powershell.exe", "wscript.exe", "cscript.exe")]
  /* regsvr32 scriptlet execution */
  [process where process.name == "regsvr32.exe" and
   (process.command_line like~ "*scrobj*" or process.command_line like~ "*/i:http*")]
```

### B4 · Logging Evasion → WMI Execution → WMI Persistence

```eql
sequence by agent.name with maxspan=10m
  /* Phase A, ScriptBlockLogging disabled, keyed on registry.path
     (so the obfuscated StdRegProv variant can't evade it) */
  [any where winlog.event_id == 13 and
   registry.path like~ "*ScriptBlockLogging*" and
   registry.data.strings like~ "*0*"]
  /* Phase B, WMI spawns execution (Sysmon EID 1) */
  [process where process.parent.name == "WmiPrvSE.exe"]
  /* Phase C, WMI persistence subscription (Sysmon EID 19/20/21) */
  [any where winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
   winlog.event_id in ("19","20","21")]
```

### B5 · Cross-Source: Wazuh Alert → Sysmon Process Injection

```eql
sequence by agent.name with maxspan=5m
  /* Event 1, Wazuh EDR generates a high severity alert */
  [any where rule.level >= 7]
  /* Event 2, Sysmon detects process injection */
  [any where winlog.event_id == "8" and
   winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
   not process.name in ("MsMpEng.exe", "svchost.exe")]
```

### B6 · File Staging (FIM) → Execution → Registry Modification

```eql
sequence by agent.name with maxspan=10m
  /* Wazuh FIM detects suspicious file in staging directory */
  [any where syscheck.event in ("added", "modified") and
   (
     syscheck.path like~ "*\\windows\\temp\\*" or
     syscheck.path like~ "*\\appdata\\*" or
     syscheck.path like~ "*\\programdata\\*"
   ) and
   (
     syscheck.path like~ "*.ps1" or syscheck.path like~ "*.hta" or
     syscheck.path like~ "*.vbs" or syscheck.path like~ "*.bat"
   )]
  /* Sysmon detects script interpreter or LOLBin execution */
  [process where
   (
     process.name in ("powershell.exe", "mshta.exe",
                      "wscript.exe", "cscript.exe") or
     process.executable like~ "*\\windows\\temp\\*"
   )]
  /* Sysmon detects registry modification to persistence or policy path */
  [any where winlog.event_id == "13" and
   winlog.channel == "Microsoft-Windows-Sysmon/Operational" and
   (
     registry.path like~ "*\\Run*" or
     registry.path like~ "*\\SOFTWARE\\Policies\\Microsoft\\Windows\\PowerShell*" or
     registry.path like~ "*\\EventLog*" or
     registry.path like~ "*\\CurrentVersion\\Policies*"
   )]
```
