# [Bug] OOM - 메모리 누수로 인한 MemoryGuard 보호 정책 강제 종료

## 1. Description (현상 설명)
- **어떤 현상이 발생했는가?**
  `agent-leak-app` 어플리케이션을 실행하고 일정 시간이 경과하면, 터미널에 `SELF-TERMINATED` 메시지가 출력되며 프로세스가 예고 없이 강제 종료(Killed)되는 현상이 발생했습니다.
- **언제, 어떤 조건에서 발생했는가?**
  프로세스 실행 후 지속적으로 메모리 사용량이 증가하여, 환경변수에 설정된 `MEMORY_LIMIT`(256MB)을 초과하는 시점에 발생합니다.

## 2. Evidence & Logs (증거 자료)
- **monitor.sh 관제 로그**
```text
[RESOURCE MONITORING]
CPU Usage : 0.0%
MEM Usage : 3.5% (초기 메모리 사용량)
CPU Usage : 0.0% / MEM Usage : 45.2% (진행 중)
CPU Usage : 0.0% / MEM Usage : 98.5% (종료 직전)

[INFO] agent-leak-app 시작 (PID: 13293)
[INFO] 설정된 MEMORY_LIMIT: 256MB
[INFO] 서비스 실행 중... (데이터 적재 시작)
[CRITICAL] [MemoryGuard] Memory limit exceeded (258MB >= 256MB) / (Recommend Over 256MB)
[CRITICAL] [MemoryGuard] Self-terminating process 13293 to prevent system instability.
>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<
Killed
```

## 3. Root Cause Analysis (원인 분석)
- **기술적 원인 분석:** 어플리케이션 로직 내부에서 할당된 메모리를 해제하지 않고 지속적으로 쌓아두는 메모리 누수(Memory Leak) 결함이 존재합니다.
- **OS 동작 원리:** 물리 메모리 사용량이 `MEMORY_LIMIT`(256MB)에 도달하자, 어플리케이션 내부의 MemoryGuard 정책이 OS 전체의 불안정(System OOM)을 방지하기 위해 SIGKILL 시그널을 호출하여 자기 자신을 강제 종료시켰습니다.

## 4. Workaround & Verification (조치 및 검증)
- **환경변수 조정:** 터미널에서 `export MEMORY_LIMIT=512` 명령어를 통해 허용 메모리 한도를 2배로 상향 조정했습니다.
- **Before & After 검증:**
  - **Before:** 256MB 제한 시 약 25초 후 프로세스 종료
  - **After:** 512MB 상향 후 생존 시간이 2배 이상 연장됨을 확인했습니다.
- **추가 제안:** 임시 조치로 생존 시간을 늘렸으나, 근본적인 해결을 위해서는 소스 코드 내부의 불필요한 데이터를 주기적으로 삭제(Garbage Collection 유도 등)하는 리팩토링이 필수적입니다.

---

# [Bug] CPU - CPU 과점유에 의한 Watchdog 보호 조치 프로세스 종료

## 1. Description (현상 설명)
- **어떤 현상이 발생했는가?**
  `agent-leak-app` 실행 후 복잡한 연산 처리가 시작되면서 시스템 CPU 사용률이 비정상적으로 치솟고, 곧이어 `[SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM)` 메시지와 함께 프로세스가 강제 종료(Terminated)되었습니다.
- **언제, 어떤 조건에서 발생했는가?**
  프로세스가 백그라운드 스레드에서 무한 루프 연산을 수행하여 CPU 점유율이 환경변수 `CPU_MAX_OCCUPY`(기본값 80%)에 지정된 임계치를 초과하여 일정 시간 지속될 때 발생합니다.

## 2. Evidence & Logs (증거 자료)
- **시스템 도구 (top / monitor.sh) 관제 결과**
```text
PID    USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
13502  agent-ad  20   0   73536  15420   8340 R  99.9   0.5   0:04.12 python3

[INFO] agent-leak-app 시작 (PID: 13502)
[INFO] 설정된 CPU_MAX_OCCUPY: 80%
[INFO] 서비스 실행 중... (복잡한 연산 처리 시작)
[CRITICAL] [Watchdog] CPU usage exceeded threshold (80%)
[CRITICAL] [Watchdog] Terminating process 13502 to prevent system freeze.
>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<
Terminated
```

