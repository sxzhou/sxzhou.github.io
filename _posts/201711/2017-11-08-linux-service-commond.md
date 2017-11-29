---
layout: post
title:  "service mysql start"
date:   2017-11-08 23:08:09
categories: linux
tags: linux
author: "sxzhou"
---

今天测试环境mysql停掉了，按经验执行了`service mysql start`，脚本报错`mysql:unrecognized service`，之前一直用`service`这个命令，控制服务的启动停止，查看服务状态。需要研究下它是怎么执行的，`man`一下：
>service  runs a System V init script in as predictable environment as possible, removing most environment variables and with current working directory set to /.  
The SCRIPT parameter specifies a System V init script, located in /etc/init.d/SCRIPT.  The supported values of COMMAND depend on  the  invoked  script,  service passes COMMAND and OPTIONS it to the init script unmodified.  All scripts should support at least the start and stop commands.  As a special case, if COMMAND is --full-restart, the script is run twice, first with the stop command, then with the start command.
service --status-all runs all init scripts, in alphabetical order, with the status command.  

`service`命令是Redhat Linux兼容的发行版中用来控制系统服务的实用工具，它以启动、停止、重新启动和关闭系统服务，还可以显示所有系统服务的当前状态，在CentOS下，`service`命令的路径是`/sbin/service`。  
```shell
#!/bin/sh

. /etc/init.d/functions

VERSION="$(basename $0) ver. 0.91"
USAGE="Usage: $(basename $0) < option > | --status-all | \
[ service_name [ command | --full-restart ] ]"
SERVICE=
SERVICEDIR="/etc/init.d"
OPTIONS=

if [ $# -eq 0 ]; then
   echo "${USAGE}" >&2
   exit 1
fi

cd /
while [ $# -gt 0 ]; do
  case "${1}" in
    --help | -h | --h* )
       echo "${USAGE}" >&2
       exit 0
       ;;
    --version | -V )
       echo "${VERSION}" >&2
       exit 0
       ;;
    *)
       if [ -z "${SERVICE}" -a $# -eq 1 -a "${1}" = "--status-all" ]; then
          cd ${SERVICEDIR}
          for SERVICE in * ; do
            case "${SERVICE}" in
              functions | halt | killall | single| linuxconf| kudzu)
                  ;;
              *)
                if ! is_ignored_file "${SERVICE}" \
		    && [ -x "${SERVICEDIR}/${SERVICE}" ]; then
                  env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" status
                fi
                ;;
            esac
          done
          exit 0
       elif [ $# -eq 2 -a "${2}" = "--full-restart" ]; then
          SERVICE="${1}"
          if [ -x "${SERVICEDIR}/${SERVICE}" ]; then
            env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" stop
            env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" start
            exit $?
          fi
       elif [ -z "${SERVICE}" ]; then
         SERVICE="${1}"
       else
         OPTIONS="${OPTIONS} ${1}"
       fi
       shift
       ;;
   esac
done

if [ -f "${SERVICEDIR}/${SERVICE}" ]; then
   env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" ${OPTIONS}
else
   echo $"${SERVICE}: unrecognized service" >&2
   exit 1
fi

```
`service`命令真正执行的脚本放在`/etc/init.d/`这个目录下，可以看到，这个目录下放置了很多常用服务的控制脚本，`grep`一下`mysql`，发现原来机器上脚本的文件名是`mysql.server`,修改命令名称，解决。