---
title: Android实用技巧之adb命令：am 命令的使用
categories: Android实用技巧
comments: true
tags: [Android, am]
description: 介绍 am 的使用
date: 2015-3-26 10:00:00
---

## 概述

在 adb shell 中，您可以使用 Activity Manager (am) 工具发出命令以执行各种系统操作，如启动 Activity、强行停止进程、广播 intent、修改设备屏幕属性及其他操作。在 shell 中，此语法为：

```
am <command>
```

您也可以直接从 adb 发出 Activity Manager 命令，无需进入远程 shell。例如：

```
adb shell am start -a android.intent.action.VIEW
```

可用的 am 命令：

```
usage: am [subcommand] [options]
usage: am start [-D] [-N] [-W] [-P <FILE>] [--start-profiler <FILE>]
               [--sampling INTERVAL] [-R COUNT] [-S]
               [--track-allocation] [--user <USER_ID> | current] <INTENT>
       am startservice [--user <USER_ID> | current] <INTENT>
       am stopservice [--user <USER_ID> | current] <INTENT>
       am force-stop [--user <USER_ID> | all | current] <PACKAGE>
       am kill [--user <USER_ID> | all | current] <PACKAGE>
       am kill-all
       am broadcast [--user <USER_ID> | all | current] <INTENT>
       am instrument [-r] [-e <NAME> <VALUE>] [-p <FILE>] [-w]
               [--user <USER_ID> | current]
               [--no-window-animation] [--abi <ABI>] <COMPONENT>
       am profile start [--user <USER_ID> current] [--sampling INTERVAL] <PROCESS> <FILE>
       am profile stop [--user <USER_ID> current] [<PROCESS>]
       am dumpheap [--user <USER_ID> current] [-n] <PROCESS> <FILE>
       am set-debug-app [-w] [--persistent] <PACKAGE>
       am clear-debug-app
       am set-watch-heap <PROCESS> <MEM-LIMIT>
       am clear-watch-heap
       am bug-report [--progress]
       am monitor [--gdb <port>]
       am hang [--allow-restart]
       am restart
       am idle-maintenance
       am screen-compat [on|off] <PACKAGE>
       am package-importance <PACKAGE>
       am to-uri [INTENT]
       am to-intent-uri [INTENT]
       am to-app-uri [INTENT]
       am switch-user <USER_ID>
       am start-user <USER_ID>
       am unlock-user <USER_ID> [TOKEN_HEX]
       am stop-user [-w] [-f] <USER_ID>
       am stack start <DISPLAY_ID> <INTENT>
       am stack movetask <TASK_ID> <STACK_ID> [true|false]
       am stack resize <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>
       am stack resize-animated <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>
       am stack resize-docked-stack <LEFT,TOP,RIGHT,BOTTOM> [<TASK_LEFT,TASK_TOP,TASK_RIGHT,TASK_BOTTOM>]
       am stack size-docked-stack-test: <STEP_SIZE> <l|t|r|b> [DELAY_MS]
       am stack move-top-activity-to-pinned-stack: <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>
       am stack positiontask <TASK_ID> <STACK_ID> <POSITION>
       am stack list
       am stack info <STACK_ID>
       am stack remove <STACK_ID>
       am task lock <TASK_ID>
       am task lock stop
       am task resizeable <TASK_ID> [0 (unresizeable) | 1 (crop_windows) | 2 (resizeable) | 3 (resizeable_and_pipable)]
       am task resize <TASK_ID> <LEFT,TOP,RIGHT,BOTTOM>
       am task drag-task-test <TASK_ID> <STEP_SIZE> [DELAY_MS] 
       am task size-task-test <TASK_ID> <STEP_SIZE> [DELAY_MS] 
       am get-config
       am suppress-resize-config-changes <true|false>
       am set-inactive [--user <USER_ID>] <PACKAGE> true|false
       am get-inactive [--user <USER_ID>] <PACKAGE>
       am send-trim-memory [--user <USER_ID>] <PROCESS>
               [HIDDEN|RUNNING_MODERATE|BACKGROUND|RUNNING_LOW|MODERATE|RUNNING_CRITICAL|COMPLETE]
       am get-current-user

am start: start an Activity.  Options are:
    -D: enable debugging
    -N: enable native debugging
    -W: wait for launch to complete
    --start-profiler <FILE>: start profiler and send results to <FILE>
    --sampling INTERVAL: use sample profiling with INTERVAL microseconds
        between samples (use with --start-profiler)
    -P <FILE>: like above, but profiling stops when app goes idle
    -R: repeat the activity launch <COUNT> times.  Prior to each repeat,
        the top activity will be finished.
    -S: force stop the target app before starting the activity
    --track-allocation: enable tracking of object allocations
    --user <USER_ID> | current: Specify which user to run as; if not
        specified then run as the current user.
    --stack <STACK_ID>: Specify into which stack should the activity be put.
am startservice: start a Service.  Options are:
    --user <USER_ID> | current: Specify which user to run as; if not
        specified then run as the current user.

am stopservice: stop a Service.  Options are:
    --user <USER_ID> | current: Specify which user to run as; if not
        specified then run as the current user.

am force-stop: force stop everything associated with <PACKAGE>.
    --user <USER_ID> | all | current: Specify user to force stop;
        all users if not specified.

am kill: Kill all processes associated with <PACKAGE>.  Only kills.
  processes that are safe to kill -- that is, will not impact the user
  experience.
    --user <USER_ID> | all | current: Specify user whose processes to kill;
        all users if not specified.

am kill-all: Kill all background processes.

am broadcast: send a broadcast Intent.  Options are:
    --user <USER_ID> | all | current: Specify which user to send to; if not
        specified then send to all users.
    --receiver-permission <PERMISSION>: Require receiver to hold permission.

am instrument: start an Instrumentation.  Typically this target <COMPONENT>
  is the form <TEST_PACKAGE>/<RUNNER_CLASS> or only <TEST_PACKAGE> if there 
  is only one instrumentation.  Options are:
    -r: print raw results (otherwise decode REPORT_KEY_STREAMRESULT).  Use with
        [-e perf true] to generate raw output for performance measurements.
    -e <NAME> <VALUE>: set argument <NAME> to <VALUE>.  For test runners a
        common form is [-e <testrunner_flag> <value>[,<value>...]].
    -p <FILE>: write profiling data to <FILE>
    -w: wait for instrumentation to finish before returning.  Required for
        test runners.
    --user <USER_ID> | current: Specify user instrumentation runs in;
        current user if not specified.
    --no-window-animation: turn off window animations while running.
    --abi <ABI>: Launch the instrumented process with the selected ABI.
        This assumes that the process supports the selected ABI.

am trace-ipc: Trace IPC transactions.
  start: start tracing IPC transactions.
  stop: stop tracing IPC transactions and dump the results to file.
    --dump-file <FILE>: Specify the file the trace should be dumped to.

am profile: start and stop profiler on a process.  The given <PROCESS> argument
  may be either a process name or pid.  Options are:
    --user <USER_ID> | current: When supplying a process name,
        specify user of process to profile; uses current user if not specified.

am dumpheap: dump the heap of a process.  The given <PROCESS> argument may
  be either a process name or pid.  Options are:
    -n: dump native heap instead of managed heap
    --user <USER_ID> | current: When supplying a process name,
        specify user of process to dump; uses current user if not specified.

am set-debug-app: set application <PACKAGE> to debug.  Options are:
    -w: wait for debugger when application starts
    --persistent: retain this value

am clear-debug-app: clear the previously set-debug-app.

am set-watch-heap: start monitoring pss size of <PROCESS>, if it is at or
    above <HEAP-LIMIT> then a heap dump is collected for the user to report

am clear-watch-heap: clear the previously set-watch-heap.

am bug-report: request bug report generation; will launch a notification
    when done to select where it should be delivered. Options are: 
   --progress: will launch a notification right away to show its progress.

am monitor: start monitoring for crashes or ANRs.
    --gdb: start gdbserv on the given port at crash/ANR

am hang: hang the system.
    --allow-restart: allow watchdog to perform normal system restart

am restart: restart the user-space system.

am idle-maintenance: perform idle maintenance now.

am screen-compat: control screen compatibility mode of <PACKAGE>.

am package-importance: print current importance of <PACKAGE>.

am to-uri: print the given Intent specification as a URI.

am to-intent-uri: print the given Intent specification as an intent: URI.

am to-app-uri: print the given Intent specification as an android-app: URI.

am switch-user: switch to put USER_ID in the foreground, starting
  execution of that user if it is currently stopped.

am start-user: start USER_ID in background if it is currently stopped,
  use switch-user if you want to start the user in foreground.

am stop-user: stop execution of USER_ID, not allowing it to run any
  code until a later explicit start or switch to it.
  -w: wait for stop-user to complete.
  -f: force stop even if there are related users that cannot be stopped.

am stack start: start a new activity on <DISPLAY_ID> using <INTENT>.

am stack movetask: move <TASK_ID> from its current stack to the top (true) or   bottom (false) of <STACK_ID>.

am stack resize: change <STACK_ID> size and position to <LEFT,TOP,RIGHT,BOTTOM>.

am stack resize-docked-stack: change docked stack to <LEFT,TOP,RIGHT,BOTTOM>
   and supplying temporary different task bounds indicated by
   <TASK_LEFT,TOP,RIGHT,BOTTOM>

am stack size-docked-stack-test: test command for sizing docked stack by
   <STEP_SIZE> increments from the side <l>eft, <t>op, <r>ight, or <b>ottom
   applying the optional [DELAY_MS] between each step.

am stack move-top-activity-to-pinned-stack: moves the top activity from
   <STACK_ID> to the pinned stack using <LEFT,TOP,RIGHT,BOTTOM> for the
   bounds of the pinned stack.

am stack positiontask: place <TASK_ID> in <STACK_ID> at <POSITION>
am stack list: list all of the activity stacks and their sizes.

am stack info: display the information about activity stack <STACK_ID>.

am stack remove: remove stack <STACK_ID>.

am task lock: bring <TASK_ID> to the front and don't allow other tasks to run.

am task lock stop: end the current task lock.

am task resizeable: change resizeable mode of <TASK_ID>.
   0 (unresizeable) | 1 (crop_windows) | 2 (resizeable) | 3 (resizeable_and_pipable)

am task resize: makes sure <TASK_ID> is in a stack with the specified bounds.
   Forces the task to be resizeable and creates a stack if no existing stack
   has the specified bounds.

am task drag-task-test: test command for dragging/moving <TASK_ID> by
   <STEP_SIZE> increments around the screen applying the optional [DELAY_MS]
   between each step.

am task size-task-test: test command for sizing <TASK_ID> by <STEP_SIZE>   increments within the screen applying the optional [DELAY_MS] between
   each step.

am get-config: retrieve the configuration and any recent configurations
  of the device.
am suppress-resize-config-changes: suppresses configuration changes due to
  user resizing an activity/task.

am set-inactive: sets the inactive state of an app.

am get-inactive: returns the inactive state of an app.

am send-trim-memory: send a memory trim event to a <PROCESS>.

am get-current-user: returns id of the current foreground user.


<INTENT> specifications include these flags and arguments:
    [-a <ACTION>] [-d <DATA_URI>] [-t <MIME_TYPE>]
    [-c <CATEGORY> [-c <CATEGORY>] ...]
    [-e|--es <EXTRA_KEY> <EXTRA_STRING_VALUE> ...]
    [--esn <EXTRA_KEY> ...]
    [--ez <EXTRA_KEY> <EXTRA_BOOLEAN_VALUE> ...]
    [--ei <EXTRA_KEY> <EXTRA_INT_VALUE> ...]
    [--el <EXTRA_KEY> <EXTRA_LONG_VALUE> ...]
    [--ef <EXTRA_KEY> <EXTRA_FLOAT_VALUE> ...]
    [--eu <EXTRA_KEY> <EXTRA_URI_VALUE> ...]
    [--ecn <EXTRA_KEY> <EXTRA_COMPONENT_NAME_VALUE>]
    [--eia <EXTRA_KEY> <EXTRA_INT_VALUE>[,<EXTRA_INT_VALUE...]]
        (mutiple extras passed as Integer[])
    [--eial <EXTRA_KEY> <EXTRA_INT_VALUE>[,<EXTRA_INT_VALUE...]]
        (mutiple extras passed as List<Integer>)
    [--ela <EXTRA_KEY> <EXTRA_LONG_VALUE>[,<EXTRA_LONG_VALUE...]]
        (mutiple extras passed as Long[])
    [--elal <EXTRA_KEY> <EXTRA_LONG_VALUE>[,<EXTRA_LONG_VALUE...]]
        (mutiple extras passed as List<Long>)
    [--efa <EXTRA_KEY> <EXTRA_FLOAT_VALUE>[,<EXTRA_FLOAT_VALUE...]]
        (mutiple extras passed as Float[])
    [--efal <EXTRA_KEY> <EXTRA_FLOAT_VALUE>[,<EXTRA_FLOAT_VALUE...]]
        (mutiple extras passed as List<Float>)
    [--esa <EXTRA_KEY> <EXTRA_STRING_VALUE>[,<EXTRA_STRING_VALUE...]]
        (mutiple extras passed as String[]; to embed a comma into a string,
         escape it using "\,")
    [--esal <EXTRA_KEY> <EXTRA_STRING_VALUE>[,<EXTRA_STRING_VALUE...]]
        (mutiple extras passed as List<String>; to embed a comma into a string,
         escape it using "\,")
    [--f <FLAG>]
    [--grant-read-uri-permission] [--grant-write-uri-permission]
    [--grant-persistable-uri-permission] [--grant-prefix-uri-permission]
    [--debug-log-resolution] [--exclude-stopped-packages]
    [--include-stopped-packages]
    [--activity-brought-to-front] [--activity-clear-top]
    [--activity-clear-when-task-reset] [--activity-exclude-from-recents]
    [--activity-launched-from-history] [--activity-multiple-task]
    [--activity-no-animation] [--activity-no-history]
    [--activity-no-user-action] [--activity-previous-is-top]
    [--activity-reorder-to-front] [--activity-reset-task-if-needed]
    [--activity-single-top] [--activity-clear-task]
    [--activity-task-on-home]
    [--receiver-registered-only] [--receiver-replace-pending]
    [--receiver-foreground]
    [--selector]
    [<URI> | <PACKAGE> | <COMPONENT>]
```

