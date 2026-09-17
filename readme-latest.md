# [Bug] OOM - 메모리 누수로 인한 MemoryGuard 보호 정책 강제 종료

## 1. Description (현상 설명)
- **어떤 현상이 발생했는가?**
  `agent-app-leak` 어플리케이션을 실행하고 일정 시간이 경과하면, 터미널에 `SELF-TERMINATED` 메시지가 출력되며 프로세스가 예고 없이 강제 종료(Killed)되는 현상이 발생했습니다.
- **언제, 어떤 조건에서 발생했는가?**
  프로세스 실행 후 지속적으로 메모리 사용량이 증가하여, 환경변수에 설정된 `MEMORY_LIMIT`(초기 256MB)을 초과하는 시점에 발생합니다.

## 2. Evidence & Logs (증거 자료)
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
...
2026-05-14 03:16:03,405 [INFO] [MemoryWorker] Current Heap: 250MB
2026-05-14 03:16:06,424 [INFO] [MemoryWorker] Current Heap: 275MB
2026-05-14 03:16:06,424 [CRITICAL] [MemoryGuard] Memory limit exceeded (275MB >= 256MB) / (Recommend Over 256MB)
2026-05-14 03:16:06,425 [CRITICAL] [MemoryGuard] Self-terminating process 69 to prevent system instability.

>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<

Killed
```

## 3. Root Cause Analysis (원인 분석)
- **기술적 원인 분석:** 어플리케이션 로직 내부에서 할당된 메모리를 해제하지 않고 지속적으로 쌓아두는 메모리 누수(Memory Leak) 결함이 존재합니다.
- **OS 동작 원리:** 물리 메모리 사용량이 `MEMORY_LIMIT`(256MB)에 도달하자, 어플리케이션 내부의 MemoryGuard 정책이 OS 전체의 불안정(System OOM)을 방지하기 위해 프로세스를 강제 종료시켰습니다.

## 4. Workaround & Verification (조치 및 검증)
- **환경변수 조정:** 터미널에서 `export MEMORY_LIMIT=512` 명령어를 통해 허용 메모리 한도를 2배로 상향 조정했습니다.
- **Before & After 검증:**
  - **Before:** 256MB 제한 시 약 32초 후 프로세스 종료
  - **After:** 512MB 상향 후 OOM으로 인한 즉각적인 강제 종료를 회피하고 다음 단계로 진행할 수 있었습니다.
- **추가 제안:** 임시 조치로 생존 시간을 늘렸으나, 근본적인 해결을 위해서는 소스 코드 내부의 불필요한 데이터를 주기적으로 삭제(Garbage Collection 유도 등)하는 리팩토링이 필수적입니다.

```bash
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

---

# [Bug] CPU - CPU 과점유에 의한 Watchdog 보호 조치 프로세스 종료

## 1. Description (현상 설명)
- **어떤 현상이 발생했는가?**
  메모리 제한을 상향한 뒤 실행 시, 시스템 CPU 사용률이 증가하면서 `[SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM)` 메시지와 함께 프로세스가 강제 종료되었습니다.
- **언제, 어떤 조건에서 발생했는가?**
  CPU 점유율이 애플리케이션의 내부 안전 권고치(50%)를 초과하여 약 50.6% 이상에 도달했을 때 발생합니다. `CPU_MAX_OCCUPY` 설정이 80% 또는 100%로 50%를 초과하게 설정되어 있으면 보호 로직이 작동하여 프로세스를 죽입니다.

## 2. Evidence & Logs (증거 자료)
```bash
# export MEMORY_LIMIT=512
...
==================================================
 [ Agent Initiate ] Resource Check 
==================================================
 [ MEMORY ] Limit: 512MB 		[ OK ]
 [ CPU    ] Limit: 80%  		[ WARNING: Recommend Under 50% ]
...
2026-05-14 03:23:08,848 [INFO] [CpuWorker] Started. Maximum CPU Limit: 80%
...
2026-05-14 03:23:36,854 [INFO] [CpuWorker] Current Load: 50.60%
2026-05-14 03:23:36,961 [CRITICAL] [CpuWorker] CPU Threshold Violated! (50.6%).

>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<

Terminated
```

<img width="1517" height="726" alt="Image" src="https://github.com/user-attachments/assets/e47488f6-b364-4220-9094-dbda3534911c" />

## 3. Root Cause Analysis (원인 분석)
- **기술적 원인 분석:** 애플리케이션 로직 내부에서 CPU 로드가 50% 이상으로 올라가는 것을 허용할 경우 시스템 프리징을 막기 위한 강제 종료 로직이 하드코딩되어 작동하고 있습니다.
- **OS 동작 원리:** 특정 프로세스의 과점유로 인해 OS 전체가 지연(Latency)되는 현상을 막기 위해, 내부 Watchdog 데몬이 권고 임계치(50%) 초과 상태를 감지하고 해당 프로세스에 `SIGTERM` 시그널을 보내 안전하게 종료시킵니다.

## 4. Workaround & Verification (조치 및 검증)
- **환경변수 조정:** 터미널에서 `export CPU_MAX_OCCUPY=10` 명령어를 통해 CPU 사용률 제한을 내부 권고 임계치(50%) 아래로 안전하게 제한했습니다.
- **Before & After 비교 결과:**
  - **Before:** `CPU_MAX_OCCUPY=80` 또는 `100`일 때, 로드가 50%를 넘으면서 약 28~31초 경과 후 Watchdog에 의해 강제 종료됨.
  - **After:** `CPU_MAX_OCCUPY=10`으로 하향 설정 후, CPU Load가 10%에 도달하면 스스로 Cooldown(휴식)을 취하며 프로세스가 종료되지 않고 유지됨을 확인했습니다.
