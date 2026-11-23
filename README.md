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
