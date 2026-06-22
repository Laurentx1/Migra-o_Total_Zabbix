# Inventário de Hosts - Migração Zabbix 7.0

## AGROSANTA (SRV-ZBXAGROSANTA)

| IP | Host |
|----|------|
| 192.168.0.7 | AGROSANTA-SRV-TORONTO-AD |
| 192.168.2.17 | SRV-BACKUP-ALTAMIRA |
| 192.168.0.22 | SRV-BACKUP-SANTAREM |

## ALIAR (SRV-ZBXALIAR)

| IP | Host |
|----|------|
| 172.16.12.51 | SRV-BACKUP |
| 172.16.12.9 | SRV-DOMINIO |
| 172.16.12.3 | SRV-DUBAI-AD |

## CAPECA (SRV-ZBXCAPECA)

| IP | Host |
|----|------|
| 192.168.3.99 | SRV-AD |
| 192.168.3.2 | SRV-AD02 |
| 192.168.3.103 | SRV-AGROTIS |
| 192.168.3.102 | SRV-ALTERDATA |
| 192.168.3.82 | SRV-BACKUP |
| 192.168.3.101 | SRV-FEEDBACK |

## CDC (SRV-CDCZBXPROXY)

| IP | Host |
|----|------|
| 10.100.22.3 | SERVERAD |
| 10.100.22.8 | SERVERAD-2008 |
| 10.100.22.19 | SRV-BACKUP |
| 10.100.22.2 | VMSERVBD |
| 10.100.22.9 | PROXMOX (Linux Agent) |

## CESARO (SRV-ZBXCESARO)

| IP | Host |
|----|------|
| 192.168.2.10 | SRV-AD |
| 192.168.2.199 | SRV-BACKUP |
| 192.168.2.5 | SRV-MSP |

## Scripts de Atualização do Zabbix Agent

### Windows (PowerShell)

```powershell
(Get-Content "C:\Program Files\Zabbix Agent\zabbix_agentd.conf") -replace '191\.31\.160\.14','177.104.184.226' |
Set-Content "C:\Program Files\Zabbix Agent\zabbix_agentd.conf"

Restart-Service "Zabbix Agent"
```

### Linux / Proxmox (Bash)

```bash
sed -i 's/191\.31\.160\.14/177.104.184.226/g' /etc/zabbix/zabbix_agent2.conf
systemctl restart zabbix-agent2
```
