# MU

<pre>

#!/bin/bash

# Cleanup SFs from /opt/exactpro/tools/builds/SF_Farms.xml

cfg_path="$1"
sf_list=$(cat ${cfg_path} | grep 'DeployPath' | sed 's/^.*<DeployPath>//g'  | sed 's/<\/DeployPath>//g')
sfports=$(cat ${cfg_path} | grep 'HTTPPort'   | sed 's/<HTTPPort>//g'       | sed 's/<\/HTTPPort>//g')
sfnames=$(cat ${cfg_path} | grep 'fxcore'      | sed 's/<Name>//g'           | sed 's/<\/Name>//g' | sort | uniq)
sfpaths=$(cat ${cfg_path} | grep 'DeployPath' | sed 's/<DeployPath>//g'     | sed 's/<\/DeployPath>//g')

echo "$sf_list"
echo "$sfports"
echo "$sfnames"
echo "$sfpaths"

# REST API
echo "Cleanup SFs via REST API"
h="localhost" #$(hostname)
weekago=$(date -d "-7days" +%Y-%m-%dT%H:%M:%S.000Z)
stuff_for_delete="reports,matrices,messages,events,trafficdump,logs"
for p in ${sfports}
do
    echo "Port:${p}."
    for n in $sfnames
    do
        echo "Name:${n}."
        cmd="curl -i -X DELETE '${h}:${p}/${n}/sfapi/resources/clean?olderthan=${weekago}&targets=${stuff_for_delete}'"
        echo $cmd
        #eval $cmd
        `$cmd`
    done
done

# Stop SFs
echo "Stop SFs"
for path in $sfpaths
do
    if ! [[ ${path:0:1} == '/' ]]; then path=$(dirname $cfg_path)/${path}; fi
    cmd="cd ${path}/Tomcat/*/bin/ && ./catalina.sh stop 5 -force && cd -"
    eval $cmd
    echo $cmd
done

# Heap-dump cleaup
echo "Cleanup heap-dumps older that 10 days"
AGE=10
for path in $sfpaths
do
    if ! [[ ${path:0:1} == '/' ]]; then path=$(dirname $cfg_path)/${path}; fi
    echo $path
    heap_dumps=$(find $path/Tomcat/*/heap_dumps/ -maxdepth 1 -type f -mtime +${AGE} | sort)
    echo $heap_dumps
    if [[ "${heap_dumps}" ]]
    then
        echo "Heap-dumps found: ${heap_dumps}"
        for h in $heap_dumps
        do
            cmd="rm -f $h"
            eval $cmd
            echo $cmd
        done
    fi
done

# Deletion old reports, that could not removed via REST API
echo "Cleanup old reports older than 7 days"
AGE=7
for path in $sfpaths
do
    if ! [[ ${path:0:1} == '/' ]]; then path=$(dirname $cfg_path)/${path}; fi
    reports=$(find $path/*/user_writable_layer/reports/ -maxdepth 1 -type d -mtime +${AGE} | sort)
    if [[ "${reports}" ]]
    then
        echo "Old reports found: ${reports}"
        for r in $reports
        do
            cmd="rm -rf $r"
            eval $cmd
            echo $cmd
        done
    fi
done

# Start SFs
echo "Start SFs"
for path in $sfpaths
do
    if ! [[ ${path:0:1} == '/' ]]; then path=$(dirname $cfg_path)/${path}; fi
    cmd="cd ${path}/Tomcat/*/bin/ && ./catalina.sh start && cd -"
    eval $cmd
    echo $cmd
    echo "Only first SF has been started, please delete 'break' in script if you want to start both"
    break
done

  </pre>
