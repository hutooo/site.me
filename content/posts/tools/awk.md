---
title: Linux文本三剑客之 --- AWK
author: ash
tags: ["linux", "awk", "unix"]
categories: ["UNIX哲学"]
date: 2021-06-24T18:55:01+08:00
image: touhou03.jpg
---

```sh
tmp_path="/mnt/sensetime/host00/tmp/"

removeExpireFiles()
{
    local cmdres=$(ls -e ${tmp_path} | awk 'BEGIN {
            M["Jan"] = 1; M["Feb"] = 2; M["Mar"] = 3; M["Apr"] = 4;
            M["May"] = 5; M["Jun"] = 6; M["Jul"] = 7; M["Aug"] = 8; 
            M["Sep"] = 9; M["Oct"] = 10; M["Nov"] = 11; M["Dec"] = 12;
            now = systime();
            yesterday = now - 86400;}
    NR > 1 {
        datestr=sprintf("%s %s %s %s %s %s", $10, M[$7], $8, substr($9,1,2), substr($9,4,2), substr($9,7,2));
        date=mktime(datestr);
        if (date < yesterday) {
            printf("%s\n", $NF);
        };
    }')
    for file in ${cmdres}
    do
        # echo ${file}
        $(rm -rf ${tmp_path}${file})
    done
}

runMonitor()
{
    while true
    do
        local duration=$(awk 'BEGIN {h=strftime("%H", systime()); print ((5-h) > 0)?(5-h)*3600:0}')
        if [ ${duration} -ne 0 ]; then
            removeExpireFiles
            # echo ${duration}
            $(sleep ${duration})
        fi
        # echo "sleep 10"
        $(sleep 3600)
    done
}

test_start()
{
    $(pidof tmpPathMonitor.sh || nohup ./shell/tmpPathMonitor.sh > /dev/null 2>&1 &)
}

test_stop()
{
    $(ps | grep "sleep 3600" | grep -v grep | awk '{print $1}' | xargs kill > /dev/null 2>&1)
    pidof tmpPathMonitor.sh && pidof tmpPathMonitor.sh | xargs kill
}
```