## Intent 参数规范

由于好几个 `am` 命令都可以带 `Intent` 参数，比如 `am start`、`am broadcast` 等，这里就先介绍一下 Intent 参数规范。
[可以参考官方文档](https://developer.android.com/studio/command-line/adb#IntentSpec)

 - `[-a <ACTION>]`：指定 intent 的 Action（setAction()方法），如“android.intent.action.VIEW”。此指定只能声明一次。 `am start -a android.intent.action.hq.TEST_AM`
 - `[-d <DATA_URI>]`：指定 intent 的 Data（setData()方法），如“content://contacts/people/1”。此指定只能声明一次。
 - `[-t <MIME_TYPE>]`：指定 intent MIME 类型，如“image/png”。此指定只能声明一次。 
 - `[-c <CATEGORY> [-c <CATEGORY>] ...]`：指定 intent 类别（addCategory(String category)），如“android.intent.category.APP_CONTACTS”。 
 - `[-n <COMPONENT>]`：指定带有包名前缀的组件名称以创建显式 intent，如“com.example.heqiang.testsomething/.commontest.OtherTestActivity”。 
 - `[--f <FLAG>]`：将标志添加到intent
 - `[-e|--es <EXTRA_KEY> <EXTRA_STRING_VALUE> ...]`：添加一个 null extra。URI intent 不支持此选项。 `adb shell am start -a android.intent.action.hq.TEST_AM --esn TEST`
 - `[--esn <EXTRA_KEY> ...]`：添加字符串数据作为键值对。 `adb shell am start -a android.intent.action.hq.TEST_AM --es TEST test`
 - `[--ez <EXTRA_KEY> <EXTRA_BOOLEAN_VALUE> ...]`：添加布尔型数据作为键值对。 `adb shell am start -a android.intent.action.hq.TEST_AM --ez TEST true`
 - `[--ei <EXTRA_KEY> <EXTRA_INT_VALUE> ...]`：添加整数型数据作为键值对。 
 - `[--el <EXTRA_KEY> <EXTRA_LONG_VALUE> ...]`：添加长整型数据作为键值对。 
 - `[--ef <EXTRA_KEY> <EXTRA_FLOAT_VALUE> ...]`：添加浮点型数据作为键值对。 
 - `[--eu <EXTRA_KEY> <EXTRA_URI_VALUE> ...]`：添加 URI 数据作为键值对。 
 - `[--ecn <EXTRA_KEY> <EXTRA_COMPONENT_NAME_VALUE>]`：添加组件名称，将其作为 ComponentName 对象进行转换和传递。
 - `[--eia <EXTRA_KEY> <EXTRA_INT_VALUE>[,<EXTRA_INT_VALUE...]]`：添加整数数组， 转换成Integer[]进行传递
 - `[--eial <EXTRA_KEY> <EXTRA_INT_VALUE>[,<EXTRA_INT_VALUE...]]`：添加整数数组，转换成List<Integer>进行传递
 - `[--ela <EXTRA_KEY> <EXTRA_LONG_VALUE>[,<EXTRA_LONG_VALUE...]]`：添加长整型数组，转换成Long[]进行传递
 - `[--elal <EXTRA_KEY> <EXTRA_LONG_VALUE>[,<EXTRA_LONG_VALUE...]]`：添加长整型数组，转换成List<Long>进行传递
 - `[--efa <EXTRA_KEY> <EXTRA_FLOAT_VALUE>[,<EXTRA_FLOAT_VALUE...]]`：添加浮点型数组，转换成Float[]进行传递
 - `[--efal <EXTRA_KEY> <EXTRA_FLOAT_VALUE>[,<EXTRA_FLOAT_VALUE...]]`：添加浮点型数组，转换成List<Float>进行传递
 - `[--esa <EXTRA_KEY> <EXTRA_STRING_VALUE>[,<EXTRA_STRING_VALUE...]]`：添加字符串数组，转换成String[]进行传递`adb shell am start -a android.intent.action.hq.TEST_AM --esa TEST a,b,c`
 - `[--esal <EXTRA_KEY> <EXTRA_STRING_VALUE>[,<EXTRA_STRING_VALUE...]]`：添加字符串数组，转换成List<String>进行传递
 - `[--grant-read-uri-permission]` ：包含标志 FLAG_GRANT_READ_URI_PERMISSION。 
 - `[--grant-write-uri-permission]`：包含标志 FLAG_GRANT_WRITE_URI_PERMISSION。 
 - `[--grant-persistable-uri-permission]`：
 - `[--grant-prefix-uri-permission]`：
 - `[--debug-log-resolution]`：
 - `[--exclude-stopped-packages]`：
 - `[--include-stopped-packages]`：
 - `[--activity-brought-to-front]`：
 - `[--activity-clear-top]`：
 - `[--activity-clear-when-task-reset]`：
 - `[--activity-exclude-from-recents]`：
 - `[--activity-launched-from-history]`：
 - `[--activity-multiple-task]`：
 - `[--activity-no-animation]`：
 - `[--activity-no-history]`：
 - `[--activity-no-user-action]`：
 - `[--activity-previous-is-top]`：
 - `[--activity-reorder-to-front]`：
 - `[--activity-reset-task-if-needed]`：
 - `[--activity-single-top]`：
 - `[--activity-clear-task]`：
 - `[--activity-task-on-home]`：
 - `[--receiver-registered-only]`：
 - `[--receiver-replace-pending]`：
 - `[--receiver-foreground]`：
 - `[--selector]`：
 - `[<URI> | <PACKAGE> | <COMPONENT>]`：如果不受上述某一选项的限制，那么就认为是直接指定 URI、包名和组件名称。当参数不受限制时，如果参数包含一个“:”（冒号），则认为参数是一个 URI；如果参数包含一个“/”（正斜杠），则认为参数是一个组件名称；否则认为参数是一个包名。 
 
## am start

启动 intent 指定的 Activity：

```
am start [-D] [-N] [-W] [-P <FILE>] [--start-profiler <FILE>]
               [--sampling INTERVAL] [-R COUNT] [-S]
               [--track-allocation] [--user <USER_ID> | current] <INTENT>
```

可以使用一下选项：

 - -D：启用调试。
 - -N：启用 Natvie 调试。
 - -W：等待启动完成。 
  ```
  Status: ok
  Activity: com.example.heqiang.testsomething/.commontest.OtherTestActivity
  ThisTime: 131
  TotalTime: 131
  WaitTime: 157
  Complete
  ```
 - --start-profiler <FILE>：启动分析器并将结果发送到 file。 
 - --sampling INTERVAL：制定采样率
 - -P <FILE>：类似于 --start-profiler，但当应用进入空闲状态时分析停止。 
 - -R <COUNT>：重复 Activity 启动 count 次数。在每次重复前，将完成顶部 Activity。 
 - -S：启动 Activity 前强行停止目标应用。 
 - --track-allocation：打开Allocation Tracker用来跟踪内存分配
 - --user <USER_ID> | current：指定要作为哪个用户运行；如果未指定，则作为当前用户运行。 
 - --stack <STACK_ID>：指定将Activity放入的栈

## am startservice

启动 intent 指定的 Service：

```
am startservice [--user <USER_ID> | current] <INTENT>
```

可以使用一下选项：

 - --user <USER_ID> | current：指定要作为哪个用户运行；如果未指定，则作为当前用户运行。 

## am stopservice

停止一个 Service

```
am stopservice [--user <USER_ID> | current] <INTENT>
```

 - --user <USER_ID> | current：指定要作为哪个用户运行；如果未指定，则作为当前用户运行。 

## am force-stop 

```
am force-stop [--user <USER_ID> | all | current] <PACKAGE>
```

强行停止与 package（应用的包名称）关联的所有应用。 

## am kill

终止与 package（应用的包名称）关联的所有进程。此命令仅终止可安全终止且不会影响用户体验的进程。 

```
am kill [--user <USER_ID> | all | current] <PACKAGE>
```

## am kill-all

终止所有后台进程。 

## am broadcast

发出广播 intent。 

```
am broadcast [--user <USER_ID> | all | current] <INTENT>
```

## am stack

 - `am stack start <DISPLAY_ID> <INTENT>`：
 - `am stack movetask <TASK_ID> <STACK_ID> [true|false]`：
 - `am stack resize <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>`：
 - `am stack resize-animated <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>`：
 - `am stack resize-docked-stack <LEFT,TOP,RIGHT,BOTTOM> [<TASK_LEFT,TASK_TOP,TASK_RIGHT,TASK_BOTTOM>]`：
 - `am stack size-docked-stack-test: <STEP_SIZE> <l|t|r|b> [DELAY_MS]`：
 - `am stack move-top-activity-to-pinned-stack: <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>`：
 - `am stack positiontask <TASK_ID> <STACK_ID> <POSITION>`：
 - `am stack list`：
 - `am stack info <STACK_ID>`：
 - `am stack remove <STACK_ID>`：
 - `am task lock <TASK_ID>`：
 - `am task lock stop`：
 - `am task resizeable <TASK_ID> [0 (unresizeable) | 1 (crop_windows) | 2 (resizeable) | 3 (resizeable_and_pipable)]`：
 - `am task resize <TASK_ID> <LEFT,TOP,RIGHT,BOTTOM>`：
 - `am task drag-task-test <TASK_ID> <STEP_SIZE> [DELAY_MS] `：
 - `am task size-task-test <TASK_ID> <STEP_SIZE> [DELAY_MS]`：

## am get-config

## am send-trim-memory

发送回收内存的命令，会调用 Application 和 Activity 的 `onTrimMemory(int level)` 方法。

```
 am send-trim-memory [--user <USER_ID>] <PROCESS>
[HIDDEN|RUNNING_MODERATE|BACKGROUND|RUNNING_LOW|MODERATE|RUNNING_CRITICAL|COMPLETE]
```

```
adb shell am send-trim-memory com.example.heqiang.testsomething RUNNING_CRITICAL
```
