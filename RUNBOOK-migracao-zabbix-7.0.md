# RUNBOOK - Migracao Zabbix 6.4 para 7.0 (nuvem)

**Data:** 2026-06-18
**Use este documento do inicio ao fim. Ele e o guia definitivo da migracao.**

---

## Antes de comecar (leia 2 minutos)

### Dois tipos de firewall - isso define o que migra agora

- **EDGE** (DNS termina em `.stsec.com.br`) = firewall proprio, base **Debian**. Suporta o proxy + ODBC. **MIGRA AGORA**: o proxy 7.0 vai instalado NO PROPRIO FIREWALL EDGE.
- **OSTEC** (DNS termina em `.ddns.com.br`) = appliance de terceiro, base **FreeBSD**. NAO suporta o driver ODBC da Microsoft e nao deve ser modificado. **NAO MIGRA AGORA**: o cliente so entra quando trocar o firewall para EDGE. Ate la fica no Zabbix 6.4.
- Dominio proprio (nem stsec nem ddns) = **verificar** com `uname -s` no firewall.

Como confirmar no firewall:
```bash
uname -s
```
`Linux` = EDGE (Debian, migra). `FreeBSD` = OSTEC (espera).

### O que voce vai migrar

**Objetivo: levar TODOS os hosts de cada cliente EDGE para o 7.0 e ir desligando as VMs de proxy.** Nao e so o backup - e o cliente inteiro (backup + ESXi + AD + Acronis + tudo que o proxy monitora), menos os lixos.

1. **Hosts via proxy** - o proxy vai no firewall EDGE; coleta o backup (via ODBC), os ESXi, AD, etc. e repassa para a nuvem.
2. **Firewalls EDGE** (os hosts "- EDGE") - reportam direto para a nuvem, sem proxy.

**ATENCAO carga:** levar tudo significa o proxy puxar todos os itens do cliente. Nos grandes (Campo Florido ~4140 itens, Global ~3944, Pirajuba ~3470, Capeca ~1726, e os >1100: Eurominas, Renovoltech, Aliar, Cesaro) isso e pesado para um firewall. Conferir CPU/RAM do firewall; se nao aguentar, NESSES casos manter a VM do proxy (so atualizar para 7.0) em vez de jogar no firewall.

No firewall EDGE vao coexistir duas coisas separadas, cada uma com seu arquivo de config:
- O agente (zabbix_agent2.conf) - monitora o proprio firewall (host EDGE), reporta direto para a nuvem
- O proxy (zabbix_proxy.conf) - coleta do servidor de backup, repassa para a nuvem

### Classificacao dos clientes de backup

**EDGE (stsec) - MIGRAM AGORA:**
CLUBP, CDC, AGROSANTA, ALIAR, CAMPOFLORIDO, CAPECA, CESARO, DERMAC, ESPLLENDA, EUROMINAS, GLOBAL, GUINDASTES, GURUPI, LEDIFRAN, MILENIUM, PIRAJUBA, RENATA (so o NEW), RENOVOLTECH, VALO

**OSTEC (ddns) - NAO MIGRAM AGORA:**
- ELAJ (grupoelaj.ddns.com.br)
- MIRO (miroagronegocios.ddns.com.br)
- RI NOVA PONTE (cartorionovaponte.ddns.com.br)

**VERIFICAR com uname (dominio proprio):**
- CONSTROI (remoto.constroiimobiliaria.com.br) - rodar `uname -s`; Linux = migra, FreeBSD = espera

**Fora de escopo por outros motivos (sao stsec, mas nao agora):**
- AGRICERT (proxy offline ha 6 meses), FAMA (offline ha dias), ICL (servidor de backup desligado)

### Dados fixos

