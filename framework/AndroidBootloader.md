kernel/common/init/main.c
system/core/init/main.cpp
system/core/init/first_stage_init.cpp

FirstStageMain

SetupSelinux 最小权限原则

SecondStageMain
1. 清理僵尸进程
2. 解析init.rc
3. 进入循环等待

system/core/rootdir/init.rc

on zygote-start
  start zygote

system/core/rootdir/init.zygote32.rc
system/core/rootdir/init.zygote64.rc

frameworks/base/cmds/app_process/app_main.cpp

AppRuntime AndroidRuntime