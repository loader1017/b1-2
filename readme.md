
```markdown
# [Bug] OOM - 메모리 누수로 인한 MemoryGuard 보호 정책 강제 종료

## 1. Description (현상 설명)
- **어떤 현상이 발생했는가?**
  `agent-app-leak` 어플리케이션을 실행하고 일정 시간이 경과하면, 터미널에 `SELF-TERMINATED` 메시지가 출력되며 프로세스가 예고 없이 강제 종료(Killed)되는 현상이 발생했습니다.
- **언제, 어떤 조건에서 발생했는가?**
  프로세스 실행 후 지속적으로 메모리 사용량이 증가하여, 환경변수에 설정된 `MEMORY_LIMIT`(초기 256MB)을 초과하는 시점에 발생합니다.

## 2. Evidence & Logs (증거 자료)
```bash
2026-09-17 03:16:03,405 [INFO] [MemoryWorker] Current Heap: 250MB
2026-09-17 03:16:06,424 [INFO] [MemoryWorker] Current Heap: 275MB
2026-09-17 03:16:06,424 [CRITICAL] [MemoryGuard] Memory limit exceeded (275MB >= 256MB) / (Recommend Over 256MB)
2026-09-17 03:16:06,425 [CRITICAL] [MemoryGuard] Self-terminating process 69 to prevent system instability.

>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<
Killed

```

## 3. Root Cause Analysis (원인 분석)

* **기술적 원인 분석:** 어플리케이션 로직 내부에서 할당된 메모리를 해제하지 않고 지속적으로 쌓아두는 메모리 누수(Memory Leak) 결함이 존재합니다.
* **OS 동작 원리:** 물리 메모리 사용량이 `MEMORY_LIMIT`(256MB)에 도달하자, 어플리케이션 내부의 MemoryGuard 정책이 OS 전체의 불안정(System OOM)을 방지하기 위해 SIGKILL 시그널을 호출하여 프로세스를 강제 종료시켰습니다.

## 4. Workaround & Verification (조치 및 검증)

* **환경변수 조정:** 터미널에서 `export MEMORY_LIMIT=512` 명령어를 통해 허용 메모리 한도를 2배로 상향 조정했습니다.
* **Before & After 검증:**
* **Before:** 256MB 제한 시 약 32초 후 OOM 발생 및 프로세스 종료
* **After:** 512MB 상향 후 즉각적인 OOM 강제 종료를 회피하고 다음 단계를 수행할 수 있음을 확인했습니다.
* **추가 제안:** 임시 조치로 생존 시간을 늘렸으나, 근본적인 해결을 위해서는 소스 코드 내부의 불필요한 데이터를 주기적으로 삭제(Garbage Collection 유도 등)하는 리팩토링이 필수적입니다.

---

# [Bug] CPU - CPU 과점유에 의한 Watchdog 보호 조치 프로세스 종료

## 1. Description (현상 설명)

* **어떤 현상이 발생했는가?**
메모리 제한 상향 후 실행 시, 시스템 CPU 사용률이 치솟으면서 `[SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM)` 메시지와 함께 프로세스가 강제 종료(Terminated)되었습니다.
* **언제, 어떤 조건에서 발생했는가?**
프로세스의 CPU 점유율이 애플리케이션 내부 안전 권고 임계치(50%)를 초과하여 약 50.6%~51.4% 이상 도달했을 때 발생합니다.

## 2. Evidence & Logs (증거 자료)

```bash
2026-09-17 03:23:33,745 [INFO] [CpuWorker] Current Load: 49.06%
2026-09-17 03:23:36,854 [INFO] [CpuWorker] Current Load: 50.60%
2026-09-17 03:23:36,961 [CRITICAL] [CpuWorker] CPU Threshold Violated! (50.6%).

>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<
Terminated

