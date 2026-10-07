# MITRE ATT&CK Mapping

This table maps each simulated technique to the MITRE ATT&CK ID, the custom Wazuh
rule and the EQL rule that detects it in Kibana. 

## Individual techniques (A-series)

| ID  | Technique                              | ATT&CK      | Primary source            | Wazuh rule        | EQL rule |
|-----|----------------------------------------|-------------|---------------------------|-------------------|----------|
| A1  | PowerShell encoded command             | T1059.001   | Sysmon process creation   | 100001            | A1       |
| A2  | Mshta proxy execution                  | T1218.005   | Sysmon process creation   | 100002            | A2       |
| A3  | WMI child process execution            | T1047       | Sysmon parent-child       | 100003            | A3       |
| A4  | Reflective / remote-thread injection   | T1055.001   | Sysmon remote thread (EID 8) | 100004, 100005 | A4       |
| A5  | Regsvr32 scriptlet (squiblydoo)        | T1218.010   | Sysmon process creation   | 100006            | A5       |
| A6  | WMI event subscription persistence     | T1546.003   | Sysmon WMI (EID 19/20/21) | 100007, 100016, 100017 | A6 |
| A7  | Alternate data stream creation         | T1564.004   | Sysmon file stream (EID 15) | 100008          | A7       |
| A8  | Suspicious file in staging directory   | T1105       | Wazuh FIM                 | 100014            | A8       |
| A9  | Security policy disabled via registry  | T1562.002 / T1562.001 | Sysmon registry (EID 13) | 100009, 100010 | A9    |

## Chained scenarios (B-series)

| ID  | Scenario                                               | ATT&CK chain                       | Wazuh chained rules | EQL rule |
|-----|--------------------------------------------------------|------------------------------------|---------------------|----------|
| B1  | Defence evasion → encoded PowerShell → injection       | T1562.002 → T1059.001 → T1055.001  | 100020, 100021      | B1       |
| B2  | Encoded PowerShell → WMI persistence → log clearing    | T1059.001 → T1546.003 → T1070.001  | 100022, 100023      | B2       |
| B3  | LOLBin execution chain                                  | T1218.005 → T1047 → T1218.010      | 100024, 100025      | B3       |
| B4  | Logging evasion → WMI execution → WMI persistence       | T1562.002 → T1047 → T1546.003      | 100026, 100027      | B4       |
| B5  | Cross-source: Wazuh alert → Sysmon injection           | (composite)                        | —                   | B5       |
| B6  | File staging → execution → registry modification       | T1105 → T1059.001 → T1562.002      | 100029, 100030      | B6       |


