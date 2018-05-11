# Java中的位运算

---

Java中的位运算主要有以下几种

| 运算描述 | 运算符 | 运算规则 | 例子 |
| :--- | :--- | :--- | :--- |
| 与 | & | 都是1，结果才为1；其他情况结果为0 | 1&1=1；1&0=0；0&1=0；0&0=0 |
| 或 | \| | 只要有1个1，结果就为1；都是0的时候，结果才为0； | 1\|1=1；1\|0=1；0\|1=1；0\|0=0 |
| 取反 | ~ | 1变0，0变1 | ~0=1;~1=0 |
| 异或 | ^ | 相同为1，不同为0 | 1^1=1；0^0=1；1^0=0；0^1=0 |
| 右移 | &gt;&gt; |  |  |
|  |  |  |  |

注：位运算时，进行运算的两个数，要转换为二进制，然后从最低位到最高位，一一对应。上面所述的运算规则，均为某位上的两个值的运算规则。例子均是某个位的数值。
