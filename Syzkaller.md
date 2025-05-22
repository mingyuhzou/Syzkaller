# Syzkaller







# Syzlang语法

Syzlang是Syzkaller使用的**接口定义语言**，用于描述**Linux系统调用的参数规格**，Syzkaller根据这些定义自动生成fuzz输入。



## 参数类型

| 类型     | 示例                           | 含义                                                         |
| -------- | ------------------------------ | ------------------------------------------------------------ |
| intN     | int32[0:1000，4]               | N位整数，指定了0到100(大括号是**闭区间**)且必须是4的倍数     |
| ptr      | ptr[in,XXX]                    | 指针(**in只读/out只写/inout读写**)，例子表示的是一个指向XXX的只读指针 |
| array    | array[int32]/ array[int32,5]   | 可变或固定长度**数组**                                       |
| flags    | flags[flagname,int32]          | 标志集合                                                     |
| string   | string['foo']/string[filename] | 字符串或文件名，结合ptr[in, string["/dev/null"]]传入指向字符串的输入指针 |
| const    | const[值 , 类型]               | **常量值**                                                   |
| len      | len[buf]                       | 某个字段的长度，**动态关联**                                 |
| proc     | proc[20000,4,int16be]          | 进程特有的值                                                 |
| resource | resource fd[int32]             | **定义**一种资源类型，名称是fd，底层是int32                  |
| struct   | ...                            | 结构体，包含多个字段                                         |



结构体

```python
my_struct {
    field0 const[1, int32] (in)
    field1 int32 (inout)
    field2 fd (out)
}
```

+ field0一个用于写入的常量
+ field1 可被修改的整数值
+ field2 输出字段

结构体作为参数使用**指针**传递

```python
some_syscall(arg ptr[inout, my_struct])
```



union表示**多选一**的字段结构，如果没有条件判断，**默认**的就是**随机选择**，字段之间不需要加**逗号**。

```python
command {
  cmd_id int32
  payload [
    foo int32 (if[value[cmd_id] == 1])
    bar array[int8, 4] (if[value[cmd_id] == 2])
    baz int64
  ] [varlen]
} [packed]
```





## 类型别名

```python
type signalno int32[0:65]
type buffer[DIR] ptr[DIR, array[int8]]
```

和C一样





## 基本结构

系统调用的基本结构如下

```
syscallname(arg1 type1,arg2 type2) return_type (attr1,attr2)
```

+ **syscallname：系统调用名**
+ **arg，type：参数名称与类型**
+ **return_type：返回值类型**
+ **attri：控制调用行为的属性**



```python
write(fd fd, buf ptr[in, array[int8]], count len[buf])
```

+ fd是一个资源，其类型fd是预定义的resource fd[int32]，用于表示文件
+ buf是一个执行int8类型的数组的指针
+ count是表示buf长度的变量，会自动同步

相当于

```python
ssize_t write(int fd, const void *buf, size_t count);
```



## 调用属性

| 属性名            | 含义                                                         |
| ----------------- | ------------------------------------------------------------ |
| `disabled`        | **禁用**该 syscall，不生成测试用例（但保留定义）             |
| `timeout[N]`      | **增加**该 syscall 的**单独执行超时时间**（毫秒）            |
| `prog_timeout[N]` | 如果程序包含此 syscall，整个程序的执行时间将增加 N 毫秒      |
| `ignore_return`   | 忽略该 syscall 的返回值，不用它参与反馈机制（常用于时间、随机数等） |
| `breaks_returns`  | 执行此调用后，**忽略所有后续调用的返回值**（返回值不可被信任） |
| `no_generate`     | 不自动生成该 syscall，只使用种子程序中的它（适用于只读分析） |
| `no_minimize`     | 不对该 syscall 进行最小化（保持其原始调用参数）              |
| `fsck`            | 针对压缩文件系统的检查调用，结合 `compressed_image` 使用     |
| `remote_cover`    | 调用执行完后等得更久一点，用于收集远程 coverage（适用于嵌套 RPC 等） |



```python
reboot(magic const[0xfee1dead, int32]) int32 (disabled)
```

禁用reboot



```python
mount(src ptr[in, string], ...) int32 (timeout[5000])
```

增加时限防止误杀



## 条件字段

```python
field type (if[value[路径] == 常量])
```

其中**value[X]**表示取字段的值，判断条件用**()**包裹，if的条件用**[]**包裹



条件字段还包括**嵌套引用**，即

```python
packet {
  header header_struct
  data   array[int8] (if[value[header:has_data] == 1])
}

header_struct {
  magic     const[0x1234, int16]
  has_data  int8
} [packed]
```

表示访问header字段下的has_data字段

在结构体末尾添加**[packed]**，表示结构体是可变的