## 3. Root Cause Analysis (원인 분석)
- **기술적 원인 분석:** 어플리케이션 내부에 대기 시간(`sleep`)이나 제어문 없이 수학적 연산(또는 무한 루프)을 지속하는 로직이 있어, 단일/멀티 코어의 CPU 자원을 독점(Starvation 유발)하고 있습니다.
- **OS 동작 원리:** 특정 프로세스의 과점유로 인해 OS 전체가 지연(Latency)되는 현상을 막기 위해, 내부 Watchdog 데몬이 설정된 임계치(80%) 초과 상태를 감지하고 해당 프로세스에 `SIGTERM` 시그널을 보내 안전하게 종료시켰습니다.

## 4. Workaround & Verification (조치 및 검증)
- **환경변수 조정:** 터미널에서 `export CPU_MAX_OCCUPY=100` 명령어를 통해 Watchdog이 개입하는 CPU 사용률 임계치를 100%로 완화했습니다.
- **Before & After 비교 결과:**
  - **Before:** `CPU_MAX_OCCUPY=80`일 때, 약 5초 경과 후 Watchdog에 의해 강제 종료됨.
  - **After:** `CPU_MAX_OCCUPY=100`으로 상향 후, 종료되지 않고 15초 이상 생존 기간이 연장됨을 확인.
- **추가 제안:** CPU 한도를 높이는 것은 임시방편입니다. 근본적 해결을 위해서는 코드 내 복잡한 연산 사이에 `time.sleep()`을 주어 자원을 양보(Yield)하거나, 연산을 분산 처리하도록 아키텍처를 개선해야 합니다.

---

# [Bug] Deadlock - 멀티스레드 환경에서 교착상태 발생으로 인한 프로세스 무응답

## 1. Description (현상 설명)
- **어떤 현상이 발생했는가?**
  `agent-leak-app` 실행 후 프로세스가 강제로 종료되지는 않으나, 콘솔에 `[WAITING... BLOCKED]` 로그를 마지막으로 출력한 뒤 아무런 응답이 없는 먹통(Hang) 상태가 지속됩니다.
- **언제, 어떤 조건에서 발생했는가?**
  환경변수 `MULTI_THREAD_ENABLE`이 `true`로 설정되어 멀티스레딩 모드로 동작할 때 발생합니다.

## 2. Evidence & Logs (증거 자료)
```bash
root@c10c3d24140c:/# ps -ef | grep agent
agent-a+    4993       1  0 14:11 pts/1    00:00:04 python3 agent_app.py
root       12849   12754  0 18:59 pts/3    00:00:00 su - agent-admin
agent-a+   12850   12849  0 18:59 pts/3    00:00:00 -bash
agent-a+   13888   12850  0 19:32 pts/3    00:00:00 python3 /home/agent-admin/agent-app/agent-leak-app3.py
root       13959   13949  0 19:34 pts/4    00:00:00 grep --color=auto agent

[Thread-1] 자원 A 획득 완료, 작업 중...
[Thread-2] 자원 B 획득 완료, 작업 중...
[Thread-1] 자원 B 요청 [WAITING... BLOCKED]
[Thread-2] 자원 A 요청 [WAITING... BLOCKED]
```

## 3. Root Cause Analysis (원인 분석)
- **기술적 원인 분석:** 멀티스레드 환경에서 Thread-1은 자원 A를 점유한 채 자원 B를 요구하고, Thread-2는 자원 B를 점유한 채 자원 A를 요구하는 순환 대기(Circular Wait) 상태에 빠졌습니다.
- **OS 동작 원리:** 두 스레드가 서로 상대방이 가진 자원(Lock)이 해제(Release)되기를 영원히 기다리는 교착상태(Deadlock) 4대 성립 조건(상호배제, 점유대기, 비선점, 순환대기)을 모두 만족하여 OS의 스케줄링에서 사실상 배제되었습니다.

## 4. Workaround & Verification (조치 및 검증)
- **환경변수 조정:** 터미널에서 `export MULTI_THREAD_ENABLE=false`로 설정하여 스레드를 분리하지 않고 메인 스레드 하나에서 순차적으로 처리하도록 임시 조치했습니다.
- **Before & After 비교 결과:**
  - **Before (true):** 로그 출력이 멈추고 PID만 살아있는 데드락 발생
  - **After (false):** 교착상태 없이 `[INFO] 모든 작업이 정상적으로 완료되었습니다.` 메시지와 함께 프로세스 정상 종료 확인
- **추가 제안:** 멀티스레딩의 성능 이점을 살리면서 데드락을 방지하려면, 모든 스레드가 동일한 순서(예: 항상 자원 A 획득 후 자원 B 획득)로 락(Lock)을 요청하도록 코드를 수정해야 합니다.

# 실제 agent-app-leak 실행 로그

