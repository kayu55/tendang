## max 2hp teloransi 5menit blokir 10menit
````
cat > /usr/bin/tendang << 'EOF'
#!/bin/bash
LOG="/var/log/auth.log"
MAX=2
NOW=$(date +%s)
mkdir -p /tmp/toleransi /tmp/baned
for F in /tmp/baned/*; do
  [ -f "$F" ] || continue
  U=$(basename "$F")
  OLD=$(cat "$F")
  if [ $((NOW - OLD)) -ge 600 ]; then
    passwd -u "$U" 2>/dev/null
    rm -f /tmp/baned/$U /tmp/toleransi/$U
    echo "$(date +"%Y-%m-%d %X") - $U - BLOKIR DIBUKA" >> /root/log-limit.txt
  fi
done
declare -A ips
declare -A pids
cat $LOG | grep -i dropbear | grep -i "Password auth succeeded" > /tmp/log-db.txt
for PID in $(ps aux | grep -i dropbear | grep -v grep | awk '{print $2}'); do
  grep "dropbear\[$PID\]" /tmp/log-db.txt > /tmp/log-db-pid.txt
  NUM=$(cat /tmp/log-db-pid.txt | wc -l)
  USER=$(cat /tmp/log-db-pid.txt | awk '{print $10}' | sed "s/'//g")
  IP=$(cat /tmp/log-db-pid.txt | awk '{print $12}' | sed "s/'//g")
  if [ "$NUM" -eq 1 ] && [ -n "$USER" ] && [ -n "$IP" ]; then
    ips[$USER]="${ips[$USER]} $IP"
    pids[$USER]="${pids[$USER]} $PID"
  fi
done
cat $LOG | grep -i sshd | grep -i "Accepted password for" > /tmp/log-sshd.txt
for PID in $(ps aux | grep "\[priv\]" | grep -v grep | awk '{print $2}'); do
  grep "sshd\[$PID\]" /tmp/log-sshd.txt > /tmp/log-sshd-pid.txt
  NUM=$(cat /tmp/log-sshd-pid.txt | wc -l)
  USER=$(cat /tmp/log-sshd-pid.txt | awk '{print $9}')
  IP=$(cat /tmp/log-sshd-pid.txt | awk '{print $11}')
  if [ "$NUM" -eq 1 ] && [ -n "$USER" ] && [ -n "$IP" ]; then
    ips[$USER]="${ips[$USER]} $IP"
    pids[$USER]="${pids[$USER]} $PID"
  fi
done
for USER in "${!ips[@]}"; do
  [ -f /tmp/baned/$USER ] && continue
  CNT=$(echo ${ips[$USER]} | tr ' ' '\n' | sort -u | grep -v "^$" | wc -l)
  if [ $CNT -gt $MAX ]; then
    if [ ! -f /tmp/toleransi/$USER ]; then
      echo "$NOW" > /tmp/toleransi/$USER
      echo "$(date +"%Y-%m-%d %X") - $USER - $CNT/$MAX IP - TOLERANSI 5 MENIT" >> /root/log-limit.txt
    else
      OLD=$(cat /tmp/toleransi/$USER)
      if [ $((NOW - OLD)) -ge 300 ]; then
        echo "$(date +"%Y-%m-%d %X") - $USER - $CNT/$MAX IP - DIBLOKIR 10 MENIT" >> /root/log-limit.txt
        for KPID in ${pids[$USER]}; do kill $KPID 2>/dev/null; done
        passwd -l "$USER" 2>/dev/null
        echo "$NOW" > /tmp/baned/$USER
        rm -f /tmp/toleransi/$USER
      fi
    fi
  else
    rm -f /tmp/toleransi/$USER
  fi
done
EOF
chmod +x /usr/bin/tendang
grep -q "tendang" /etc/cron.d/tendang 2>/dev/null || echo "*/1 * * * * root /usr/bin/tendang" > /etc/cron.d/tendang
for U in $(ls /tmp/baned/ 2>/dev/null); do passwd -u $U 2>/dev/null; done
rm -f /tmp/toleransi/* /tmp/baned/*
service cron restart
tendang
echo "=== CEK FINAL ==="
grep -n "TOLERANSI\|DIBLOKIR\|-ge" /usr/bin/tendang
cat /etc/cron.d/tendang
service cron status | grep Active
````
## cek status
````
cat /root/log-limit.txt; echo "---"; ls -l /usr/bin/tendang /etc/cron.d/tendang
````
## cek user kna baned
````
cat > /usr/bin/cek-blok << 'EOF'
#!/bin/bash
echo "================ CEK BLOK UNIVERSAL ================"
echo ""
echo "--- 1. TENDANG ---"
if [ "$(ls -A /tmp/baned 2>/dev/null)" ]; then
  ls /tmp/baned/ | while read U; do echo "❌ $U - KE BLOK TENDANG"; done
else
  echo "✅ Gak enek sing ke blok tendang"
fi
echo ""
echo "--- 2. SSH KE KUNCI (L) ---"
for U in $(awk -F: '$3 >= 1000 && $1 != "nobody" {print $1}' /etc/passwd); do
  ST=$(passwd -S $U 2>/dev/null | awk '{print $2}')
  if [ "$ST" = "L" ]; then echo "❌ SSH $U - KE KUNCI"; fi
done
echo "Selesai cek SSH..."
echo ""
echo "--- 3. XRAY EXPIRED ---"
TODAY=$(date +%Y-%m-%d)
grep "^### " /etc/xray/config.json 2>/dev/null | while read -r _ USER EXP; do
  if [[ "$EXP" < "$TODAY" ]]; then echo "❌ XRAY $USER - EXPIRED $EXP"; fi
done
echo "Selesai cek XRAY..."
echo ""
echo "--- 4. LOG TENDANG ---"
tail -10 /root/log-limit.txt 2>/dev/null || echo "Log kosong"
echo "====================================================="
EOF
chmod +x /usr/bin/cek-blok
cek-blok
````
## offkan
````
rm -f /etc/cron.d/tendang /usr/bin/tendang
for U in $(ls /tmp/baned/ 2>/dev/null); do passwd -u $U 2>/dev/null; done
rm -rf /tmp/toleransi/* /tmp/baned/*
service cron restart
echo "TENDANG OFF TOTAL"
````
## cek
````
cat /etc/cron.d/tendang; ls -l /usr/bin/tendang
````
### buka blok
````
passwd -u najib
rm -f /tmp/baned/NAMAUSER /tmp/toleransi/NAMAUSER
echo "USER NAMAUSER WES DIBUKA MANUAL" >> /root/log-limit.txt
````

### 1. Pasang kunci ne (copy paste iki)
# 1. Buka dulu najib nek isih ke blok
````
passwd -u najib
rm -f /tmp/baned/najib /tmp/toleransi/najib

# 2. Pasang whitelist neng tendang
cp /usr/bin/tendang /usr/bin/tendang.bak
python3 << 'PY'
p="/usr/bin/tendang"
t=open(p).read()
# Tambah whitelist neng nduwur
if "WHITELIST" not in t:
    t=t.replace('MAX=2', 'MAX=2\nWHITELIST="najib"')
# Tambah skip neng loop user
if "WHITELIST" in t and "skip whitelist" not in t.lower():
    t=t.replace('for USER in "${!ips[@]}"; do', 'for USER in "${!ips[@]}"; do\n  # WHITELIST - SKIP USER ISTIMEWA\n  if [[ " $WHITELIST " == *" $USER "* ]]; then continue; fi')
    open(p,'w').write(t)
    print("OK - najib wes dikunci!")
else:
    print("Wes enek whitelist, cek manual!")
PY

chmod +x /usr/bin/tendang
grep -n "WHITELIST\|najib" /usr/bin/tendang
````
### 2. Cek hasil e

Nek metu ne koyo iki = *BERHASIL!*
WHITELIST="najib"
# WHITELIST - SKIP USER ISTIMEWA
if [[ " $WHITELIST " == *" $USER "* ]]; then continue; fi
### max 50 user blok 10mnit tnpa teloransi
````
cat > /usr/bin/tendang << 'EOF'
#!/bin/bash
LOG="/var/log/auth.log"
MAX=50
WHITELIST="najib"
NOW=$(date +%s)
mkdir -p /tmp/baned
# BUKA BLOKIR SING WES 10 MENIT (600 DETIK)
for F in /tmp/baned/*; do
  [ -f "$F" ] || continue
  U=$(basename "$F")
  OLD=$(cat "$F")
  if [ $((NOW - OLD)) -ge 600 ]; then
    passwd -u "$U" 2>/dev/null
    rm -f /tmp/baned/$U
    echo "$(date +"%Y-%m-%d %X") - $U - BLOKIR DIBUKA (10 MENIT HABIS)" >> /root/log-limit.txt
  fi
done
declare -A ips
declare -A pids
cat $LOG | grep -i dropbear | grep -i "Password auth succeeded" > /tmp/log-db.txt
for PID in $(ps aux | grep -i dropbear | grep -v grep | awk '{print $2}'); do
  grep "dropbear\[$PID\]" /tmp/log-db.txt > /tmp/log-db-pid.txt
  NUM=$(cat /tmp/log-db-pid.txt | wc -l)
  USER=$(cat /tmp/log-db-pid.txt | awk '{print $10}' | sed "s/'//g")
  IP=$(cat /tmp/log-db-pid.txt | awk '{print $12}' | sed "s/'//g")
  if [ "$NUM" -eq 1 ] && [ -n "$USER" ] && [ -n "$IP" ]; then
    ips[$USER]="${ips[$USER]} $IP"
    pids[$USER]="${pids[$USER]} $PID"
  fi
done
cat $LOG | grep -i sshd | grep -i "Accepted password for" > /tmp/log-sshd.txt
for PID in $(ps aux | grep "\[priv\]" | grep -v grep | awk '{print $2}'); do
  grep "sshd\[$PID\]" /tmp/log-sshd.txt > /tmp/log-sshd-pid.txt
  NUM=$(cat /tmp/log-sshd-pid.txt | wc -l)
  USER=$(cat /tmp/log-sshd-pid.txt | awk '{print $9}')
  IP=$(cat /tmp/log-sshd-pid.txt | awk '{print $11}')
  if [ "$NUM" -eq 1 ] && [ -n "$USER" ] && [ -n "$IP" ]; then
    ips[$USER]="${ips[$USER]} $IP"
    pids[$USER]="${pids[$USER]} $PID"
  fi
done
for USER in "${!ips[@]}"; do
  # WHITELIST - NAJIB GAK KENE TENDANG
  if [[ " $WHITELIST " == *" $USER "* ]]; then continue; fi
  [ -f /tmp/baned/$USER ] && continue
  CNT=$(echo ${ips[$USER]} | tr ' ' '\n' | sort -u | grep -v "^$" | wc -l)
  # TANPA TOLERANSI - LANGSUNG BLOKIR NEK > 50
  if [ $CNT -gt $MAX ]; then
    echo "$(date +"%Y-%m-%d %X") - $USER - $CNT/$MAX IP - DIBLOKIR 10 MENIT (TANPA TOLERANSI)" >> /root/log-limit.txt
    for KPID in ${pids[$USER]}; do kill $KPID 2>/dev/null; done
    passwd -l "$USER" 2>/dev/null
    echo "$NOW" > /tmp/baned/$USER
  fi
done
EOF
chmod +x /usr/bin/tendang
grep -q "tendang" /etc/cron.d/tendang 2>/dev/null || echo "*/1 * * * * root /usr/bin/tendang" > /etc/cron.d/tendang
for U in $(ls /tmp/baned/ 2>/dev/null); do passwd -u $U 2>/dev/null; done
rm -f /tmp/toleransi/* /tmp/baned/*
service cron restart
tendang
echo "=== CEK FINAL ==="
grep -n "MAX=\|WHITELIST\|DIBLOKIR\|TOLERANSI" /usr/bin/tendang
cat /etc/cron.d/tendang
service cron status | grep Active
````
