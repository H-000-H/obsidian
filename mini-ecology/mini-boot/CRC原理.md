## 总原理
**[bit0] [bit1] [bit2] [bit3] [bit4] [bit5] [bit6] [bi7] [bi8-bit15] [bit16-bi23] [bit24-bit31]**
crc寄存器本身就有一个值假设reg
先init 寄存器的值 再判断输入反转还是正常
反转：
- 反转每一个进去的byte再放到最高位(法一)
- 反转寄存器初始值再去 反转 poly 左移变右移 运算高位变算低位(法二)
```
static uint32_t reflect(uint32_t value, uint8_t width)

{
    uint32_t result = 0;
    for (uint8_t i = 0; i < width; i++)
    {
        result = (result << 1) | (value & 1u);
        value >>= 1;
    }
    return result;
}
```
## 过程
- reg 等于传入的值(初始化)
- for byte in (总字节数)
- 依次对高8位判断(每次单拿出一个byte)看是否为1 然后为1直接异poly或再左移 不是1直接返回丢弃**若为refin== true**(反转后的poly与最低位8bit 判断异或右移)法一则对最高位反转逻辑和正常非反转一样
- reg ^= byte1 << 24;
- reg = [byte1⊕R3_old , R2_old , R1_old , R0_old]
- reg = [R2_old , R1_old , R0_old , 【新算出来8bit → R0】]
- 得到的crc新结果和最高位r2_old异或后重复上述8位过程
- **(若为查表法用2^8次方个元素的数组先和poly异或得到数组(32bit存储的是8次位移后对整体的影响)，先用高8位和新byte异或得到idx，然后直接左移8位，然后用idx去查对应的元素，然后去异或整个元素得到新元素。**
- 依次进行直到内容轮询结束

