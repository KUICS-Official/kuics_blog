---
title: Codegate2020 SimpleMachine writeup
published: true
---

간단한 CPU를 구현한 리버싱 문제이다.

파일을 받으면 명령어가 들어있는 바이너리 파일과 CPU를 구현하고 있는 ELF 파일이 있다.

```
00000000: 0624 0000 0040 2400 0129 0040 0040 bdb0  .$...@$..).@.@..
00000010: 0444 0100 0000 0040 0520 0000 0100 a001  .D.....@. ......
00000020: 0129 0240 0240 bcba 0444 0100 0000 0240  .).@.@...D.....@
00000030: 0520 0000 0100 a001 0129 0440 0440 b9be  . .......).@.@..
00000040: 0444 0100 0000 0440 0520 0000 0100 a001  .D.....@. ......
00000050: 0129 0640 0640 acba 0444 0100 0000 0640  .).@.@...D.....@
00000060: 0520 0000 0100 a001 0129 0840 0840 cecf  . .......).@.@..
...
```

바이너리를 IDA로 열어보면 메인 함수는 아래와 같다.

메인 함수는 가상 머신의 여러 상태 정보를 담고 있는 Context 구조체를 초기화한다.
```
int main(int argc, char **argv)
{
  // ...
  context = (Context *)operator new(0x48uLL);
  initialize_context(context, (const void **)&file_addr_ptr);

  while ( keep_running(context) )
    evaluate(context);

  // ...
}
```

아래는 Context 구조체 정의이다.
```
struct Context {
    buffer: u64,
    buffer_end: u64,
    buffer_end_: u64,
    keep_running: u32,
    rax_cache: u16[3],
    pc: u16,
    inst_low0: u8,
    inst_low7: u8,
    inst_low10: u8,
    inst_low13: u8,
    operand_1: u32,
    operand_2: u32,
    operand_3: u32,
    mem_fetch: u8,
    dummy: u8,
    opcode: u8,
    inst_low7_: u8,
    operand_1_: u32,
    reg_1: u32,
    reg_2: u32,
    is_executable: u8,
    dummy: u8,
    inst_low7__: u8,
    dummy: u8,
    operand_1__: u32,
    rax_: u32,
    full_pipeline: u8,
    dummy: u8,
    dummy: u8,
    dummy: u8,
    dummy: u8,
    dummy: u8,
    dummy: u8,
    dummy: u8,
}
```

`execute()` 함수는 아래와 같다.

전체적인 코드 구조를 살펴보면 파이프라인 구조에 기반해서 CPU를 구현하고 있다.

```
void evaluate(Context *context)
{
  mem_write_back(context);
  if ( context->is_executable )
    execute(context);
  if ( context->mem_fetch )
    mem_fetch(context);
  inst_fetch(context);
}
```

`inst_fetch()`를 살펴보면 아래와 같다.

```
void inst_fetch(Context *context)
{
  // variable declarations

  instruction = get_value(context, context->pc);
  context->inst_low0 = instruction & 0x7F;
  context->inst_low7 = (instruction >> 7) & 7;
  context->inst_low10 = (instruction >> 10) & 7;
  context->inst_low13 = (instruction >> 13) & 7;
  
  pc = context->pc;

  context->operand_1 = get_value(context, pc_ + 2);
  context->operand_2 = get_value(context, context->pc + 4);
  context->operand_3 = get_value(context, context->pc + 6);

  context->mem_fetch = 1;
  context->pc += 8;
}
```

가상 머신은 8 바이트 명령어를 사용하고 프로그램 카운터와 세 개의 레지스터를 갖고 있으며 이 중에서

두 개는 연산에 쓰이고 나머지 하나는 연산의 결과를 저장하는 데 쓰인다.


`inst_fetch()`는 명령어 데이터에서 2 바이트를 읽어 Little endian으로 저장한다.

명령어는 네 부분으로 이루어져 있다.
- instruction parts(2 bytes)
- operand 1(2 bytes)
- operand 2(2 bytes)
- operand 3(2 bytes)