| Item | Valor |
|------|-------|
| Zabbix origem (6.4) | 191.31.160.14 |
| Zabbix destino (7.0 nuvem) - IP publico p/ proxies | 177.104.184.226 |
| Frontend web da nuvem (acesso interno) | 172.31.12.55 |
| Porta proxy para servidor | 10051 |
| Template a migrar | Template Monitoramento ODBC Veeam B-R v1.1 |
| Macros ODBC | {$USERNAME} = sqlzabbix, {$PASSWORD} = sqlzabbix |

### Ordem do dia

```
PASSO 0  - Importar o template ODBC Veeam no 7.0 (fazer UMA vez)
BLOCO A  - Migrar os backups dos clientes EDGE (proxy no firewall)
BLOCO B  - Migrar os firewalls EDGE (sem proxy)
PASSO Z  - Conferir Grafana e aposentar VMs antigas
```

---

## PASSO 0 - Templates customizados (fazer uma unica vez)

Como vamos migrar TODOS os hosts, todos os templates customizados precisam existir no 7.0 antes. Os templates nativos do Zabbix (Windows by Zabbix agent, VMware, Linux by Zabbix agent, ICMP Ping) JA vem no 7.0 - nao exportar esses.

**0.1 - Identificar e exportar os customizados do 6.4 (191.31.160.14):**
- Data collection > Templates
- Selecionar os templates de nome proprio (nao-nativos). Confirmados ate agora:
  - Template Monitoramento ODBC Veeam B-R v1.1
  - Windows Usuario administrador
  - Windows Update (se nao for nativo na sua versao)
  - qualquer outro template proprio (Acronis, etc.)
- Marcar todos > Export > salvar como templates-custom.yaml

**0.2 - Importar no 7.0 (frontend 172.31.12.55):**
- Data collection > Templates > Import
- Selecionar o arquivo
- Marcar Create new e Update existing
- Import

Se na importacao de hosts (Bloco A) aparecer erro "template ... not found", e um customizado que faltou: volte aqui, exporte esse template e importe.

---

## BLOCO A - Backup de cliente EDGE (proxy no firewall)

Fazer este bloco para cada cliente EDGE com backup. So para EDGE (stsec). OSTEC (ddns) NAO entra agora.

### A1 - Acessar o firewall EDGE e confirmar que e Debian

```bash
uname -s                                  # tem que retornar Linux
cat /etc/os-release | grep -E "VERSION_CODENAME|VERSION_ID"
```
Se retornar FreeBSD, PARE: e OSTEC, nao migra agora.

### A2 - Instalar o proxy 7.0 no firewall EDGE

Exemplo Debian 12 (bookworm). Para Debian 11 trocar debian12 por debian11.
```bash
wget https://repo.zabbix.com/zabbix/7.0/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.0+debian12_all.deb
dpkg -i zabbix-release_latest_7.0+debian12_all.deb
apt update
apt install -y zabbix-proxy-sqlite3 zabbix-sql-scripts
```
Em duvida na URL, confirmar em zabbix.com/download (7.0 + Debian + versao).

### A3 - Gerar PSK e configurar o proxy

```bash
openssl rand -hex 32 | tee /etc/zabbix/zabbix_proxy.psk
chown zabbix:zabbix /etc/zabbix/zabbix_proxy.psk
chmod 600 /etc/zabbix/zabbix_proxy.psk
cat /etc/zabbix/zabbix_proxy.psk        # copiar a chave para o frontend
```

Editar /etc/zabbix/zabbix_proxy.conf:
```
Server=177.104.184.226
Hostname=SRV-ZBX[CLIENTE]        # IDENTICO ao nome do proxy no 6.4
DBName=/var/lib/zabbix/zabbix_proxy.db
LogFile=/var/log/zabbix/zabbix_proxy.log
TLSConnect=psk
TLSPSKIdentity=psk-[cliente]
TLSPSKFile=/etc/zabbix/zabbix_proxy.psk

# Cliente com ESXi/VMware (a maioria tem): habilitar coletores VMware.
# Sem estas linhas, os hosts ESXi e a descoberta de VMs NAO coletam.
# Guindastes (piloto) nao tem ESXi - pode pular estas 3 linhas.
StartVMwareCollectors=2
VMwareFrequency=60
VMwareCacheSize=16M
```