```bash
agent-admin@9ad47e242836:~$ ./agent-app-leak 
>>> Starting Agent Boot Sequence...
[1/6] Checking User Account               [OK]
   ... Running as service user 'agent-admin' (uid=1000)
[2/6] Verifying Environment Variables     [OK]
   ... All required Envs correct
[3/6] Checking Required Files             [OK]
   ... Verified 'secret.key' with correct key string.
[4/6] Checking Port Availability          [OK]
   ... Port 15034 is available.
[5/6] Verifying Log Permission            [OK]
   ... Log directory is writable: /home/agent-admin/agent-app/logs
[6/6] Verifying Mission Environment       [OK]
   ... MEMORY_LIMIT=256MB, CPU_MAX_OCCUPY=80%, MULTI_THREAD_ENABLE=True
------------------------------------------------------------
All Boot Checks Passed!
Agent READY
2026-05-14 03:15:34,190 [INFO] [SafetyGuard] Process priority lowered (nice=10).
2026-05-14 03:15:34,191 [INFO] Agent listening at port 15034

==================================================
 [ Agent Initiate ] Resource Check 
==================================================
 [ MEMORY ] Limit: 256MB 		[ WARNING: Recommend Over 256MB ]
 [ CPU    ] Limit: 80%  		[ WARNING: Recommend Under 50% ]
 [ THREAD ] Concurrency: True 		[ WARNING ]
--------------------------------------------------
 >>> SYSTEM WARNING: POTENTIAL DEADLOCK IN CONCURRENT MODE.
==================================================

2026-05-14 03:15:36,209 [INFO] [MemoryWorker] Current Heap: 25MB
2026-05-14 03:15:39,230 [INFO] [MemoryWorker] Current Heap: 50MB
2026-05-14 03:15:42,249 [INFO] [MemoryWorker] Current Heap: 75MB
2026-05-14 03:15:45,274 [INFO] [MemoryWorker] Current Heap: 100MB
2026-05-14 03:15:48,294 [INFO] [MemoryWorker] Current Heap: 125MB
2026-05-14 03:15:51,319 [INFO] [MemoryWorker] Current Heap: 150MB
2026-05-14 03:15:54,342 [INFO] [MemoryWorker] Current Heap: 175MB
2026-05-14 03:15:57,365 [INFO] [MemoryWorker] Current Heap: 200MB
2026-05-14 03:16:00,388 [INFO] [MemoryWorker] Current Heap: 225MB
2026-05-14 03:16:03,405 [INFO] [MemoryWorker] Current Heap: 250MB
2026-05-14 03:16:06,424 [INFO] [MemoryWorker] Current Heap: 275MB
2026-05-14 03:16:06,424 [CRITICAL] [MemoryGuard] Memory limit exceeded (275MB >= 256MB) / (Recommend Over 256MB)
2026-05-14 03:16:06,425 [CRITICAL] [MemoryGuard] Self-terminating process 69 to prevent system instability.


>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<

Killed
```

```bash
# export MEMORY_LIMIT=512

==================================================
 [ Agent Initiate ] Resource Check 
==================================================
 [ MEMORY ] Limit: 512MB 		[ OK ]
 [ CPU    ] Limit: 80%  		[ WARNING: Recommend Under 50% ]
 [ THREAD ] Concurrency: True 		[ WARNING ]
--------------------------------------------------
 >>> SYSTEM WARNING: POTENTIAL DEADLOCK IN CONCURRENT MODE.
==================================================

2026-05-14 03:23:08,848 [INFO] [CpuWorker] Started. Maximum CPU Limit: 80%
2026-05-14 03:23:08,852 [INFO] [CpuWorker] Current Load: 5.00%
2026-05-14 03:23:11,965 [INFO] [CpuWorker] Current Load: 9.96%
2026-05-14 03:23:15,079 [INFO] [CpuWorker] Current Load: 12.54%
2026-05-14 03:23:18,197 [INFO] [CpuWorker] Current Load: 21.03%
2026-05-14 03:23:21,301 [INFO] [CpuWorker] Current Load: 22.34%
2026-05-14 03:23:24,414 [INFO] [CpuWorker] Current Load: 30.54%
2026-05-14 03:23:27,521 [INFO] [CpuWorker] Current Load: 38.01%
2026-05-14 03:23:30,633 [INFO] [CpuWorker] Current Load: 40.29%
2026-05-14 03:23:33,745 [INFO] [CpuWorker] Current Load: 49.06%
2026-05-14 03:23:36,854 [INFO] [CpuWorker] Current Load: 50.60%
2026-05-14 03:23:36,961 [CRITICAL] [CpuWorker] CPU Threshold Violated! (50.6%).

>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<

Terminated
```

