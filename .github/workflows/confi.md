
 # Metricas para el detalle de AS400
# */5 * * * * /tmp/dbAs400/as400EMD_30Min.sh > /dev/null 2>&1

*/5 8-22 * * * /tmp/dbAs400/as400EMD_30Min.sh > /dev/null 2>&1
0 23 * * * /tmp/dbAs400/as400EMD_30Min.sh > /dev/null 2>&1


 timedatectl
               Local time: Fri 2026-11-13 15:20:06 UTC
           Universal time: Fri 2026-11-13 15:20:06 UTC
                 RTC time: Fri 2026-11-13 15:20:05
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
[root@F2D005ADGV23 ansible]#
