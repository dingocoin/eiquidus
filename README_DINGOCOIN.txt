


Index
=================

- Origin
- Changes specific to DINGOCOIN
- Metadata "author"
- Scripts
- Start script
- Stop script
- Update Top 100 script
- Update Peers script
- Update Transactions script
- Rescan and flatten the tx count script
- Compare databases script
- Prepare for db sync script
- Logrotate





Origin
====================

eIquidus explorer is forked from:

	https://github.com/team-exor/eiquidus




Changes specific to DINGOCOIN
=====================================

1. Recommended config: dingocoin ready settings.json.template - default social images in footer (insert users, passwords, database name at the top).
2. Config with better social media images. Dingocoin ready settings.json.with_logos.template (insert users, passwords, database name at the top).
   * Social images and configuration are ready to use. 
3. Added Discord, X, Github and Telegram footer images to public/img (please see 2.).
4. Added basic SEO meta data into the head: title, author and keywords. How-to add more meta data is described in settings.json as a comment.
5. Background of Richlist pie set to transparent, grid: {borderWidth:0, shadow:false, background:'transparent'} in views/richlist.pug.
6. Dingocoin images everywhere.
7. Dingocoin screenshot.
8. Added robots.txt and sitemap.xml.
9. Added the Dingocoin theme public/css/themes/dingo/.
10. Dingocoin specific changes are described in file README_DINGOCOIN.txt.
 



Metadata "author"
=====================

Metadata "author" (head) is "Various authors listed in file LICENSE (located on code root). Dingocoin specific settings added by Dingocoin Project.".




Scripts
==================

Few scripts to handle tasks. Please remove comments before use.




Start script
================

Start script is to be executed as a unprivileged user at "init multuiser" time, rc.local style. The "sudo sync" command is recommended after execution.
It disables cron updates, starts dingocoind and eiquidus. Last 100 blocks in database are checked. Cron updates are enabled at the end.

logfile /home/user/eiquidus_start.log:
su - user -c "/usr/local/bin/rc.eiqui | tee ~/eiquidus_start.log"

########################################################################< remove before use>###
#!/bin/bash

# Start script is to be executed as a unprivileged user at "init multuiser" time, rc.local style. The "sudo sync" command is recommended after execution.
# It disables cron updates, starts dingocoind and eiquidus. Last 50 blocks in database are checked. Cron updates are enabled at the end.

# logfile /home/user/eiquidus_start.log:
# su - user -c "/usr/local/bin/rc.eiqui | tee ~/eiquidus_start.log"

# disable cron db updates and wait tasks to stop

if [[ -f "/dev/shm/canupdate.txt" ]] ; then
   echo -e "$( date +"%F %H:%M:%S" ) \t disable cron db updates"
   rm -f /dev/shm/canupdate.txt
   sleep 30
fi

# start daemon

echo -e "$( date +"%F %H:%M:%S" ) \t dingocoind start"
/home/user/Downloads/dingo/dingocoind

# wait daemon sync

p=0
b=0
pb=0
while [[ $pb -lt 2 ]]
do
   echo -e "$( date +"%F %H:%M:%S" ) \t wait dingocoind sync"
   sleep 5
   p=$( ps -ef | grep -v grep | grep dingocoind | wc -l )
   b=$( tail -n 1 /home/user/.dingocoin/debug.log | grep "new best=" | grep "\(1.000000\|0.999999\|0.999998\|0.999997\|0.999996\|0.999995\)" | wc -l )
   pb=$(( $p+$b ))
done

# start eiquidus