Iniciar:
```bash
mkdir -p /var/lib/zabbix
chown zabbix:zabbix /var/lib/zabbix
systemctl enable zabbix-proxy
systemctl restart zabbix-proxy
tail -f /var/log/zabbix/zabbix_proxy.log
```
Esperar: received configuration data from server at "177.104.184.226" (aparece depois do A5).

### A4 - Instalar e testar o ODBC no firewall

```bash
apt install -y unixodbc
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
curl https://packages.microsoft.com/config/debian/12/prod.list | tee /etc/apt/sources.list.d/mssql-release.list
apt update
ACCEPT_EULA=Y apt install -y msodbcsql18
```

Editar /etc/odbc.ini (IP do servidor de backup - ver tabela de IPs):
```ini
[VEEAM]
Driver = ODBC Driver 18 for SQL Server
Server = <IP_SERVIDOR_VEEAM>,1433
Encrypt = no
TrustServerCertificate = yes
DataBase = VeeamBackup
```

Testar:
```bash
isql -v VEEAM sqlzabbix sqlzabbix         # esperado: Connected!
```
Casos com 2 servidores (Agrosanta, Pirajuba): copiar o odbc.ini da VM antiga, que ja tem os 2 DSNs.

### A5 - Cadastrar o proxy no 7.0

Frontend 7.0:
- Data collection > Proxies > Create proxy
- Proxy name: SRV-ZBX[CLIENTE] (igual ao Hostname)
- Proxy mode: Active
- Aba Encryption: Connections from proxy = PSK; PSK identity = psk-[cliente]; PSK = colar a chave
- Add

### A6 - Exportar TODOS os hosts do proxy (do 6.4)

- Data collection > Hosts
- Filtro: Monitored by = Proxy, e selecionar o proxy SRV-ZBX[CLIENTE]
- Lista todos os hosts daquele cliente (backup + ESXi + AD + Acronis + etc.)
- Marcar TODOS, menos os lixos (ver lista de lixos)
- Export > salvar

### A7 - Importar no 7.0

- Data collection > Hosts > Import > selecionar o arquivo
- Marcar Create new em hosts, host groups e value maps
- Import

### A8 - Vincular todos ao proxy do firewall

Conferir se os hosts importados ja vieram com Monitored by proxy = SRV-ZBX[CLIENTE]. Se vierem como Server:
- Selecionar todos os hosts importados do cliente
- Mass update > aba Monitored by > Proxy > SRV-ZBX[CLIENTE] > Update

### A9 - Validar

- Monitoring > Latest data filtrando pelo proxy: varios hosts coletando
- Host de backup: item Conexao com o Banco = 1
- ESXi: descoberta de VMs OK
- Execute now nos itens de descoberta dos principais

### A10 - Parar a coleta no 6.4

Para nao haver coleta dupla, no 6.4 (191.31.160.14):
- Selecionar os hosts daquele cliente
- Disable (ou deletar, se ja estiver confiante)

### A11 - Aposentar a VM do proxy

- Confirmar que TUDO daquele cliente esta coletando no 7.0
- Desligar a VM do proxy no Proxmox/ESXi
- Aguardar 24h observando alertas
- Deletar a VM

---

## BLOCO B - Firewalls EDGE (sem proxy)

Apenas os EDGE (stsec). Os OSTEC (ddns) continuam no 6.4 ate trocarem de firewall.

### B1 - Exportar os EDGE do 6.4
- Data collection > Hosts > filtrar por host group Edge Protect
- Marcar os EDGE validos (excluir os lixos)
- Export

### B2 - Importar no 7.0
- Data collection > Hosts > Import > Create new > Import
- Conferir: Monitored by proxy vazio; interface por DNS

