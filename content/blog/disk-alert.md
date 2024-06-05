+++
title = "Simple disk usage alert"
date = "2024-06-05T16:42:04-04:00"
tags = ["linux","administration",]
+++

My docker host VM was running out of disk of space and I was unaware. I don't want 
this to happen again so I created a basic script to notify me.

The script uses sSMTP as the MTA and I have a free account with smtp2go as my
SMTP service.

The script is below:

```bash
#!/usr/bin/env bash

#/dev/device/by-uuid/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
LV=/dev/mapper/vg--root-lv--var
LIMIT=90
USAGE="$(df -h $LV | grep -v Filesystem | awk '{print $5}' | cut
-d'%' -f1)"

if [ $USAGE -gt $LIMIT ]; then
    ssmtp mail@example.com < ~/disk.txt
else
    exit 0
fi
```
Some notes:
- you can use the UUID if you'd like. I prefer the mapper id for LVM
- change the mail address and supply your own template as input
- add the execute bit with ***chmod +x***
- run this as a cronjob or systemd timer

That's all for now. Until next time.

AG