- **추가 제안:** 연산을 분산 처리하거나 내부적으로 `time.sleep()`을 적절히 사용하여 자원을 양보(Yield)하는 아키텍처 개선이 필요합니다.

*(`export CPU_MAX_OCCUPY=100`으로 설정했을 때에는, 조금더 높은 51.4% 지점에서 강제 종료 발생 확인)*

```bash
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
---

# [Bug] Deadlock - 멀티스레드 환경에서 교착상태 발생으로 인한 프로세스 무응답

## 1. Description (현상 설명)
- **어떤 현상이 발생했는가?**
  CPU 한도까지 낮춰 프로세스 생존을 보장한 이후, 프로세스가 강제로 종료되지는 않으나 콘솔에 `WAITING... (Status: BLOCKED)` 로그를 출력한 뒤 무응답(Hang) 상태가 지속됩니다.
- **언제, 어떤 조건에서 발생했는가?**
  환경변수 `MULTI_THREAD_ENABLE`이 `true`로 설정되어 멀티스레딩 모드로 동작할 때 발생합니다.

## 2. Evidence & Logs (증거 자료)
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

2026-05-14 03:52:15,656 [INFO] [Worker-Thread-1] Process Started. Attempting to lock [Shared_Memory_A]...
2026-05-14 03:52:15,657 [INFO] [AgentWorker][Worker-Thread-1] LOCK ACQUIRED: [Shared_Memory_A]. (Holding...)
2026-05-14 03:52:15,657 [INFO] [AgentWorker][Worker-Thread-2] Process Started. Attempting to lock [Socket_Pool_B]...
2026-05-14 03:52:15,658 [INFO] [AgentWorker][Worker-Thread-2] LOCK ACQUIRED: [Socket_Pool_B]. (Holding...)
2026-05-14 03:52:17,664 [INFO] [AgentWorker][Worker-Thread-2] Need resource [Shared_Memory_A] to write logs.
2026-05-14 03:52:17,664 [INFO] [AgentWorker][Worker-Thread-1] Need resource [Socket_Pool_B] to finish job.
2026-05-14 03:52:17,665 [INFO] [AgentWorker][Worker-Thread-2] WAITING for [Shared_Memory_A]... (Status: BLOCKED)
2026-05-14 03:52:17,666 [INFO] [AgentWorker][Worker-Thread-1] WAITING for [Socket_Pool_B]... (Status: BLOCKED)
```

## 3. Root Cause Analysis (원인 분석)
- **기술적 원인 분석:** 멀티스레드 환경에서 Worker-Thread-1은 `Shared_Memory_A`를 점유한 채 `Socket_Pool_B`를 요구하고, Worker-Thread-2는 `Socket_Pool_B`를 점유한 채 `Shared_Memory_A`를 요구하는 순환 대기(Circular Wait) 상태에 빠졌습니다.
- **OS 동작 원리:** 두 스레드가 서로 상대방이 가진 자원(Lock)이 해제(Release)되기를 영원히 기다리는 교착상태(Deadlock)에 빠져 프로세스 처리가 멈췄습니다.

## 4. Workaround & Verification (조치 및 검증)
- **환경변수 조정:** 터미널에서 `export MULTI_THREAD_ENABLE=false`로 설정하여 병렬 처리를 비활성화하고, 스케줄러가 작업을 순차적으로 처리하도록 조치했습니다.
- **Before & After 비교 결과:**
  - **Before (true):** 자원 점유 상태에서 상대방 자원을 대기하며 데드락 발생
  - **After (false):** Task Scheduler에 의해 `Thread-A`, `B`, `C`가 순차적으로 실행되어 교착상태 없이 모든 작업이 완료(`All tasks completed.`)되었으며, 최종적으로 메모리 캐시가 정리되며 정상 궤도(Stabilized)에 안착했습니다.
- **추가 제안:** 멀티스레딩의 성능 이점을 살리면서 데드락을 방지하려면, 모든 스레드가 동일한 순서(예: 항상 자원 A 획득 후 자원 B 획득)로 락(Lock)을 요청하도록 코드를 수정해야 합니다.

## 5. Final Success Log (최종 성공 로그)
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

2026-05-14 03:54:41,611 [INFO] [Scheduler] Task Scheduler Initialized.
...
2026-05-14 03:54:42,586 [INFO] [Thread-A] Resumed. Calculating... (100%)
2026-05-14 03:54:42,643 [INFO] [Thread-B] Resumed. Calculating... (100%)
2026-05-14 03:54:42,695 [INFO] [Thread-C] Resumed. Calculating... (100%)
2026-05-14 03:54:42,751 [INFO] [Scheduler] All tasks completed.
2026-05-14 03:54:42,771 [INFO] [CpuWorker] Started. Maximum CPU Limit: 10%
2026-05-14 03:54:44,886 [INFO] [CpuWorker] Peak reached (10.00%). Starting cooldown...
...
2026-05-14 03:55:43,181 [WARNING] [MemoryWorker] Memory Usage Reached Limit (525MB). Starting cleanup...
2026-05-14 03:55:43,203 [INFO] [System] Memory Cache Flushed. Process Stabilized.

>>> [SYSTEM] MEMORY RECOVERED (Cache Cleared) <<<
```
