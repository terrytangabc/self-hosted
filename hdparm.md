# hdparm usage

### Install
```bash
sudo apt-get install hdparm
```

### Checkout Disk Status
```bash
sudo hdparm -C /dev/sda
```
output: `active/Idle` / `standby`

### Checkout Disk Information
```bash
sudo hdparm -I /dev/sda
```

### Set Disk Status Temporarily
```bash
# standby
sudo hdparm -y /dev/sda

# sleep
sudo hdparm -Y /dev/sda
```

### Edit hdparm Config File(Persistent Configuration)
```bash
# /etc/hdparm.conf
# to see your disk ID:
# ls /dev/disk/by-id
/dev/disk/by-id/ata-TOSHIBA_MD04ABA400V_2818KRSKFMYB {
    # see value meaning below
    spindown_time = 60
}
```

### spindown_time rules
```
0 = disabled
1..240 = multiples of 5 seconds (5 seconds to 20 minutes)
241..251 = 1..11 x 30 mins
252 = 21 mins
253 = vendor defined (8..12 hours)
254 = reserved
255 = 21 mins + 15 secs
```

### reload hdparm conf
```bash
sudo /usr/lib/pm-utils/power.d/95hdparm-apm resume
```