그리고 instruction parts는 다시 네 부분으로 이루어져 있다.
- reg_2_type: Immediate 값을 사용할 것인지 메모리에 있는 값을 가져올 것인지 결정(두 번째 레지스터)
- reg_1_type: Immediate 값을 사용할 것인지 메모리에 있는 값을 가져올 것인지 결정(첫 번째 레지스터)
- mem_write: 이전 연산의 결과를 메모리에 저장할 것인지 결정
- opcode: CPU가 어떤 연산을 수행할 지를 결정
  - 0: mov
  - 1: add
  - 2: mul
  - 3: xor
  - 4: cmp
  - 5: branch
  - 6: 표준 입력에서 데이터를 가져와서 버퍼에 저장.
  - 7: 표준 출력에 버퍼의 내용을 쓴다.
  - 8: 가상 머신 종료
```
+----------------------------------------------------------------------------------+
|  reg_2_type(3-bit)  |  reg_1_type(3-bit)  |  mem_write(3-bit)  |  opcode(7-bit)  |
+----------------------------------------------------------------------------------+
```

`mem_fetch()`는 파이프라인에서 필요한 여러 포워딩 로직을 구현하고 있다.

구현이 매우 복잡한데, 사실 구체적으로 분석할 필요는 없었다.

중요한 건 각 레지스터에 어떤 값이 저장되는 지가 중요하기 떄문이다.

그래서 `execute()`에 브레이크 포인트를 설정하고 매번 명령어가 실행할 때 어떤 연산이 어떤 값에 대해 실행되는 지 확인하면 된다.

```
void execute(Context *context)
{
  // variable declarations

  switch ( (unsigned __int64)context->opcode )
  {
    case 0uLL:                                  // mov
      context->rax_ = context->reg_1;
      goto LABEL_3;
    case 1uLL:                                  // add
      context->rax_ = context->reg_1 + context->reg_2;
      goto LABEL_3;
    case 2uLL:                                  // mul
      context->rax_ = context->reg_2 * context->reg_1;
      goto LABEL_3;
    case 3uLL:                                  // xor
      context->rax_ = context->reg_2 ^ context->reg_1;
      goto LABEL_3;
    case 4uLL:                                  // cmp
      context->rax_ = context->reg_1 < context->reg_2;
      goto LABEL_3;
    case 5uLL:                                  // branch
      if ( !context->reg_1 )
        goto LABEL_3;
      v2 = context->reg_2;
      context->mem_fetch = 0;
      context->is_executable = 0;
      context->full_pipeline = 0;
      context->pc = v2;
      break;
    case 6uLL:                                  // read
      context->rax_ = read_(context, context->reg_1, context->reg_2);
      goto LABEL_3;
    case 7uLL:                                  // write
      context->rax_ = write(1, (const void *)(context->buffer + context->reg_1), context->reg_2);
      goto LABEL_3;
    case 8uLL:                                  // terminate
      context->mem_fetch = 0;
      context->is_executable = 0;
      context->full_pipeline = 0;
      context->keep_running = 0;
      return;
    default:
LABEL_3:
      inst_low7_ = context->inst_low7_;
      context->full_pipeline = 1;
      context->inst_low7__ = inst_low7_;
      context->operand_1__ = context->operand_1_;
      break;
  }
}
```

정리하면 플래그 인증 과정은 아래와 같다.
1. 명령어 초반부
   * 입력에서 2 바이트를 가져온다.
   * 어떤 2 바이트 Immediate 값을 위에서 읽은 값과 더한다.
   * 더한 결과가 0이 돼야 한다. (0이 아니라면 바로 종료)
   * 다르게 표현하면 입력값이 2 바이트 Immediate 값에 대해 2의 보수가 돼야 한다.
   * 이 부분을 분석하고 나면 플래그가 "CODEGATE2020"로 시작한다는 것을 알 수 있다.
2. 플래그 내용 부분
   * `0xdead`가 초기 key로 쓰인다.
   * `key = (key * 2) ^ key;`를 계산
   * key와 입력값 2바이트를 xor한다.
   * 위 결과와 어떤 2 바이트 Immediate 값을 더한다.
   * 더한 결과가 0이 아니라면 종료.
   * 두 번째 단계로 돌아가서 반복

직접 디버깅을 하든지 스크립트를 작성하면 플래그를 얻을 수 있다.
```
CODEGATE2020{ezpz_but_1t_1s_pr3t3xt}
```
