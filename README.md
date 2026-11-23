# tendang
````
cat > /usr/bin/tendang << 'EOF'
#!/bin/bash
LOG="/var/log/auth.log"
MAX=2
# BUKA SEMUA BLOKIR LAMA
> /root/blokir.txt
cat /etc/passwd | grep "/home/" | cut -d":" -f1 | grep -v "^root$" | while read U; do passwd -u "$U" 2>/dev/null; done

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

mkdir -p /tmp/toleransi
NOW=$(date +%s)
for USER in "${!jumlah[@]}"; do
  CNT=${jumlah[$USER]}
  if [ $CNT -gt $MAX ]; then
    if [ ! -f /tmp/toleransi/$USER ]; then
      # BARU PERTAMA KALI KELEBIHAN, KASIH TOLERANSI 1 MENIT, JANGAN DIAPA-APAIN
      echo "$NOW" > /tmp/toleransi/$USER
      echo "$(date +"%Y-%m-%d %X") - $USER - $CNT/$MAX - TOLERANSI 1 MENIT, MUNGKIN STOP-START" >> /root/log-limit.txt
    else
      OLD=$(cat /tmp/toleransi/$USER)
      if [ $((NOW - OLD)) -ge 60 ]; then
        # UDAH LEWAT 1 MENIT MASIH KELEBIHAN, BARU TENDANG YG PALING LAMA, TANPA BLOKIR
        LEBIH=$((CNT - MAX))
        echo "$(date +"%Y-%m-%d %X") - $USER - $CNT/$MAX - TENDANG $LEBIH YG LAMA" >> /root/log-limit.txt
        for KPID in $(echo ${pids[$USER]} | tr ' ' '\n' | sort -n | head -n $LEBIH); do
          kill $KPID 2>/dev/null
        done
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
echo "OK MAX=2 + TOLERANSI 1 MENIT, TANPA BLOKIR"
````
````
sed -i '/BUKA SEMUA BLOKIR/,/done/d' /usr/bin/tendang
grep -v "passwd -u" /usr/bin/tendang > /tmp/t && mv /tmp/t /usr/bin/tendang
chmod +x /usr/bin/tendang
tendang
````
````
grep "MAX=" /usr/bin/tendang
cat /root/log-limit.txt | tail -3
````