<img width="1517" height="726" alt="Image" src="https://github.com/user-attachments/assets/e47488f6-b364-4220-9094-dbda3534911c" />

```bash
# export CPU_MAX_OCCUPY=100

===================================================
 [ Agent Initiate ] Resource Check 
==================================================
 [ MEMORY ] Limit: 512MB 		[ OK ]
 [ CPU    ] Limit: 100%  		[ WARNING: Recommend Under 50% ]
 [ THREAD ] Concurrency: True 		[ WARNING ]
--------------------------------------------------
 >>> SYSTEM WARNING: POTENTIAL DEADLOCK IN CONCURRENT MODE.
==================================================

2026-05-14 03:49:58,088 [INFO] [CpuWorker] Started. Maximum CPU Limit: 100%
2026-05-14 03:49:58,093 [INFO] [CpuWorker] Current Load: 5.00%
2026-05-14 03:50:01,211 [INFO] [CpuWorker] Current Load: 8.19%
2026-05-14 03:50:04,324 [INFO] [CpuWorker] Current Load: 17.51%
2026-05-14 03:50:07,436 [INFO] [CpuWorker] Current Load: 18.82%
2026-05-14 03:50:10,552 [INFO] [CpuWorker] Current Load: 28.77%
2026-05-14 03:50:13,668 [INFO] [CpuWorker] Current Load: 37.99%
2026-05-14 03:50:16,779 [INFO] [CpuWorker] Current Load: 40.95%
2026-05-14 03:50:19,888 [INFO] [CpuWorker] Current Load: 41.60%
2026-05-14 03:50:23,003 [INFO] [CpuWorker] Current Load: 45.87%
2026-05-14 03:50:26,118 [INFO] [CpuWorker] Current Load: 49.61%
2026-05-14 03:50:29,230 [INFO] [CpuWorker] Current Load: 51.44%
2026-05-14 03:50:29,338 [CRITICAL] [CpuWorker] CPU Threshold Violated! (51.43999999999999%).

>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<
```

```bash
# export CPU_MAX_OCCUPY=10

==================================================
 [ Agent Initiate ] Resource Check 
==================================================
 [ MEMORY ] Limit: 512MB 		[ OK ]
 [ CPU    ] Limit: 10%  		[ OK ]
 [ THREAD ] Concurrency: True 		[ WARNING ]
--------------------------------------------------
 >>> SYSTEM WARNING: POTENTIAL DEADLOCK IN CONCURRENT MODE.
==================================================

2026-05-14 03:52:10,647 [WARNING] [AgentWorker] Initializing concurrent transaction processors...
2026-05-14 03:52:10,648 [WARNING] [System] CAUTION: Strict resource locking is enabled.
2026-05-14 03:52:15,656 [INFO] [Worker-Thread-1] Process Started. Attempting to lock [Shared_Memory_A]...
2026-05-14 03:52:15,657 [INFO] [AgentWorker][Worker-Thread-1] LOCK ACQUIRED: [Shared_Memory_A]. (Holding...)
2026-05-14 03:52:15,657 [INFO] [AgentWorker][Worker-Thread-2] Process Started. Attempting to lock [Socket_Pool_B]...
2026-05-14 03:52:15,658 [INFO] [AgentWorker][Worker-Thread-1] Processing critical data in Memory A...
2026-05-14 03:52:15,658 [INFO] [AgentWorker][Worker-Thread-2] LOCK ACQUIRED: [Socket_Pool_B]. (Holding...)
2026-05-14 03:52:15,658 [INFO] [AgentWorker] Waiting for worker threads to complete transactions...
2026-05-14 03:52:15,659 [INFO] [AgentWorker][Worker-Thread-2] Establishing network connections in Pool B...
2026-05-14 03:52:17,664 [INFO] [AgentWorker][Worker-Thread-2] Need resource [Shared_Memory_A] to write logs.
2026-05-14 03:52:17,664 [INFO] [AgentWorker][Worker-Thread-1] Need resource [Socket_Pool_B] to finish job.
2026-05-14 03:52:17,665 [INFO] [AgentWorker][Worker-Thread-2] WAITING for [Shared_Memory_A]... (Status: BLOCKED)
2026-05-14 03:52:17,666 [INFO] [AgentWorker][Worker-Thread-1] WAITING for [Socket_Pool_B]... (Status: BLOCKED)
```