### B3 - Atualizar o agente em cada firewall EDGE
```bash
grep -E "^Server|^ServerActive|^Hostname" /etc/zabbix/zabbix_agent2.conf
sed -i 's/191\.31\.160\.14/177.104.184.226/g' /etc/zabbix/zabbix_agent2.conf
systemctl restart zabbix-agent2
```

### B4 - Validar
- Monitoring > Latest data: cada EDGE coletando.
Nota de seguranca: os EDGE expoem a 10050 na internet. Recomendado PSK tambem nos agentes (2a passada).

---

## TABELA DE IPs - servidor Veeam por cliente (para o odbc.ini, passo A4)

IMPORTANTE: e o ponto de partida (interface do host no 6.4). Validar com ping e isql antes de fechar cada odbc.ini.

| Ordem | Host de backup | Proxy (Hostname) | IP servidor Veeam | Firewall | Obs |
|-------|----------------|------------------|-------------------|----------|-----|
| 1 | G T - SRV - LONDON | SRV-ZBXGUINDASTES | guindastestriangulo.stsec.com.br | EDGE | PILOTO - 1 host so; Veeam por DNS |
| 2 | LEDIFRAN-SRV-BACKUP | SRV-ZBXLEDIFRAN | 192.168.100.22 | EDGE | |
| 3 | CESARO - SRV-BACKUP | SRV-ZBXCESARO | 192.168.2.199 | EDGE | ODBC ja validado |
| 4 | SUPERRENATA - SRV-BACKUP - NEW | SRV-ZBXRENATA | 192.168.0.72 | EDGE | so o NEW |
| 5 | GURUPI-SRV-BACKUPNEW | SRV-ZBXGURUPI | 192.168.100.20 | EDGE | |
| 6 | CLUBP-SRV-BACKUP | CLUBP-SRV-ZBXCLUBP | 192.168.100.235 | EDGE | usar .235 (o .233 morreu) |
| 7 | CDC-SRV-BACKUP | SRV-CDCZBXPROXY | 10.100.22.19 | EDGE | |
| 8 | CAPECA-SRV-BACKUP | SRV-ZBXCAPECA | 192.168.3.82 | EDGE | |
| 9 | DERMAC - SRV - BACKUP | SRV-ZBXDERMAC | 10.10.10.97 | EDGE | |
| 10 | CONSTROI - SRV-BACKUP | SRV-ZBXCONSTROI | 172.17.1.41 | VERIFICAR | uname antes |
| 11 | ESPLLENDA-SRV-BACKUP | SRV-ZBXESPLLENDA | 192.168.99.108 | EDGE | |
| 12 | EUROMINAS-SRV-BACKUP | SRV-ZBXEUROMINAS | 192.168.1.21 | EDGE | |
| 13 | MILENIUM-SRV-AD-VEEAM | SRV-ZBXMILENIUM | 172.21.12.5 | EDGE | |
| 14 | RENOVOLTECH - SRV-BACKUP | SRV-ZBXRENOVOLTECH | 192.168.1.192 | EDGE | |
| 15 | VALO-SRV-LONDON | SRV-ZBXVALO-UDI | 172.31.16.2 | EDGE | |
| 16 | ALIAR - SRV-BACKUP | SRV-ZBXALIAR | 172.16.12.51 | EDGE | |
| 17 | SRV-BACKUP - SANTAREM | SRV-ZBXAGROSANTA | 192.168.0.22 | EDGE | mesmo proxy do Altamira |
| 17 | SRV-BACKUP - ALTAMIRA | SRV-ZBXAGROSANTA | 192.168.2.17 | EDGE | 2 DSNs no mesmo proxy |
| 18 | PIRAJUBA-SRV-BACKUP - 105 | SRV-ZBXPIRAJUBA | 192.168.0.105 | EDGE | mesmo proxy do 124 |
| 18 | PIRAJUBA-SRV-BACKUP2 - 124 | SRV-ZBXPIRAJUBA | 192.168.0.124 | EDGE | 2 DSNs no mesmo proxy |
| 19 | GLOBAL LOG - SRV-BACKUP | SRV-ZBXGLOBAL | 192.168.0.62 | EDGE | |
| 20 | PMCF - SRV-BACKUP | SRV-ZBXCAMPOFLORIDO | 10.10.10.124 | EDGE | |

