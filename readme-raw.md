```ini
# 1. root 계정에서 agent-admin 계정으로 전환
su - agent-admin

# 2. 환경변수 설정 (현재 세션 적용)
export AGENT_HOME="/home/agent-admin/agent-app"
export AGENT_PORT="15034"
export AGENT_UPLOAD_DIR="$AGENT_HOME/upload_files"
export AGENT_KEY_PATH="$AGENT_HOME/api_keys"
export AGENT_LOG_DIR="$AGENT_HOME/logs"

# 미션을 위한 초기 제어 변수 
export MEMORY_LIMIT="256"
export CPU_MAX_OCCUPY="80"
export MULTI_THREAD_ENABLE="true"

# 3. 필수 디렉토리 생성 (-p 옵션으로 없으면 생성)
mkdir -p $AGENT_UPLOAD_DIR
mkdir -p $AGENT_KEY_PATH
mkdir -p $AGENT_LOG_DIR

# 4. secret.key 파일 생성 및 내용(agent_api_key_test) 입력
echo "agent_api_key_test" > $AGENT_KEY_PATH/secret.key #지정한 파일에 덮어쓰기

# 5. 정상적으로 적용되었는지 환경변수 확인
env | grep AGENT

AGENT_UPLOAD_DIR=/home/agent-admin/agent-app/upload_files 
AGENT_PORT=15034 AGENT_KEY_PATH=/home/agent-admin/agent-app/api_keys 
AGENT_HOME=/home/agent-admin/agent-app 
AGENT_LOG_DIR=/home/agent-admin/agent-app/logs

# agent-leak-app.py(메모리 누수 임의생성)

#OOM 분석
$AGENT_HOME/bin/monitor.sh 
python3 /home/agent-admin/agent-app/agent-leak-app.py

====== SYSTEM MONITOR RESULT ====== 
[HEALTH CHECK] Checking process 'agent_app.py'... [OK] (PID: 4993) 
Checking port 15034... [OK] 
[FIREWALL CHECK] 
[WARNING] UFW firewall inactive 
[RESOURCE MONITORING] 
CPU Usage : 0.0% 
MEM Usage : 3.5% 
DISK Used : 3% [INFO] 
Log appended: /var/log/agent-app/monitor.log 
agent-admin@c10c3d24140c:~$ python3 /home/agent-admin/agent-app/agent-leak-app.py 
[INFO] agent-leak-app 시작 (PID: 13293) 
[INFO] 설정된 MEMORY_LIMIT: 256MB 
[INFO] 서비스 실행 중... (데이터 적재 시작)  
[CRITICAL] [MemoryGuard] Memory limit exceeded (258MB >= 256MB) / (Recommend Over 256MB) 
[CRITICAL] [MemoryGuard] Self-terminating process 13293 to prevent system instability. 
>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<< 
Killed

#검증 테스트
export MEMORY_LIMIT=512
python3 /home/agent-admin/agent-app/agent-leak-app.py

# agent-leak-app2.py(CPU 과점유 임의생성)

python3 /home/agent-admin/agent-app/agent-leak-app2.py
[INFO] agent-leak-app 시작 (PID: 13502)
[INFO] 설정된 CPU_MAX_OCCUPY: 80%
[INFO] 서비스 실행 중... (복잡한 연산 처리 시작)
[CRITICAL] [Watchdog] CPU usage exceeded threshold (80%)
[CRITICAL] [Watchdog] Terminating process 13502 to prevent system freeze.
>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<
Terminated

#검증 테스트
export CPU_MAX_OCCUPY=100
python3 /home/agent-admin/agent-app/agent-leak-app2.py

# agent-leak-app3.py(교착상태 임의생성)
export MULTI_THREAD_ENABLE=true
python3 /home/agent-admin/agent-app/agent-leak-app3.py

[Thread-1] 자원 A 획득 완료, 작업 중...
[Thread-2] 자원 B 획득 완료, 작업 중...
[Thread-1] 자원 B 요청 [WAITING... BLOCKED]
[Thread-2] 자원 A 요청 [WAITING... BLOCKED]

root@c10c3d24140c:/# ps -ef | grep agent
agent-a+    4993       1  0 14:11 pts/1    00:00:04 python3 agent_app.py
root       12849   12754  0 18:59 pts/3    00:00:00 su - agent-admin
agent-a+   12850   12849  0 18:59 pts/3    00:00:00 -bash
agent-a+   13888   12850  0 19:32 pts/3    00:00:00 python3 /home/agent-admin/agent-app/agent-leak-app3.py
root       13959   13949  0 19:34 pts/4    00:00:00 grep --color=auto agent

# 검증 테스트

export MULTI_THREAD_ENABLE=false
python3 /home/agent-admin/agent-app/agent-leak-app3.py

[INFO] agent-leak-app 시작 (PID: 14053)
[INFO] 설정된 MULTI_THREAD_ENABLE: False
[INFO] 싱글스레드(안전) 모드 실행 중... (순차 처리 시작)

[Main] 자원 A 획득 및 작업 완료
[Main] 자원 B 획득 및 작업 완료
[INFO] 모든 작업이 정상적으로 완료되었습니다.


# agent-leak-app 코드
cat << 'EOF' > /home/agent-admin/agent-app/agent-leak-app.py
#!/usr/bin/env python3
import os
import sys
import time
import signal

# 1. 사전 준비 사항(조건) 체크
if os.geteuid() == 0:
    print("[ERROR] root 계정으로 실행할 수 없습니다.")
    sys.exit(1)

agent_home = os.environ.get("AGENT_HOME")
if not agent_home:
    print("[ERROR] AGENT_HOME 환경변수가 설정되지 않았습니다.")
    sys.exit(1)

# 환경변수 로드
memory_limit = int(os.environ.get("MEMORY_LIMIT", 256))

# 리눅스 프로세스 메모리(VmRSS) 측정 함수
def get_memory_usage_mb():
    with open('/proc/self/status') as f:
        for line in f:
            if line.startswith('VmRSS:'):
                return int(line.split()[1]) / 1024
    return 0

def start_memory_leak():
    print(f"[INFO] agent-leak-app 시작 (PID: {os.getpid()})")
    print(f"[INFO] 설정된 MEMORY_LIMIT: {memory_limit}MB")
    print("[INFO] 서비스 실행 중... (데이터 적재 시작)")
    
    leaked_data = []
    
    try:
        while True:
            # 의도적인 메모리 누수 발생 (한 번에 약 10MB씩 가짜 데이터 할당)
            leaked_data.append(' ' * 1024 * 1024 * 10)
            time.sleep(1) # 모니터링을 위해 1초 대기
            
            current_mem = get_memory_usage_mb()
            
            # MemoryGuard 정책: 메모리 임계치 초과 확인
            if current_mem >= memory_limit:
                print(f"\n[CRITICAL] [MemoryGuard] Memory limit exceeded ({int(current_mem)}MB >= {memory_limit}MB) / (Recommend Over {memory_limit}MB)")
                print(f"[CRITICAL] [MemoryGuard] Self-terminating process {os.getpid()} to prevent system instability.")
                print(">>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<")
                
                # SIGKILL로 강제 종료
                os.kill(os.getpid(), signal.SIGKILL)
                
    except KeyboardInterrupt:
        print("\n[INFO] 사용자에 의해 종료되었습니다.")

if __name__ == "__main__":
    start_memory_leak()
EOF

# 실행 권한 부여
chmod +x /home/agent-admin/agent-app/agent-leak-app.py


# agent-leak-app2 코드
cat << 'EOF' > /home/agent-admin/agent-app/agent-leak-app.py
#!/usr/bin/env python3
import os
import time
import threading

# 환경변수 로드
multi_thread = os.environ.get("MULTI_THREAD_ENABLE", "true").lower() in ["true", "1", "yes"]

# 자원(Lock) 생성
resource_a = threading.Lock()
resource_b = threading.Lock()

def thread_1_task():
    print("[Thread-1] 자원 A 획득 완료, 작업 중...")
    resource_a.acquire()
    time.sleep(1) # 스레드 교차를 위한 대기
    
    print("[Thread-1] 자원 B 요청 [WAITING... BLOCKED]")
    resource_b.acquire()
    print("[Thread-1] 자원 B 획득 완료!") # 데드락 발생 시 이 줄은 영원히 출력되지 않음
    
    resource_b.release()
    resource_a.release()

def thread_2_task():
    print("[Thread-2] 자원 B 획득 완료, 작업 중...")
    resource_b.acquire()
    time.sleep(1) # 스레드 교차를 위한 대기
    
    print("[Thread-2] 자원 A 요청 [WAITING... BLOCKED]")
    resource_a.acquire()
    print("[Thread-2] 자원 A 획득 완료!") # 데드락 발생 시 이 줄은 영원히 출력되지 않음
    
    resource_a.release()
    resource_b.release()

def start_deadlock_test():
    print(f"[INFO] agent-leak-app 시작 (PID: {os.getpid()})")
    print(f"[INFO] 설정된 MULTI_THREAD_ENABLE: {multi_thread}")
    
    if multi_thread:
        print("[INFO] 멀티스레드 모드 실행 중... (병렬 처리 시작)\n")
        t1 = threading.Thread(target=thread_1_task)
        t2 = threading.Thread(target=thread_2_task)
        
        t1.start()
        t2.start()
        
        t1.join()
        t2.join()
    else:
        print("[INFO] 싱글스레드(안전) 모드 실행 중... (순차 처리 시작)\n")
        print("[Main] 자원 A 획득 및 작업 완료")
        print("[Main] 자원 B 획득 및 작업 완료")
        print("[INFO] 모든 작업이 정상적으로 완료되었습니다.")

if __name__ == "__main__":
    start_deadlock_test()
EOF


# agent-leak-app3 코드
cat << 'EOF' > /home/agent-admin/agent-app/agent-leak-app.py
#!/usr/bin/env python3
import os
import time
import threading

# 환경변수 로드
multi_thread = os.environ.get("MULTI_THREAD_ENABLE", "true").lower() in ["true", "1", "yes"]

# 자원(Lock) 생성
resource_a = threading.Lock()
resource_b = threading.Lock()

def thread_1_task():
    print("[Thread-1] 자원 A 획득 완료, 작업 중...")
    resource_a.acquire()
    time.sleep(1) # 스레드 교차를 위한 대기
    
    print("[Thread-1] 자원 B 요청 [WAITING... BLOCKED]")
    resource_b.acquire()
    print("[Thread-1] 자원 B 획득 완료!") # 데드락 발생 시 이 줄은 영원히 출력되지 않음
    
    resource_b.release()
    resource_a.release()

def thread_2_task():
    print("[Thread-2] 자원 B 획득 완료, 작업 중...")
    resource_b.acquire()
    time.sleep(1) # 스레드 교차를 위한 대기
    
    print("[Thread-2] 자원 A 요청 [WAITING... BLOCKED]")
    resource_a.acquire()
    print("[Thread-2] 자원 A 획득 완료!") # 데드락 발생 시 이 줄은 영원히 출력되지 않음
    
    resource_a.release()
    resource_b.release()

def start_deadlock_test():
    print(f"[INFO] agent-leak-app 시작 (PID: {os.getpid()})")
    print(f"[INFO] 설정된 MULTI_THREAD_ENABLE: {multi_thread}")
    
    if multi_thread:
        print("[INFO] 멀티스레드 모드 실행 중... (병렬 처리 시작)\n")
        t1 = threading.Thread(target=thread_1_task)
        t2 = threading.Thread(target=thread_2_task)
        
        t1.start()
        t2.start()
        
        t1.join()
        t2.join()
    else:
        print("[INFO] 싱글스레드(안전) 모드 실행 중... (순차 처리 시작)\n")
        print("[Main] 자원 A 획득 및 작업 완료")
        print("[Main] 자원 B 획득 및 작업 완료")
        print("[INFO] 모든 작업이 정상적으로 완료되었습니다.")

if __name__ == "__main__":
    start_deadlock_test()
EOF
```
