---
title: Memory Protection Techniques in Linux
published: true
---

## Chapter 0: Introduction - Why, in the Point of Exploitation
공룡이 뛰어다니고 레드햇과 데비안이 막 태동하던 초창기의 리눅스는 해커들의 온상이었다. Buffer Overflow라는 취약점이 발견된 이후로, 해커들에게 있어 가장 공략하기 쉬운 먹잇감은 Stack이었다. 단순히 Stack Frame의 RET 주소를 쉘코드의 주소로 덮어씌우는 것만으로도 해커들은 손쉽게 공격 대상에 대한 권한을 장악할 수 있었다.
![전형적인 Stack Overflow. 쉘코드를 적당히 넣어두고 그 주소로 뛰기만 하면 된다.](/blog/assets/2019-09-26-#1-Memory-Protection-Techniques/Stack_Overflow.png)

물론 이대로만 당하고 있을 OS 개발자들이 아니었다. 그들은 해커들이 자신들의 시스템에 함부로 접근할 수 없게 하는 보호 기법을 도입하기 시작했다.

## Chapter 1: ASLR - Do not allow Fixed Address Attack
ASLR은 Address Space Layout Randomization으로, 어떤 Section이(ASLR은 Stack 영역에만 국한된 기법이 아니다. Heap, Shared Library 영역에도 적용될 수 있다(그리고 대개는 그렇다)) 시작하는 주소를 랜덤화하는 기법이다. 전형적인 Buffer Overflow 공격은 버퍼를 부술만한 충분한 수의 A와 Shellcode로 점프하는 주소, NOP Sled와 Shellcode로 이루어진다고 봐도 과언이 아닌데, 이 ASLR이라는 기법 때문에 Shellcode로 점프하는 주소가 프로그램을 실행할 때 마다 달라지면서 기존의 방식으로 해커가 공격을 성공할 확률을 기적적으로 낮출 수 있었다.

### 1-1. Checking ASLR
Linux의 Process는 Memory에 Mapping되어있는, 일종의 파일로써 취급된다(/proc directory 하의 숫자 폴더들이 이를 증명한다.). ASLR은 ELF 표준에 의한 것이 아닌, OS에 의해서 설정되는 옵션이다. 이제 ASLR이 무엇이고, 실제로 ASLR이 Stack에 대해서 동작하는지 확인해보자.

적당한 Binary 하나를 잡아 ```/path/to/binary &``` 명령어를 이용해 백그라운드로 실행시키고, ```ps -ef | grep binary``` 명령어를 이용해 해당 binary의 pid를 알아오자.

그리고 ```cat /proc/pid/maps``` 커맨드를 실행시켜보자.
![실행 결과. Stack이 Mapping된 시작 주소는 0xfff0c000 이다.](/blog/assets/2019-09-26-#1-Memory-Protection-Techniques/ASLR#1.png)

이제 ```kill -9 pid``` 명령어로 process를 kill하고, 위 과정을 다시 반복해보자.
![실행 결과. Stack이 Mapping된 시작 주소는 0xffab1000 이다. 이전의 실행 결과와 다른 곳에 Stack이 Mapping되어있는 것을 볼 수 있다.](/blog/assets/2019-09-26-#1-Memory-Protection-Techniques/ASLR#2.png)

Stack이 Mapping된 주소가 binary를 실행할 때마다 달라지는 것을 볼 수 있다.

Trivia - 왜 매핑된 메모리에서 하위 12비트는 0으로 설정되는 것일까? 이것은 OS에서 메모리를 Page라는 단위로 관리하는데, 대개 Page의 크기는 4MB, 즉, 12비트이기 때문이다. 더 자세히 알고 싶다면 운영체제 수업을 들어보자.

### 1-2. Bypassing ASLR
그렇다면 ASLR은 어떻게 우회될 수 있을까? 더 다양한 방법이 있을 수 있겠지만 크게 세 가지 선에서 정리된다고 생각한다.

#### 1-2-1. By Brute-Forcing
해커가 exploit payload에 적어넣은 주소가 마치 태양계의 행성들이 일직선으로 나열되듯 딱 맞아떨어지는 Base Address가 나올 때까지 시도를 반복하는 것이다. 하지만 이 방식은 OS 설정에서 ASLR Address Bit의 자유도를 높이면 높일 수록, 성공할 확률이 정말로 일직선으로 나열될 확률에 가까워질 가능성이 커지므로 정말로 자유도가 높지 않다는 판단이 서지 않는다면 이 방식을 사용하지 않는 것이 여러모로 이로울 것이다.

#### 1-2-2. By Base Address Leak
프로그램이 사용하지 않는 스택의 어딘가에는 이전에 프로세스가 실행한 후 남아있는 찌꺼기 값들이 가득할 것이다. 리눅스에서는 이 값들을 명시적으로 지우지 않는 한 남겨두기 때문에, 경우에 따라서 아주 귀중한 자료로써 쓰일 수 있다. 만약 어떤 스택 주소 값이 스택에 남아있고, 그 값을 Leak해올 수 있다면, 스택 주소 값에서 Offset은 일정하므로 Stack Base Address를 구할 수 있으며, 적절한 계산을 통해 Shellcode의 Address를 하드코딩하면 된다.

#### 1-2-3. Jump to Non-Randomized Section
그렇다면 아예 Random화되지 않는 메모리 부분으로 점프하면 어떨까? 대개 코드를 저장하고 있는 .text 섹션은 랜덤화되지 않는다. 이 발상을 이용한 기법이 ROP로, 추후 충분한 고찰 이후 다루어볼 것이다.

## 2: NX bit - Do not Execute Codes in Stack
이전에 언급했던 전형적인 BOF payload에서, 문제가 되는 부분은 스택 주소를 예측하기 쉽다는 점 뿐만이 아니었다. 프로그래머가 작동하는 것을 의도한 .text 섹션의 코드 뿐만이 아닌(다른 섹션은 편의상 넘어가도록 하자), stack에 Shellcode 같은 해커가 주입한 악의적인 코드 등을 실행할 수 있다는 것이었다. 이 취약점을 막기 위해 OS 설계자들은 ELF 파일을 읽어들여, 만약 ELF 파일에 설정된 경우 stack 영역에서 eXecute 권한을 없애는는 것으로 이 문제를 해결했다(이 기법 또한 stack 영역에만 한정된 것은 아니다). 이 보호 기법을 NX bit (No eXecute) 라고 하며, DEP (Data Execution Protection), W^X (xor 연산에서 write or execute 권한만 true) 라고도 한다.

### 2.1. Checking NX bit
이번에는 ```cat /proc/pid/maps```가 아닌, 조금 우아한 방법을 사용해보자. 바로 [pwndbg](https://github.com/pwndbg/pwndbg)의 기능을 이용하는 것이다. pwndbg를 설치한 후, gdb로 binary를 부착, ```b *main``` 커맨드로 main function의 prologue에 bp를 걸고, 바이너리를 실행해보자.

그리고 ```vmmap``` 커맨드를 입력하자.
![실행 결과. Stack의 rwx 권한 중 eXecute 권한이 없는 것을 볼 수 있다. NX bit가 설정된 것이다.](/blog/assets/2019-09-26-#1-Memory-Protection-Techniques/ASLR#2.png)

stack에서 실행 권한이 없는 것 뿐만 아니라, 다른 모든 영역에서도 write 권한과 execute 권한이 동시에 존재하지 않는다는 것을 볼 수 있다.

### 2.2. Bypassing NX bit
NX bit를 우회할 수 있는 방법은 무엇일까? 바로 코드를 실행할 수 있는 섹션을 역이용하는 것이다.

#### 2.2.1. Dawn of ROP (Return Oriented Programming)
Retur Oriented Programming, 줄여서 ROP는 아주 강력한 기법으로, 실제 코드의 .text 섹션과 shared library의 .text 섹션이 실행 가능하다는 것을 역이용하는 기법이다. 이 기법은 함수가 실행될 때/끝날 때 stack을 참조하는 방식을 고려하여 일종의 chain을 만들고, 그에 맞는 code를 shared library의 leak과 코드의 .text 섹션에서 찾아 (통칭 gadget이라고 한다) 원하는 동작을 이끌어내는 방식으로 이루어진다. 더 자세한 내용은 일단은 생략하도록 한다.

## 3: RELRO - PLT/GOT issues
점점 BOF가 하기 힘든 환경이 만들어지고, Linux 환경에 대한 해커들의 이해도가 높아짐에 따라서 arbitrary memory write가 가능한 환경을 이용하는 무수한 기법들이 개발되었다. 이들 중 대표적인 것이 GOT overwrite이다. PLT/GOT에 대한 설명 또한 방대한 축에 속하므로 일단은 생략하도록 한다.




이미지 출처
[Wikipedia](https://en.wikipedia.org/wiki/Stack_buffer_overflow)

