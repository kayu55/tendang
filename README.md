````
cat > /usr/bin/tendang << 'EOF'
#!/bin/bash
LOG="/var/log/auth.log"
MAX=2
NOW=$(date +%s)
mkdir -p /tmp/toleransi /tmp/baned

# 1. BUKA BAN YANG UDAH LEWAT 5 MENIT
for F in /tmp/baned/*; do
  [ -f "$F" ] || continue
  U=$(basename "$F")
  OLD=$(cat "$F")
  if [ $((NOW - OLD)) -ge 300 ]; then
    passwd -u "$U" 2>/dev/null
    rm -f /tmp/baned/$U /tmp/toleransi/$U
    echo "$(date +"%Y-%m-%d %X") - $U - BLOKIR DIBUKA" >> /root/log-limit.txt
  fi
done

# 2. HITUNG DROPBEAR
cat $LOG | grep -i dropbear | grep -i "Password auth succeeded" > /tmp/log-db.txt
declare -A jumlah
declare -A pids
proc=( `ps aux | grep -i dropbear | grep -v grep | awk '{print $2}'`)
for PID in "${proc[@]}"; do
  grep "dropbear\[$PID\]" /tmp/log-db.txt > /tmp/log-db-pid.txt
  NUM=$(cat /tmp/log-db-pid.txt | wc -l)
  USER=$(cat /tmp/log-db-pid.txt | awk '{print $10}' | sed "s/'//g")
  if [ "$NUM" -eq 1 ] && [ -n "$USER" ]; then
    jumlah[$USER]=$(( ${jumlah[$USER]:-0} + 1 ))
    pids[$USER]="${pids[$USER]} $PID"
  fi
done
# 3. HITUNG SSHD
cat $LOG | grep -i sshd | grep -i "Accepted password for" > /tmp/log-sshd.txt
data=( `ps aux | grep "\[priv\]" | grep -v grep | awk '{print $2}'`)
for PID in "${data[@]}"; do
  grep "sshd\[$PID\]" /tmp/log-sshd.txt > /tmp/log-sshd-pid.txt
  NUM=$(cat /tmp/log-sshd-pid.txt | wc -l)
  USER=$(cat /tmp/log-sshd-pid.txt | awk '{print $9}')
  if [ "$NUM" -eq 1 ] && [ -n "$USER" ]; then
    jumlah[$USER]=$(( ${jumlah[$USER]:-0} + 1 ))
    pids[$USER]="${pids[$USER]} $PID"
  fi
done

# 4. CEK PELANGGARAN
for USER in "${!jumlah[@]}"; do
  [ -f /tmp/baned/$USER ] && continue
  CNT=${jumlah[$USER]}
  if [ $CNT -gt $MAX ]; then
    if [ ! -f /tmp/toleransi/$USER ]; then
      echo "$NOW" > /tmp/toleransi/$USER
      echo "$(date +"%Y-%m-%d %X") - $USER - $CNT/$MAX - TOLERANSI 1 MENIT, MUNGKIN STOP-START" >> /root/log-limit.txt
    else
      OLD=$(cat /tmp/toleransi/$USER)
      if [ $((NOW - OLD)) -ge 60 ]; then
        echo "$(date +"%Y-%m-%d %X") - $USER - $CNT/$MAX - DIBLOKIR 5 MENIT" >> /root/log-limit.txt
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
echo "*/1 * * * * root /usr/bin/tendang" > /etc/cron.d/tendang
service cron restart 2>/dev/null; systemctl restart cron 2>/dev/null
tendang
echo "OK MAX=2 + TOLERANSI 1 MENIT + BANED 5 MENIT"
````

````
cat /usr/bin/tendang | grep -E "MAX=|DIBLOKIR|TOLERANSI"
cat /root/log-limit.txt | tail -3
````