```

## 3. Root Cause Analysis (원인 분석)

* **기술적 원인 분석:** 애플리케이션 로직 내부에서 CPU 로드가 권고 임계치(50%) 이상 상승할 경우, 시스템 프리징을 방지하기 위한 강제 종료 로직(Watchdog)이 작동하도록 설계되어 있습니다.
* **OS 동작 원리:** 특정 프로세스의 과점유로 인한 시스템 지연(Latency)을 막기 위해, 내부 Watchdog 데몬이 권고 임계치(50%) 초과 상태를 감지하고 해당 프로세스에 `SIGTERM` 시그널을 보내 안전하게 종료시켰습니다.

## 4. Workaround & Verification (조치 및 검증)

* **환경변수 조정:** `CPU_MAX_OCCUPY=100`으로 상향 설정 시에도 51.4% 지점에서 강제 종료가 발생함을 확인하여, 안전 권고 임계치(50%) 이하인 `export CPU_MAX_OCCUPY=10`으로 설정을 변경했습니다.
* **Before & After 비교 결과:**
* **Before:** `CPU_MAX_OCCUPY=80` 또는 `100` 설정 시, CPU Load가 50%를 넘으며 Watchdog에 의해 강제 종료됨.
* **After:** `CPU_MAX_OCCUPY=10` 설정 시, CPU Load가 10% 도달할 때 자발적으로 Cooldown(휴식)을 수행하며 프로세스가 종료되지 않고 안정적으로 유지됨을 확인했습니다.
* **추가 제안:** CPU 한도를 낮추는 것은 임시 조치이며, 근본적인 해결을 위해 연산 사이에 `time.sleep()`을 주어 자원을 양보(Yield)하거나 분산 처리 아키텍처를 도입해야 합니다.

---

# [Bug] Deadlock - 멀티스레드 환경에서 교착상태 발생으로 인한 프로세스 무응답

## 1. Description (현상 설명)

* **어떤 현상이 발생했는가?**
CPU 한도를 조정하여 프로세스 생존을 보장한 이후, 프로세스가 종료되지는 않으나 콘솔에 `WAITING... (Status: BLOCKED)` 로그 출력 후 먹통(Hang) 상태가 지속됩니다.
* **언제, 어떤 조건에서 발생했는가?**
환경변수 `MULTI_THREAD_ENABLE`이 `true`로 설정되어 멀티스레딩 모드로 동작할 때 발생합니다.

## 2. Evidence & Logs (증거 자료)

```bash
2026-09-17 03:52:15,657 [INFO] [AgentWorker][Worker-Thread-1] LOCK ACQUIRED: [Shared_Memory_A]. (Holding...)
2026-09-17 03:52:15,658 [INFO] [AgentWorker][Worker-Thread-2] LOCK ACQUIRED: [Socket_Pool_B]. (Holding...)
2026-09-17 03:52:17,664 [INFO] [AgentWorker][Worker-Thread-2] Need resource [Shared_Memory_A] to write logs.
2026-09-17 03:52:17,664 [INFO] [AgentWorker][Worker-Thread-1] Need resource [Socket_Pool_B] to finish job.
2026-09-17 03:52:17,665 [INFO] [AgentWorker][Worker-Thread-2] WAITING for [Shared_Memory_A]... (Status: BLOCKED)
2026-09-17 03:52:17,666 [INFO] [AgentWorker][Worker-Thread-1] WAITING for [Socket_Pool_B]... (Status: BLOCKED)

```

## 3. Root Cause Analysis (원인 분석)

* **기술적 원인 분석:** `Worker-Thread-1`은 `Shared_Memory_A`를 점유한 채 `Socket_Pool_B`를 요청하고, `Worker-Thread-2`는 `Socket_Pool_B`를 점유한 채 `Shared_Memory_A`를 요청하는 순환 대기(Circular Wait) 교착상태에 빠졌습니다.
* **OS 동작 원리:** 두 스레드가 서로 상대방의 Lock 해제를 영원히 기다리는 데드락(Deadlock) 조건이 성립하여 무한 대기 상태에 돌입했습니다.

## 4. Workaround & Verification (조치 및 검증)

* **환경변수 조정:** 터미널에서 `export MULTI_THREAD_ENABLE=false`로 설정하여 싱글 스레드 안전 모드로 전환했습니다.
* **Before & After 비교 결과:**
* **Before (true):** 동시 자원 점유 요청 과정에서 데드락 발생 및 무응답
* **After (false):** Task Scheduler에 의해 작업 스레드가 순차 실행되어 교착상태 없이 모든 작업이 정상 완료(`All tasks completed.`)됨을 확인했습니다.
* **추가 제안:** 멀티스레딩 환경을 유지하려면 모든 스레드가 동일한 순서로 Lock을 획득하도록 코드 수정이 필요합니다.

---

## 5. Final System Stabilization (최종 시스템 안정화 검증)

모든 환경변수 최적화 조치를 적용한 결과, 애플리케이션이 강제 종료나 데드락 없이 안정적으로 작동함을 최종 확인하였습니다.

* **최종 적용 환경변수:**

```bash
export MEMORY_LIMIT=512
export CPU_MAX_OCCUPY=10
export MULTI_THREAD_ENABLE=false

```

* **최종 검증 로그:**

```bash
>>> [SYSTEM] ALL CONFIGURATIONS OPTIMAL. RUNNING STABILITY TEST... <<<

2026-09-17 03:54:42,751 [INFO] [Scheduler] All tasks completed.
2026-09-17 03:54:44,886 [INFO] [CpuWorker] Peak reached (10.00%). Starting cooldown...
2026-09-17 03:55:43,181 [WARNING] [MemoryWorker] Memory Usage Reached Limit (525MB). Starting cleanup...
2026-09-17 03:55:43,203 [INFO] [System] Memory Cache Flushed. Process Stabilized.

>>> [SYSTEM] MEMORY RECOVERED (Cache Cleared) <<<

```