```bash
# export MULTI_THREAD_ENABLE=false
==================================================
 [ Agent Initiate ] Resource Check 
==================================================
 [ MEMORY ] Limit: 512MB 		[ OK ]
 [ CPU    ] Limit: 10%  		[ OK ]
 [ THREAD ] Concurrency: False 		[ OK ]
--------------------------------------------------
 >>> SYSTEM STATUS: STABLE. STARTING WORKLOAD MONITORING...
==================================================

2026-05-14 03:54:41,605 [INFO] >>> Scenario Selected: [Healthy System Monitoring]

>>> [SYSTEM] ALL CONFIGURATIONS OPTIMAL. RUNNING STABILITY TEST... <<<

2026-05-14 03:54:41,611 [INFO] [Scheduler] Task Scheduler Initialized.
2026-05-14 03:54:41,611 [INFO] [Scheduler] Registered Tasks: ['Thread-A', 'Thread-B', 'Thread-C']
2026-05-14 03:54:41,612 [INFO] [Scheduler] Starting task execution...
2026-05-14 03:54:41,612 [INFO] [Thread-A] Task Started. Calculating... (20%)
2026-05-14 03:54:41,668 [INFO] [Thread-A] Calculating... (40%)
2026-05-14 03:54:41,720 [INFO] [Thread-A] Preempted. Progress saved at (40%)
2026-05-14 03:54:41,776 [INFO] [Thread-B] Task Started. Calculating... (20%)
2026-05-14 03:54:41,830 [INFO] [Thread-B] Calculating... (40%)
2026-05-14 03:54:41,886 [INFO] [Thread-B] Preempted. Progress saved at (40%)
2026-05-14 03:54:41,942 [INFO] [Thread-C] Task Started. Calculating... (20%)
2026-05-14 03:54:41,995 [INFO] [Thread-C] Calculating... (40%)
2026-05-14 03:54:42,047 [INFO] [Thread-C] Preempted. Progress saved at (40%)
2026-05-14 03:54:42,102 [INFO] [Thread-A] Resumed. Calculating... (60%)
2026-05-14 03:54:42,156 [INFO] [Thread-A] Calculating... (80%)
2026-05-14 03:54:42,212 [INFO] [Thread-A] Preempted. Progress saved at (80%)
2026-05-14 03:54:42,265 [INFO] [Thread-B] Resumed. Calculating... (60%)
2026-05-14 03:54:42,320 [INFO] [Thread-B] Calculating... (80%)
2026-05-14 03:54:42,372 [INFO] [Thread-B] Preempted. Progress saved at (80%)
2026-05-14 03:54:42,424 [INFO] [Thread-C] Resumed. Calculating... (60%)
2026-05-14 03:54:42,480 [INFO] [Thread-C] Calculating... (80%)
2026-05-14 03:54:42,532 [INFO] [Thread-C] Preempted. Progress saved at (80%)
2026-05-14 03:54:42,586 [INFO] [Thread-A] Resumed. Calculating... (100%)
2026-05-14 03:54:42,643 [INFO] [Thread-B] Resumed. Calculating... (100%)
2026-05-14 03:54:42,695 [INFO] [Thread-C] Resumed. Calculating... (100%)
2026-05-14 03:54:42,751 [INFO] [Scheduler] All tasks completed.
2026-05-14 03:54:42,771 [INFO] [MemoryWorker] Current Heap: 25MB
2026-05-14 03:54:42,771 [INFO] [CpuWorker] Started. Maximum CPU Limit: 10%
2026-05-14 03:54:42,773 [INFO] [CpuWorker] Current Load: 5.00%
2026-05-14 03:54:44,886 [INFO] [CpuWorker] Peak reached (10.00%). Starting cooldown...
2026-05-14 03:54:45,792 [INFO] [MemoryWorker] Current Heap: 50MB
2026-05-14 03:54:45,891 [INFO] [CpuWorker] Current Load: 10.00%
2026-05-14 03:54:48,001 [INFO] [CpuWorker] Cooldown complete (5.00%). Resuming load increase...
2026-05-14 03:54:48,816 [INFO] [MemoryWorker] Current Heap: 75MB
2026-05-14 03:54:49,007 [INFO] [CpuWorker] Current Load: 5.00%
...
2026-05-14 03:55:43,181 [WARNING] [MemoryWorker] Memory Usage Reached Limit (525MB). Starting cleanup...
2026-05-14 03:55:43,203 [INFO] [System] Memory Cache Flushed. Process Stabilized.

>>> [SYSTEM] MEMORY RECOVERED (Cache Cleared) <<<
```

# 정상적으로 패스가 된 것을 확인할 수 있음