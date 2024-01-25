--- 
title: CPU utilization script
date: 2024-01-20
categories: 
tags: [bash]
---

At my current job, I monitor network nodes for CPU and Disk utilization alerts.
If resource usage is over a certain threshold, an alert is sent out to a
support group. I wanted to see if I could implemenet these notifications for
users on a Linux system. Here's what I came up with.

```bash
#!/bin/bash

# Set the threshold for CPU utilization
threshold=80

# Get user-specific CPU utilization using ps and awk
user_cpu=$(ps aux --sort=-%cpu | awk -v user="$USER" '$1==user {print $3;
exit}')

# Compare the user CPU utilization with the threshold
if (( $(echo "$user_cpu > $threshold" | bc -l) )); then
    # Send an email notification using mailx
    echo "Warning: High CPU utilization ($user_cpu%) on user $USER." | mailx -s
"CPU Utilization Alert" user@example.com
fi
```
