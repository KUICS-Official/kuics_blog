---
title: Memory Protection Techniques in Linux
published: true
---

## Chapter 0: Introduction - Why, in the Point of Exploitation
공룡이 뛰어다니고 레드햇과 데비안이 막 태동하던 초창기의 리눅스는 해커들의 온상이었다. Buffer Overflow라는 취약점이 발견된 이후로, 해커들에게 있어 가장 공략하기 쉬운 먹잇감은 Stack이었다. 단순히 Stack Frame의 RET 주소를 쉘코드의 주소로 덮어씌우는 것만으로도 해커들은 손쉽게 공격 대상에 대한 권한을 장악할 수 있었다.
![전형적인 Stack Overflow. 쉘코드를 적당히 넣어두고 그 주소로 뛰기만 하면 된다.](/blog/assets/2019-09-26-1-Memory-Protection-Techniques/Stack_Overflow.png)

물론 이대로만 당하고 있을 OS 개발자들이 아니었다. 그들은 해커들이 자신들의 시스템에 함부로 접근할 수 없게 하는 보호 기법을 도입하기 시작했다.

## Chapter 1: ASLR - Do not allow Fixed Address Attack
ASLR은 Address Space Layout Randomization으로, 어떤 Section이(ASLR은 Stack 영역에만 국한된 기법이 아니다. Heap, Shared Library 영역에도 적용될 수 있다(그리고 대개는 그렇다)) 시작하는 주소를 랜덤화하는 기법이다. 전형적인 Buffer Overflow 공격은 버퍼를 부술만한 충분한 수의 A와 Shellcode로 점프하는 주소, NOP Sled와 Shellcode로 이루어진다고 봐도 과언이 아닌데, 이 ASLR이라는 기법 때문에 Shellcode로 점프하는 주소가 프로그램을 실행할 때 마다 달라지면서 기존의 방식으로 해커가 공격을 성공할 확률을 기적적으로 낮출 수 있었다.

### 1-1. Checking ASLR
Linux의 Process는 Memory에 Mapping되어있는, 일종의 파일로써 취급된다(/proc directory 하의 숫자 폴더들이 이를 증명한다.). ASLR은 ELF 표준에 의한 것이 아닌, OS에 의해서 설정되는 옵션이다. 이제 ASLR이 무엇이고, 실제로 ASLR이 Stack에 대해서 동작하는지 확인해보자.

적당한 Binary 하나를 잡아 ```/path/to/binary &``` 명령어를 이용해 백그라운드로 실행시키고, ```ps -ef | grep binary``` 명령어를 이용해 해당 binary의 pid를 알아오자.

그리고 ```cat /proc/pid/maps``` 커맨드를 실행시켜보자.
![실행 결과. Stack이 Mapping된 시작 주소는 0xfff0c000 이다.](/blog/assets/2019-09-26-1-Memory-Protection-Techniques/ASLR-1.png)

이제 ```kill -9 pid``` 명령어로 process를 kill하고, 위 과정을 다시 반복해보자.
![실행 결과. Stack이 Mapping된 시작 주소는 0xffab1000 이다. 이전의 실행 결과와 다른 곳에 Stack이 Mapping되어있는 것을 볼 수있다.](/blog/assets/2019-09-26-1-Memory-Protection-Techniques/ASLR-2.png)

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
![실행 결과. Stack의 rwx 권한 중 eXecute 권한이 없는 것을 볼 수 있다. NX bit가 설정된 것이다.](/blog/assets/2019-09-26-1-Memory-Protection-Techniques/ASLR-2.png)

stack에서 실행 권한이 없는 것 뿐만 아니라, 다른 모든 영역에서도 write 권한과 execute 권한이 동시에 존재하지 않는다는 것을 볼 수 있다.

### 2.2. Bypassing NX bit
NX bit를 우회할 수 있는 방법은 무엇일까? 바로 코드를 실행할 수 있는 섹션을 역이용하는 것이다.

#### 2.2.1. Dawn of ROP (Return Oriented Programming)
Retur Oriented Programming, 줄여서 ROP는 아주 강력한 기법으로, 실제 코드의 .text 섹션과 shared library의 .text 섹션이 실행 가능하다는 것을 역이용하는 기법이다. 이 기법은 함수가 실행될 때/끝날 때 stack을 참조하는 방식을 고려하여 일종의 chain을 만들고, 그에 맞는 code를 shared library의 leak과 코드의 .text 섹션에서 찾아 (통칭 gadget이라고 한다) 원하는 동작을 이끌어내는 방식으로 이루어진다. 더 자세한 내용은 일단은 생략하도록 한다.

## 3: Canary - Preventing RET Overwrite
BOF의 가장 기초가 되는 부분은 무엇일까? 그렇다. RET 주소를 덮어 써서 Control Flow를 해커가 원하는 곳으로 돌리는 것이다. 그렇다면 만약, RET 주소를 덮어쓰지 못하도록 할 수 있는 방법이 있다면, BOF가 일어나는 것을 감지할 수 있다면 어떨까? 이 발상에서 Canary라는 기법이 도입되었다. 해커의 목표가 되는 주소는 ```push ebp``` 명령어에 의해 생긴 SFP(Saved Frame Pointer)를 지나, ```call func``` 명령어에 의해 저장된 RET 주소이다. Canary의 발상은 RET과 SFP 이전에 특정 값을 저장하여, 함수의 epilogue에서 이 값이 변경되었을 경우 BOF 시도로 인지하는 것이다. 이 모습이 옛날 광부가 유독 가스가 나오는 곳을 피하기 위해 캐너리아, 즉 Canary를 들고 가던 풍습과 비슷하다는 의미에서 Canary라는 이름이 붙여졌고, SSP(Stack Smashing Protection) 이라고 불리기도 한다.

