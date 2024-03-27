# 1. Introduction
Summit Exerciser 脚本由以下格式语句组成：
```shell
COMMAND = MODIFIER{
	PARAM1 = VALUE1
	...
	PARAM2 = VALUE2
}
```

语法说明：
- 不区分大小写
- 除非另有说明，所有默认值为 0
- 支持 16 进制、10 进制、2 进制
	- 16 进制：0x2A, 0x54 ...
	- 10 进制：24， 1256
	- 2 进制：0b01101100, 0b01
- 可以使用表达式
	- (i - 239)
	- (BASE_ADDRESS + 0x1000)，Base_ADDRESS 使用 config=defition 定义
	- Note：仅当使用算术运算时才出现圆括号，如果将一个值放在圆括号中，而不进行任何算术运算，该值将重置为 0
- 字符串使用 “ ”
- 数组数据类型由 () 包围的整数或字符串，如 (2,23,4)
- 单行注释 ;
- 多行注释 /* */