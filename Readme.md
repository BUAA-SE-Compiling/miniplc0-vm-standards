# miniplc0-vm-standard

这里是 miniplc0 对应的虚拟机的标准。

## 寄存器

ip: 指令寄存器，总是指向下一条要执行的指令

sp: 栈寄存器，总是指向栈内最高有效地址的下一个地址

## 指令集

指令格式

```
-----------
| 4  | 4  |
| op | x  |
-----------
```


一个指令长度为 8 bytes

|编码|指令|伪代码|全称|
|--|--|--|--|
|0|ILL|正常遇到会panic，但是在debug模式下由debugger接管。|illegal instruction|
|1|LIT x|stack\[sp\]=x;sp++;|load int
|2|LOD x|stack\[sp\]=stack\[x\];sp++;|load
|3|STO x|stack\[x\]=stack\[sp-1\];sp--;|store
|4|ADD|stack\[sp-2\]+=stack\[sp-1\];sp--;|add
|5|SUB|stack\[sp-2\]-=stack\[sp-1\];sp--;|subtract
|6|MUL|stack\[sp-2\]\*=stack\[sp-1\];sp--;|multiply
|7|DIV|stack\[sp-2\]\/=stack\[sp-1\];sp--;|divide
|8|WRT|printf\(\"\%d\\n", stack\[sp-1\]\);sp--;|write


## 存储区

### 指令区

ip 永远指向下一条要执行的指令。

```
|  instruction  | <--- ip
|               |
|               |
-----------------
```

指令长度为 8 byte，定义为

```C++
enum OP: int32_t {
    ILL = 0,
    ...
}

struct instruction {
    OP op;
    int32_t x;
};
```

### 栈

以`4 bytes`为一个寻址单位，记为`1`。寻址单位内的字节序（大端/小端）是实现决定的，因此编译器的代码生成器需要和具体的VM配套。

栈从低地址向高地址生长，以栈元素为单位，栈的最低地址为`0`，最高地址为`0x3ffff`，这说明一个栈的最大内存是`1 MB`。

栈指针 sp 总是指向栈内地址最高的元素的下一个地址。地址大于等于 sp 的内存，无论其具体存储的值为多少，都视为没有使用过的内存，对这样的地址进行访问是内存越界错误。

```
2|        |<--- sp
1|1234abcd|<--- vb
0|deadbeef|<--- va
 ----------
```
如上图描述了一个栈，其中：
- 栈地址为`0`的元素，我们将其记为变量`va`，值为`0xdeadbeef`，但是`va`的四个字节存储时实际上如何排序（大端/小端），我们并不关心
- 记`addr(v)`是变量`v`的栈地址，则`addr(vb)-addr(va) = 1`
- 栈指针`sp`指向的栈地址为`2`

请特别注意这里地址描述和x86使用的描述方式的区别：x86的内存寻址单位是`1 byte`，且栈从高地址向低地址生长，如果使用x86描述，上图中的`addr(vb)-addr(va)=-4`

## 异常定义

### ILLEGAL_INSTRUCTION

- 指令不存在

### MEMORY_ERROR

- LOD和STO访问的栈地址x越界（小于0或大于等于sp）
- 栈内元素数量小于2的时候，执行ADD/SUB/MUL/DIV
- 栈空的时候，执行WRT

### ARITHMETIC_OVERFLOW

- 四则运算的结果超过了int的值域

### DIVIDE_ZERO_ERROR

- 执行DIV指令时，次栈顶是0

### STACK_OVERFLOW

- 栈满的时候 LIT/LOD

## EPF 文件二进制格式

### 文件布局

```
| header | codes |
```

内存布局同文件布局。

### 头部

```C
struct header{
    byte[4] magic;
    int32_t version;
    int32_t intructionsCounts;
    int32_t entryPoint;
}; 
```

- magic: \{0x5A, 0x51, 0x4C, 0x53\}
- version: 目前为 1
- instructionCounts: 指令总数
- entryPoint: 入口点，不应该大于指令总数，否则是UB

### 指令区

```C
struct code{
    instruction[] codes;
}
```

- codes: instruction 数组