### 3.1. Checking Canary
![binary decompile 결과.](/blog/assets/2019-09-26-1-Memory-Protection-Techniques/Canary-1.png)

function prologue와 epilogue를 보면 stack 지역변수 중 가장 큰 주소를 가지는 변수에 __readgsdword() 함수를 호출하고, xor 연산을 통해 그 값을 검증하는 부분이 있는 것을 볼 수 있다. 이 변수가 Canary로, 값을 바꿀 수 없는 프로세스의 특정 섹션에서 값을 읽어들여 저장/검증하는 것이다.

### 3.2. Bypassing Canary
그렇다면 Canary를 우회할 수 있는 방법은 무엇일까?

#### 3.2.1. Canary Leak
Leak은 위대하다. Canary 값을 Leak해올 수 있다면 그 값을 Canary에 해당하는 부분에 적어넣는 payload를 구성하면 된다.

#### 3.2.2. Move Focus (Rather than Stack)
Stack 영역이 아닌 다른 부분을 공략하는 것이다. Stack에서는 답이 안보이고, Heap 등을 공격할 수 있는 전형적인 패턴이 보인다면 그 쪽을 노려보도록 하자.

## 4: RELRO - PLT/GOT issues
점점 BOF가 하기 힘든 환경이 만들어지고, Linux 환경에 대한 해커들의 이해도가 높아짐에 따라서 arbitrary memory write가 가능한 환경을 이용하는 무수한 기법들이 개발되었다. 이들 중 대표적인 것이 GOT overwrite이다. PLT/GOT에 대한 설명 또한 방대한 축에 속하므로 일단은 생략하도록 한다.

### 4.1. Types of RELRO

#### 4.1.1. No RELRO
RELRO가 적용되지 않아 .dynamic 섹션의 거의 모든 부분에 값을 덮어쓸 수 있다.

#### 4.1.2 Partial RELRO
RELRO가 일부 부분에만 적용되어 .dynamic 섹션의 일부 부분에 값을 덮어쓸 수 없다. 가장 특이할 점은 프로그램의 시작과 종료 시에 Trigger 되는 init_array, fini_array에 값을 저장할 수 없게된다는 점(Case by Case일 때가 있으니 vmmap을 이용해 권한을 잘 확인해보자)이다.

#### 4.1.3. Full RELRO
RELRO가 대부분의 부분에 적용된 것으로, binary 기동 시 필요한 주소를 모두 GOT에 적어 넣고 write를 하지 못하도록 하는 것이다. 다시 말해서 GOT Overwrite가 불가능하다. 하지만 Lazy Binding이 일어나지 않는다는 특징 때문에 필연적으로 프로그램 기동 시 실행 시간면에서 손해를 본다.

### 4.2. Checking RELRO
이제 훨씬 간결한 방법을 써보자. checksec.sh를 사용하는 것이다. checksec.sh는 github에서 clone시켜 사용할 수도 있지만, pwntools를 설치하면 딸려오는 듯 하다. ```checksec binary```로 보호 기법을 확인할 수 있다.
![실행 결과. binary에 걸려있는 보안 기법들을 확인할 수 있다.](/blog/assets/2019-09-26-1-Memory-Protection-Techniques/RELRO-1.png)

checksec을 통해 binary 공략을 위한 방향성을 잡는 것이 좋고, RELRO가 적용되어있더라도 vmmap 등을 통해 해커가 사용할 수 있는 메모리 영역을 잘 관찰하는 것이 좋다.

## 5: PIE - Randomized (Almost) Everything!
일반적인 통념 하에서는 코드가 저장된 .text 섹션의 주소는 고정되어 있다. 이 발상을 통해 ROP를 위한 Gadget을 .text에서 찾는 것이 일반적이다. 하지만 PIE (Position Independent Execution)는 다르다. PIE가 걸린 바이너리는 .text 섹션마저 랜덤화하고, 각 명령어들을 offset으로 계산하여 base와의 합을 통해 메모리 주소에 배치시킨다. binary를 리버싱했을 때, .text 섹션의 주소 값이 비정상적으로 작거나 디버깅 시에 코드 주소의 값이 통상적이지 않거나 계속 변한다면 이 기법을 의심해보자.

PIE가 걸린 binary를 분석할 때는, 디버깅 시의 값을 통해 리버싱 프로그램에 base address를 설정해주는 것이 이롭다.

## 6: 글을 마치며
보안 기법이 중요한 이유는 보안 기법을 통해 바이너리를 공략할 수 있는 방법을 어느 정도 제시해주기 때문이다. 경험을 쌓아가며 이 기법이 왜 등장했으며, 어떻게 공략하면 좋을지를 잘 생각해보며 Exploit을 위한 계획을 잘 세워보자.


이미지 출처
[Wikipedia](https://en.wikipedia.org/wiki/Stack_buffer_overflow)