echo -e "$( date +"%F %H:%M:%S" ) \t eiquidus start"
cd /home/user/E_iquidus/ &&
rm -f /home/user/E_iquidus/tmp/* &&
#screen -S Enpm -d -m /usr/bin/npm start
/usr/bin/npm run prestart "pm2" &&
/usr/bin/pm2 start ./bin/instance -i 0 -n explorer -p "./tmp/pm2.pid" --node-args="--stack-size=10000"

# wait eiquidus

t=0
e=0
while [[ $t -eq 0 ]] || [[ $e -eq 0 ]]
do
   echo -e "$( date +"%F %H:%M:%S" ) \t eiquidus start"
   sleep 5
   e=$( ps -ef | grep -v grep | grep "PM2 v" | wc -l )
   t=$( ps -ef | grep -v grep | grep "node \/home\/" | wc -l )
   et=$(( $e+$t ))
done

# last block in db

l=0
l=$( mongosh EIxplorerdb --eval "db.coinstats.find()" | grep "last\:" | grep -v "last\: 0," | sed "1,$ s/^\(.*\)last\: \(.*\),/\2/g" )
echo -e "$( date +"%F %H:%M:%S" ) \t last block is $l"

# check last 100 blocks in db
# don't check when db is empty
# sync empty db 

if [[ $l != "" ]] && [[ $l -gt 200 ]] ; then
   n=$(( $l-100 ))
   echo -e "$( date +"%F %H:%M:%S" ) \t db check since block $n"
   cd /home/user/E_iquidus &&
   /usr/bin/node --stack-size=2048 scripts/sync.js index check $n
else
   echo -e "$( date +"%F %H:%M:%S" ) \t empty db, sync next ..."
   cd /home/user/E_iquidus &&
   /usr/bin/node --stack-size=2048 scripts/sync.js index update > /dev/null 2>&1
fi

# finish sync to the last block before going online

echo -e "$( date +"%F %H:%M:%S" ) \t sync to the last block before going online"
cd /home/user/E_iquidus &&
/usr/bin/node --stack-size=2048 scripts/sync.js index update

# become ready for cron executed scripts

l=0
l=$( mongosh EIxplorerdb --eval "db.coinstats.find()" | grep "last\:" | grep -v "last\: 0," | sed "1,$ s/^\(.*\)last\: \(.*\),/\2/g" )
echo $l > /dev/shm/canupdate.txt
echo -e -n "$( date +"%F %H:%M:%S" ) \t cron control since block "

if [[ -f "/dev/shm/canupdate.txt" ]] ; then
   cat /dev/shm/canupdate.txt
else
   echo "... ERROR"
fi

exit 0
########################################################################< remove before use>###




Stop script
===============

Stop is to be executed as an unprivileged user at "exit multiuser" time, rc.local-shutdown style. The "sudo sync" command is recommended after execution.
It disables cron updates at the beginning, stops eiquidus and dingocoind.

logfile /home/user/eiquidus_stop.log:
su - user -c "/usr/local/bin/rc.eiqui_down  | tee ~/eiquidus_stop.log"

########################################################################< remove before use>###
#!/bin/bash

# Stop is to be executed as an unprivileged user at "exit multiuser" time, rc.local-shutdown style. The "sudo sync" command is recommended after execution.
# It disables cron updates at the beginning, stops eiquidus and dingocoind.

# logfile /home/user/eiquidus_stop.log:
# su - user -c "/usr/local/bin/rc.eiqui_down  | tee ~/eiquidus_stop.log"

# disable cron db updates and wait tasks to stop

if [[ -f "/dev/shm/canupdate.txt" ]] ; then
   echo -e "$( date +"%F %H:%M:%S" ) \t disable cron db updates"
   rm -f /dev/shm/canupdate.txt
   sleep 10
fi

# stop eiquidus

echo -e "$( date +"%F %H:%M:%S" ) \t eiquidus stop"
cd /home/user/E_iquidus/ &&
/usr/bin/pm2 stop explorer

# wait eiquidus stop

t=1
while [[ $t -gt 0 ]]
do
   echo -e "$( date +"%F %H:%M:%S" ) \t wait eiquidus stop"
   sleep 5
   t=$( ps -ef | grep -v grep | grep "node \/home\/" | wc -l )
done

# stop daemon

echo -e "$( date +"%F %H:%M:%S" ) \t dingocoind stop"
/home/user/Downloads/dingo/dingocoin-cli -conf=dingocoin.conf stop

# wait daemon stop

pb=1
while [[ $pb -gt 0 ]]
do
   echo -e "$( date +"%F %H:%M:%S" ) \t wait dingocoind stop"
   sleep 5
   pb=$( ps -ef | grep -v grep | grep dingocoind | wc -l )
done

exit 0
########################################################################< remove before use>###




Update Top 100 sudo and regular version script
==================================================

########################################################################< remove before use>###
#!/bin/bash

if [[ -f "/dev/shm/canupdate.txt" ]] ; then
   #su - user -c "cd /home/user/E_iquidus && /usr/bin/npm run reindex-rich" > /dev/null 2>&1
   cd /home/user/E_iquidus && /usr/bin/npm run reindex-rich
fi

exit 0
########################################################################< remove before use>###




Update Peers sudo and regular version script
==================================================

########################################################################< remove before use>###
#!/bin/bash

if [[ -f "/dev/shm/canupdate.txt" ]] ; then
   #su - user -c "cd /home/user/E_iquidus && /usr/bin/npm run sync-peers" > /dev/null 2>&1
   cd /home/user/E_iquidus && /usr/bin/npm run sync-peers
fi

exit 0
########################################################################< remove before use>###




Update Transactions regular user version script
=======================================================

Update Transactions script is to be executed only as an unprivileged user. It's to be cron executed every minute, the "n=$(( $l-5 ))" is meant for 1min update intervals.

########################################################################< remove before use>###
#!/bin/bash

if [[ -f "/dev/shm/canupdate.txt" ]] ; then
   cd /home/user/E_iquidus && /usr/bin/node --stack-size=2048 scripts/sync.js index update

   # last block in db

   l=0
   l=$( mongosh explorerdb --eval "db.coinstats.find()" | grep "last\:" | grep -v "last\: 0," | sed "1,$ s/^\(.*\)last\: \(.*\),/\2/g" )

   # check last 5 blocks in db

   n=$(( $l-5 ))
   cd /home/user/E_iquidus &&
   /usr/bin/node --stack-size=2048 scripts/sync.js index check $n
fi

exit 0
########################################################################< remove before use>###




Rescan and flatten the tx count value for faster access, sudo and regular version script
=============================================================================================

########################################################################< remove before use>###
#!/bin/bash

if [[ -f "/dev/shm/canupdate.txt" ]] ; then
   su - user -c "cd /home/user/E_iquidus && /usr/bin/npm run reindex-txcount" > /dev/null 2>&1
fi

exit 0
########################################################################< remove before use>###




Compare databases script
============================

Run it as a regular unprivileged user. Compares richlists of two members of cluster. Script would be different in case of more than two cluster members. File dorepair.txt in /dev/shm is a sign of errors.

logfile /home/user/db_sync_check.log

########################################################################< remove before use>###
#!/bin/bash

# ten tests of richlists, 1/minute. more than 7 errors create /dev/shm/dorepeair.txt file.
# this file is a sign to correct db.

echo -e "" >> ~/db_sync_check.log
echo -e "$( date +"%F %H:%M:%S" ) \t compare databases start" >> ~/db_sync_check.log

a=""
b=""
s1=404
s2=404
sum=0

for i in 1 2 3 4 5 6 7 8 9 10 ; do
   a=""
   b=""
   s1=404
   s2=404
   a=$( curl -sw ' STATUS_CODE=%{http_code}' https://unitAexplorer.dingocoin.com/ext/getdistribution )
   b=$( curl -sw ' STATUS_CODE=%{http_code}' https://unitBexplorer.dingocoin.com/ext/getdistribution )
   s1=$( echo $a | sed "1,$ s/^\(.*\) STATUS_CODE=\(.*\)$/\2/" )
   s2=$( echo $b | sed "1,$ s/^\(.*\) STATUS_CODE=\(.*\)$/\2/" )

   echo -e "$( date +"%F %H:%M:%S" ) \t a ... $a" >> ~/db_sync_check.log
   echo -e "$( date +"%F %H:%M:%S" ) \t b ... $b" >> ~/db_sync_check.log
   echo -e "$( date +"%F %H:%M:%S" ) \t status1 ... $s1" >> ~/db_sync_check.log
   echo -e "$( date +"%F %H:%M:%S" ) \t status2 ... $s2" >> ~/db_sync_check.log

   # http status 200 only accepted as regular result

   if [[ $a != $b ]] && [[ $s1 -eq 200 ]] && [[ $s2 -eq 200 ]] ; then
      sum=$(( $sum + 1 ))
   else
      sum=$(( $sum + 0 ))
   fi

   echo -e "$( date +"%F %H:%M:%S" ) \t $i ... sum = $sum" >> ~/db_sync_check.log

   sleep 59
done

# create the dorepair.txt file in case of over 7 errors

if [[ $sum -gt 7 ]] ; then
   echo -e "$( date +"%F %H:%M:%S" ) \t dorepair.txt" >> ~/db_sync_check.log
   date > /dev/shm/dorepeair.txt
else
   echo -e "$( date +"%F %H:%M:%S" ) \t databases are equal, no need to repair" >> ~/db_sync_check.log
fi

exit 0
########################################################################< remove before use>###




Prepare for db sync script
===============================

This script is run by cron. Database is cleaned and recreated when needed.

########################################################################< remove before use>###
#!/bin/bash

# compare richlists and repair db if needed
# observed: /dev/shm/dorepeair.txt

if [[ -f "/dev/shm/dorepeair.txt" ]] ; then
   echo -e "$( date +"%F %H:%M:%S" ) \t clean old file dorepair.txt (should be missing ...)"
   rm -f /dev/shm/dorepeair.txt
fi

# check richlists

echo -e "$( date +"%F %H:%M:%S" ) \t compare databases"
su - user -c "/path-to-script/rc.check_errors"

# stop cron updates
# stop web server
# recreate clean db and reboot

if [[ -f "/dev/shm/dorepeair.txt" ]] ; then
   echo -e "$( date +"%F %H:%M:%S" ) \t disable cron db updates and stop web server"
   rm -f /dev/shm/canupdate.txt
   systemctl stop nginx
   sleep 120
   # delete database and create new empty one
   su - user -c "cd /home/user/E_iquidus && echo 'y' | npm run delete-database"
   # reboot
   sync
   reboot
fi

# db check finished

echo -e "$( date +"%F %H:%M:%S" ) \t equal richlists, no need to resync"

exit 0
########################################################################< remove before use>###




Logrotate
===============

File is in /etc/logrotate.d/ .
Location of logs is a homedir of regular user.

########################################################################< remove before use>###
/home/user/db_sync_check.log /home/user/eiquidus_start.log /home/user/eiquidus_stop.log {
       weekly
       rotate 5
       delaycompress
       compress
       notifempty
       missingok
       copytruncate
}
########################################################################< remove before use>###
