# 🛡️ Configuração Completa de Auditoria de Segurança em Linux com ClamAV, Maldet e RKHunter

## 🧰 Ferramentas utilizadas
- **ClamAV**: Antivírus open-source.
- **Maldet**: Detector de malware específico para Linux.
- **Rootkit Hunter (RKHunter)**: Detector de rootkits e backdoors.

---

## 📁 Scripts em `/usr/local/bin`

### 1. `/usr/local/bin/scan_clam_home_tmp.sh`
```bash
#!/bin/bash
WATCH_DIRS="/home /tmp"
LOG="/var/log/clamav/home_tmp_scan.log"

freshclam --quiet

clamdscan --no-summary --fdpass --multiscan --infected --remove $WATCH_DIRS 2>/dev/null | awk '
  /FOUND/ {
    status="infected";
    sig=substr($0, index($0, ":") + 2);
    sig=substr(sig, 1, length(sig)-6)
    printf("{\"timestamp\":\"%s\",\"path\":\"%s\",\"result\":\"%s\",\"signature\":\"%s\"}\n", strftime("%FT%T%z"), $1, status, sig)
  }
' >> $LOG
```

### 2. `/usr/local/bin/scan_clam_system.sh`
```bash
#!/bin/bash
WATCH_DIRS="/etc /bin /sbin /usr/bin /usr/sbin"
LOG="/var/log/clamav/system_dirs_scan.log"

freshclam --quiet

clamdscan --no-summary --fdpass --multiscan --infected --remove $WATCH_DIRS 2>/dev/null | awk '
  /FOUND/ {
    status="infected";
    sig=substr($0, index($0, ":") + 2);
    sig=substr(sig, 1, length(sig)-6)
    printf("{\"timestamp\":\"%s\",\"path\":\"%s\",\"result\":\"%s\",\"signature\":\"%s\"}\n", strftime("%FT%T%z"), $1, status, sig)
  }
' >> $LOG
```

### 3. `/usr/local/bin/scan_clam_full.sh`
```bash
#!/bin/bash
freshclam --quiet
WATCH_DIRS="/"
LOG="/var/log/clamav/full_root_scan.log"

freshclam --quiet

clamdscan --no-summary --fdpass --multiscan --infected --remove $WATCH_DIRS 2>/dev/null | awk '
  /FOUND/ {
    status="infected";
    sig=substr($0, index($0, ":") + 2);
    sig=substr(sig, 1, length(sig)-6)
    printf("{\"timestamp\":\"%s\",\"path\":\"%s\",\"result\":\"%s\",\"signature\":\"%s\"}\n", strftime("%FT%T%z"), $1, status, sig)
  }
' >> $LOG
```

### 4. `/usr/local/bin/scan_maldet.sh`
```bash
#!/bin/bash
LOG="/var/log/maldet_scan.log"
/usr/local/maldetect/maldet -a / >> $LOG
```

### 5. `/usr/local/bin/scan_rkhunter.sh`
```bash
#!/bin/bash
LOG="/var/log/rkhunter_scan.log"

# Atualiza definições
rkhunter --update

# Roda o scan e converte a saída para JSON
rkhunter --check --sk --rwo --nocolors 2>/dev/null | awk '
BEGIN {
  print "[" > "'"$LOG"'"
}
{
  gsub(/"/, "\\\"", $0);
  printf "  {\"timestamp\": \"%s\", \"message\": \"%s\"},\n", strftime("%FT%T%z"), $0 >> "'"$LOG"'"
}
END {
  print "]" >> "'"$LOG"'"
}
' | sed -i '$s/},/}/' "$LOG"
```

---

## 📁 Script para monitorar modificações com inotify e escanear arquivos modificados

Crie o arquivo `/usr/local/bin/monitor_inotify_clamav.sh` com o conteúdo abaixo:

```bash
#!/bin/bash

WATCH_DIRS="/home /tmp"
LOG="/var/log/clamav/inotify_scan.json"

inotifywait -m -r -e close_write,modify,create,move $WATCH_DIRS --format '%e|%w%f' | while IFS='|' read EVENT FILE
do
    if [[ -f "$FILE" ]]; then
        TIMESTAMP=$(date -Iseconds)
        HOSTNAME=$(hostname)
        freshclam --quiet

        # Executa o scan e captura resultado
        RESULT=$(clamdscan --fdpass --no-summary "$FILE" 2>/dev/null)
        if echo "$RESULT" | grep -q "FOUND"; then
            STATUS="infected"
        else
            STATUS="clean"
        fi

        # Formata como JSON e salva no log
        echo "{\"timestamp\":\"$TIMESTAMP\",\"hostname\":\"$HOSTNAME\",\"event\":\"$EVENT\",\"path\":\"$FILE\",\"result\":\"$STATUS\"}" >> "$LOG"
    fi
done

```

**Dê permissão de execução:**
```bash
chmod +x /usr/local/bin/monitor_inotify_clamav.sh
```

---

## ⚙️ Serviço systemd para o monitoramento inotify

Crie o arquivo `/etc/systemd/system/monitor-inotify-clamav.service` com o conteúdo:

```ini
[Unit]
Description=Monitoramento Inotify para ClamAV (scan em arquivos modificados)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/monitor_inotify_clamav.sh
Restart=always
User=root
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=monitor-inotify-clamav

[Install]
WantedBy=multi-user.target
```

### Comandos para habilitar e iniciar o serviço:

```bash
# Recarregar systemd para reconhecer o novo serviço
systemctl daemon-reload

# Habilitar para iniciar no boot
systemctl enable monitor-inotify-clamav.service

# Iniciar o serviço agora
systemctl start monitor-inotify-clamav.service

# Verificar status
systemctl status monitor-inotify-clamav.service
```

---

## 🕒 Tarefas CRON (`crontab -e` como root)

```cron
# ClamAV: scan a cada hora em /etc, /bin, /sbin
0 * * * * /usr/local/bin/scan_clam_system.sh

# ClamAV: scan completo na raiz 1x por dia, às 2h da manhã
0 2 * * * /usr/local/bin/scan_clam_full.sh

# Maldet: scan completo 1x por dia, às 3h
0 3 * * * /usr/local/bin/scan_maldet.sh

# Rootkit Hunter: auditoria diária às 4h
0 4 * * * /usr/local/bin/scan_rkhunter.sh
```

---

## 🧼 Logrotate (opcional): `/etc/logrotate.d/clamav-maldet-rkhunter`

```conf
/var/log/clamav/*.log /var/log/maldet_scan.log /var/log/rkhunter_scan.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
}
```
### Teste malware:  
```bash
nano /tmp/eicar_test_file
```
```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```
---

## ✅ Resumo da estratégia final

- **Monitoramento em tempo real com `inotify`** para escanear apenas arquivos modificados em `/home` e `/tmp`.
- Scans programados para diretórios críticos e sistema completo em horários definidos para evitar sobrecarga.
- Maldet e RKHunter rodando diariamente para complementar a segurança.
- Logs rotacionados para evitar acúmulo.
- Serviço systemd gerenciando o monitoramento inotify para maior estabilidade.

---

Se quiser, posso ajudar com scripts adicionais para alertas via email ou integração com Telegram.
