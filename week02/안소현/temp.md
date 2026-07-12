

## 1. 프로세스(Process)

프로세스 : 실행 중인 프로그램


프로그램은 디스크에 저장된 수동적인 파일이지만, 실행되면 메모리에 적재되고 CPU 시간, 메모리, 파일, 소켓 등의 자원을 할당받는 프로세스가 된다.

프로세스의 메모리 구조는 보통 다음과 같다.

```text
High Address

┌──────────────────┐
│      Stack       │  함수 호출, 지역 변수
│        ↓         │
│                  │
│        ↑         │
│       Heap       │  프로그램 실행 중에 동적으로 할당되는 메모리
├──────────────────┤
│       Data       │  전역 변수, static 변수
├──────────────────┤
│       Text       │  실행 코드
└──────────────────┘

Low Address
```

**프로세스마다 독립적인 가상 주소 공간을 가진다**

예를 들어 A 프로세스의 `0x1000`과 B 프로세스의 `0x1000`은 같은 주소처럼 보여도 실제 물리 메모리 위치는 다를 수 있다.

---

## 2. 프로세스 상태(Process State)

프로세스는 실행되는 동안 여러 상태를 이동한다.

```text
             scheduler dispatch
        Ready --------------------> Running
          ↑                           │
          │                           │ I/O 요청
          │                           ↓
          │                         Waiting
          │                           │
          └---------------------------┘
                 I/O 완료

Running → Terminated
```


| 상태                | 의미                |
| ----------------- | ----------------- |
| New               | 프로세스 생성 중         |
| Ready             | CPU를 기다리는 상태      |
| Running           | CPU에서 실행 중        |
| Waiting / Blocked | I/O나 이벤트를 기다리는 상태 |
| Terminated        | 실행 종료             |

- Ready > CPU만 주어지면 실행할 수 있는 상태
- Waiting > CPU를 줘도 실행할 수 없는 상태



```cpp
recv(socket, buffer, size, 0); 를 사용했을때 프로세스/스레드의 상태는 다음과 같이 변화한다. 
```

```text
Running

↓ recv() 시스템 콜

Kernel Mode

↓ 데이터 없음

Waiting

↓ NIC 패킷 도착

Interrupt

↓ Network Stack 처리

Waiting → Ready

↓ Scheduler 선택

Running
```


---

## 3. Process Control Block (PCB)

운영체제는 각 프로세스를 관리하기 위해 **PCB(Process Control Block)**라는 자료구조를 유지한다.

PCB에는 운영체제가 프로세스를 중단했다가 다시 실행하기 위해 필요한 모든 상태 정보를 저장하는데 다음과 같다. 

- Process ID (PID)
- Process State
- Program Counter
- CPU Registers
- CPU Scheduling Information
- Memory Management Information
- Accounting Information
- I/O Status Information

---

## 4. CPU Scheduler

하나의 CPU 코어가 여러 프로세스들을 빈번하게 교체하면서 실행할 수 있도록 한다. 

Ready 상태의 프로세스들은 **Ready Queue**에서 CPU를 기다린다.
- ready queue는 일반적으로 연결리스트로 저장된다. 

```text
              ┌─────────────┐
              │ Ready Queue │
              └──────┬──────┘
                     ↓
               CPU Scheduler
                     ↓
                    CPU
                     │
            I/O Request
                     ↓
                I/O Queue
                     │
                I/O Complete
                     │
                     └──→ Ready Queue
```

Scheduler는 Ready Queue에서 다음에 실행할 프로세스를 선택한다.


---


## 5. Context Switch

CPU는 하나의 코어에서 한 순간에 하나의 프로세스만 실행할 수 있다.
따라서 여러 프로세스를 실행하기 위해 CPU는 실행 대상을 변경한다.

Process A Running -> Save A CPU Context -> Load B CPU Context -> Process B Running


`Process A Running` -> `Interrupt / System Call / Scheduling` -> `Kernel Mode` -> `A의 CPU 상태 저장 (Registers, Program Counter 등)` -> `Scheduler가 B 선택` -> `B의 CPU 상태 복원` -> `User Mode` -> `Process B 실행`

Context Switch 자체는 애플리케이션의 실제 작업을 수행하지 않기 때문에 Overhead라는 것을 생각해야한다. 또한 실제 비용은 CPU Register 저장/복원보다 더 클 수 있다.

프로세스가 변경으로 인한 Overhead
- CPU Cache Miss 증가
- TLB Miss 증가
- Branch Predictor 상태 변화
- Working Set 변경

---

## 6. 프로세스 생성

실행 중인 프로세스는 다른 프로세스를 생성할 수 있다.

- Linux에서는 `fork()` 시스템 콜을 사용한다.

```cpp
pid_t pid = fork();

if (pid == 0)
{
    // Child Process
}
else
{
    // Parent Process
}
```

`fork()`를 호출하면 부모 프로세스를 복제한 자식 프로세스가 생성된다.
하지만 실제 메모리를 즉시 모두 복사하지 않는다.
대부분의 OS는 **Copy-on-Write(COW)**를 사용한다.

### fork(), exec()

`fork()` -> 현재 프로세스를 복제한다.

`exec()` -> 현재 프로세스의 메모리 공간을 새로운 프로그램으로 교체한다.

---

## 7. 프로세스 종료

프로세스는 `exit()`를 호출하여 종료할 수 있다.

```
Child : exit() -> Kernel: 종료 상태 저장 -> Parent wait() -> 종료 상태 회수
```

**Zombie Process** : Child가 종료했지만 Parent가 `wait()`를 호출하지 않으면 PCB가 일부 유지되어 좀비 프로세스가 된다. 

반대로 Parent가 먼저 종료하면 **Orphan Process**가 된다.
Linux에서는 일반적으로 `init/systemd` 등이 orphan process를 관리한다.

---

## 8. IPC (Interprocess Communication)

프로세스는 기본적으로 독립적인 주소 공간을 가진다.
따라서 프로세스 간 데이터를 교환하려면 IPC가 필요하다.

대표적인 방법은 두 가지이다.
- Shared Memory
- Message Passing

### Shared Memory

두 프로세스가 같은 메모리 영역을 공유한다.

```text
Process A
      ↓
Shared Memory
      ↑
Process B
```

- 장점
  - 빠르다
  - 데이터를 공유 메모리에 올린 이후에는 Kernel을 계속 거치지 않고 통신할 수 있다.
- 단점 
  - 동시성 문제가 발생하기 때문에 Mutex/Semaphore/Spinlock/Atomic 등의 동기화가 필요하다.


### Message Passing

Kernel을 통해 메시지를 전달한다.
- Pipe
- Message Queue
- Socket
- RPC

Shared Memory보다 일반적으로 Overhead가 크지만, 
  - 동기화가 상대적으로 단순
  - 프로세스 격리 유지
  - 분산 시스템에서도 사용 가능
하다는 장점이 있다.


|           | Shared Memory | Message Passing |
| --------- | ------------- | --------------- |
| 성능        | 빠름            | 상대적으로 느림        |
| Kernel 개입 | 초기 설정 이후 적음   | 통신마다 개입 가능      |
| 동기화       | 직접 구현 필요      | 상대적으로 단순        |
| 구현 난이도    | 높음            | 낮음              |
| 분산 시스템    | 사용 어려움        | 사용 가능           |