### NAO migram agora (OSTEC / fora de escopo)

| Host de backup | Proxy | Firewall | Motivo |
|----------------|-------|----------|--------|
| MIRO-SRV-BACKUP | SRV-ZBXMIRO | OSTEC (ddns) | firewall FreeBSD - espera virar EDGE |
| GRUPO-ELAJ-SRV-BACKUP | ELAJ-SRV-ZBXELAJ | OSTEC (ddns) | firewall FreeBSD - espera virar EDGE |
| RI NOVA PONTE - TORONTO | SRV-ZBXRINOVAPONTE | OSTEC (ddns) | FreeBSD + offline |
| AGRICERT - SRV - BACKUP | SRV-ZBXAGRICERT | EDGE | proxy offline 6 meses |
| FAMA-SRV-BOSTON | SRV-ZBXFAMA | EDGE | proxy offline |
| ICL - SRV-BACKUP | SRV-ZBXICLCURSOS | EDGE | servidor de backup desligado |

### Casos especiais (odbc.ini)

- **Agrosanta e Pirajuba**: 1 proxy, 2 servidores de backup. Precisam de 2 DSNs no odbc.ini. Copiar o odbc.ini da VM antiga e so atualizar IP/Driver.
- **Superrenata**: 2 hosts no IP 192.168.0.72; so o NEW tem ODBC Veeam. Migrar so o NEW.
- **Guindastes**: servidor Veeam por DNS. Conferir o IP interno na hora do odbc.ini.

---

## LIXOS - nao migrar (hosts EDGE incompletos / duplicados)
- Classe A Sementes - EDGE (3 itens)
- Produtiva Trading - EDGE (3 itens)
- Sementes Gurupi - EDGE (3 itens) - manter o SEMENTES GURUPI - EDGE com 66 itens
- Tabelionato Helilia - EDGE (3 itens)
- Sigma Assessoria - EDGE / SIGMA ASSESSORIA - EDGE (mesmo DNS - levar so um)

---

## PILOTO e ordem

**Piloto: GUINDASTES.** E EDGE (stsec), o proxy tem 1 host so (o proprio backup), entao nao deixa hosts orfaos no 6.4 - ideal para validar o processo de proxy no firewall sem complicacao. Unico ponto: o servidor Veeam e por DNS, confirmar o IP interno na hora.

Depois do piloto, seguir a ordem da tabela de IPs (Ledifran, Cesaro, etc.).

---

## Checklist por cliente (Bloco A - migrar tudo)

Cliente: ______  Proxy: ______  Firewall (EDGE?): ______  IP Veeam: ______

- [ ] uname -s = Linux (e EDGE)
- [ ] Proxy 7.0 instalado no firewall
- [ ] PSK gerada (chmod 600)
- [ ] zabbix_proxy.conf (Server nuvem, Hostname igual ao 6.4, TLS PSK)
- [ ] ODBC instalado e isql Connected!
- [ ] Proxy cadastrado no 7.0 com PSK
- [ ] Log: received configuration data from server
- [ ] TODOS os hosts do proxy exportados (menos lixos) e importados
- [ ] Mass update: todos vinculados ao proxy do firewall
- [ ] Coleta OK (varios hosts) e Conexao com o Banco = 1
- [ ] Hosts desabilitados no 6.4 (sem coleta dupla)
- [ ] VM aposentada (apos 24h)

## Checklist EDGE (Bloco B)
- [ ] EDGE validos exportados (lixos fora)
- [ ] Importados no 7.0 (sem proxy)
- [ ] Agente de cada firewall apontando para 177.104.184.226
- [ ] EDGE coletando no 7.0
