# Hydra

## 1. Fuzzing Target

![image.png](image.png)

## 2. Fuzzing IDEA

![image.png](image%201.png)

Janus → Hydra 

야누스 퍼저가 2018년도에 90 memory-safety bugs를 찾음.

하지만 야누스 퍼저는 memory-safety bugs에 만 focus했음.

Hydra는 semantic Bugs또한 찾아냄

## 3. Fuzzing Method

![image.png](image%202.png)

![image.png](image%203.png)

Semantic bug에 대한 checker 도입

![image.png](image%204.png)

Checker가 Feedback을 주어 Feedback기반 Fuzzing

![image.png](image%205.png)

전체 구조도다.

각 버그에 대한 여러 검사기를 도입했다. 

SymC3, SibylFS, KASAN, 파일 시스템 내부 검사 기능

등 여러 Checker를 결합하여 심층적인 오류 탐지가 가능하다.

![image.png](image%206.png)

**파일 시스템 이미지와 시스템 콜을 둘 다 mutate하는 Input explorer**를 새롭게 도입했다.

### Hydra Fuzzer 주요 기능

- Input Mutator ⇒ Syscall과 File System Mount
- Library OS Based Executor ⇒ LKL
- Crash Consistency Checker (SymC3)
- Other Checker Plugin
- Feedback Engine

여러 버그 Checker를 사용하는 데

메모리 버그를 위한 KASAN

Logic 버그를 위한 F2FS_CHECK_FS

Violation 버그를 위한 SibyIFS

- SibyFS
    
    파일 시스템 이미지가 명세(specification)를 위반했는지를 자동으로 검사
    
    ex) “디렉토리 엔트리는 반드시 유효한 inode를 가르켜야한다.”, “모든 파일은 루트디렉토리에서 접근 가능해야 한다.” 등
    

Crash inconsistency를 위한 B3, SymC3를 이용한다.

- B3, SymC3
    
    B3 : Crash 직전의 상태를 하나의 정답 상태로 간주, Crash 이후 복구된 파일 시스템이 이 상태와 다르면 버그, 같으면 정상
    
    SymC3 : 파일 시스템 상태를 symbolic하게 모델링하여 crash-safe 상태 집합을 계산, 복구된 파일 시스템 상태가 허용가능한 crash-safe 상태들 중 하나인지 판별
    
    c3_inode ⇒ 리눅스 inode 구조를 흉내낸 자료구조
    c3_inode로 단순 현재 상태만 저장하는 것이 아닌, 변경 이력과 fflush 여부까지 추적한다. syscall이 실행될 때마다 c3_inode를 갱신하고, syscall이 fflush한 경우, 그 시점의 스냅샷을 저장한다.
    

### 3-4. 왜 SymC3가 필요한가?

> “As a guarantee provided by file systems, any information that is flushed should be consistent even after a crash and recovery. Unfortunately, experience has revealed cases where this guarantee is violated.”
> 

![image.png](image%207.png)

B3는 crash직전 단 하나의 스냅샷만을 정답 상태고 간주한다. 하지만 실제로는 crash 타이밍, flush 여부에따라 **복구 가능한 상태가 여러 개 존재**할 수 있다. 이로 인해 **false positive가 매우 높았다.**

- [18] Bogdan Gribincea. 2009. Ext4 Data Loss. [https://bugs.launchpad.net/](https://bugs.launchpad.net/)
ubuntu/+source/linux/+bug/317781?comments=all. (2009).
- [24] Jan Kara. 2014. ext4: Forbid journal_async_commit in data=ordered
mode. [https://patchwork.ozlabs.org/patch/414750/](https://patchwork.ozlabs.org/patch/414750/). (2014).
    
    ![image.png](image%208.png)
    

> “the c3_inode also records the history of changes in the properties before the changes are committed to disk.”
> 

c3_inode라는 자료구조를 통해 `모든 변역 이력` 까지 추적한다. 

디스크의 변경되기 전의 메타데이터와 데이터 변경 내역도 함께 저장한다.

> “SymC3 is capable of generating the set of allowed post-crash states by enumerating the snapshots along with on-disk and in-memory c3_inodes.”
> 

persistent point가 발생할 때마다 snapshot을 생성하고 crash가 발생할 경우를 가정해 모든 crash-safe 상태 집합을 만든다. 복구된 이미지가 이 집합 중 어느 하나라도 포함되지 않으면, 버그로 간주한다.

### 3-5. Other Checkers

SibyIFS를 통합하여 파일 시스템의 POSIX 명세 위반 여부를 탐지한다.

- SIbyIFS는 특정 syscall 시퀀스에 대해 파일 시스템이 가질 수 있는 POSIX 허용 동작의 범위를 형식화한 도구다.
- 대부분의 로직 버그들은 silent failure를 일으킨다. 이런 버그를 찾기 위해서는 일반적으로 “파일 시스템 동작 중 도메인 특화 불변 보건(invariant)”을 검사해야한다. 
많은 파일 시스템 개발자들이 이를 인지하고 코드에 assertion 체크를 넣어주지만, 성능문제로 기본적으로 비활성화되어 있다. 하지만 Hydra는 퍼징으로 이러한 숨겨진 assertion 조건을 트리거한다. 
ex) `CONFIG_BTRFS_FS_REF_VERIFY` (Btrfs extent tree 참조 검증기)
- 메모리 오류를 감지하기 위해 KASan을 사용한다. 
BOF, UAF를 탐지한다.

### 3.6 Feedback Engine

- Branch Coverage ⇒ 전통적인 FUzzer처럼 Branch Coverage 측정
- Checker-defined signal ⇒ Hydra는 여러 Checker를 사용하는 범용 퍼징 프레임 워크다. 각 Checker가 자신만의 피드백 형식을 등록할 수 있도록 허용한다. Hydra의 모든 Checker들이 사용하는 방식처럼 피드백은 Checker에서 버그가 유발하면 boolean으로 1, 발생하지 않으면 0인 형태다.

## 4. Fuzzing Result