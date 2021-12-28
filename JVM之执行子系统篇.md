# 六、类文件结构 (通读了解即可)

## 6.1 Class类文件的结构  

Class文件是一组以**8个字节为基础单位**的二进制流， 各个数据项目严格按照顺序紧凑地排列在文件之中， 中间没有添加任何分隔符， 这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据， 没有空隙存在。 当遇到需要占用8个字节以上空间的数据项时， 则会按照**高位在前**的方式分割成若干个8个字节进行存储。

《Java虚拟机规范》规定：Class文件格式采用一种类似于C语言结构体的伪结构来存储数据， 这种伪结构中只有两种数据类型： “**无符号数**”和“**表**”。  

**无符号数**：属于基本的数据类型， 以u1、 u2、 u4、 u8来分别代表1个字节、 2个字节、 4个字节和8个字节的无符号数， 无符号数可以用来描述数字、 索引引用、 数量值或者按照UTF-8编码构成字符串值。  

**表**：由多个无符号数或者其他表作为数据项构成的复合数据类型，表用于描述有层次关系的复合结构的数据， 整个Class文件本质上也可以视作是一张表， 这张表由下表所示的数据项按严格顺序排列构成。  

| 类型           | 名称                | 数量                  |
| -------------- | ------------------- | --------------------- |
| u4             | magic               | 1                     |
| u2             | minor_version       | 1                     |
| u2             | major_version       | 1                     |
| u2             | constant_pool_count | 1                     |
| cp_info        | constant_pool       | constant_pool_count-1 |
| u2             | access_flags        | 1                     |
| u2             | this_class          | 1                     |
| u2             | super_class         | 1                     |
| u2             | interfaces_count    | 1                     |
| u2             | interfaces          | interfaces_count      |
| u2             | fields_count        | 1                     |
| field_info     | fields              | fields_count          |
| u2             | methods_count       | 1                     |
| method_info    | methods             | methods_count         |
| u2             | attributes_count    | 1                     |
| attribute_info | attributes          | attributes_count      |

### 6.1.1 魔数与Class文件的版本  

每个Class文件的头4个字节被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。 紧接着魔数的4个字节存储的是Class文件的版本号：**第5和第6个字节是次版本号（Minor Version），第7和第8个字节是主版本号（Major Version）** 。   

### 6.1.2 常量池 

**紧接着主、次版本号之后的是常量池入口**，常量池可以比喻为Class文件里的资源仓库，它是Class文件结构中与其他项目关联最多的数据，通常也是占用Class文件空间最大的数据项目之一。  

由于常量池中常量的数量是不固定的，所以在**常量池的入口需要放置一项u2类型的数据**，代表常量池容量计数值（**constant_pool_count**）。这个容量计数是从1而不是0开始的，设计者将第0项常量空出来是有特殊考虑的， 这样做的目的在于， 如果后面某些指向常量池的索引值的数据，在特定情况下需要表达“不引用任何一个常量池项目”的含义， 可以把索引值设置为0来表示。  

常量池中主要存放两大类常量：**字面量（Literal）**和**符号引用（Symbolic References）**。字面量比较接近于Java语言层面的常量概念，如文本字符串、被声明为final的常量值等。而符号引用则属于编译原理方面的概念，主要包括下面几类常量：

1. 被模块导出或者开放的包（Package）
2. 类和接口的全限定名（Fully Qualified Name）
3. 字段的名称和描述符（Descriptor）
4. 方法的名称和描述符
5. 方法句柄和方法类型（Method Handle、 Method Type、 Invoke Dynamic）
6. 动态调用点和动态常量（Dynamically-Computed Call Site、 Dynamically-Computed Constant）  

Class文件中不会保存各个方法、字段最终在内存中的布局信息，这些字段、方法的符号引用不经过虚拟机在运行期转换的话是无法得到真正的内存入口地址，也就无法直接被虚拟机使用的。当虚拟机做类加载时，将会从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。

常量池中每一项常量都是一个表，常量表中分别有17种不同类型的常量。 表结构起始的第一位是个u1类型的标志位（tag标志列） ， 代表着当前常量属于哪种常量类型。 17种常量类型所代表的具体含义如下表所示。 

| 类型                             | 标志 | 描述                         |
| -------------------------------- | ---- | ---------------------------- |
| CONSTANT_Utf8_info               | 1    | UTF-8编码的字符串            |
| CONSTANT_Integer_info            | 3    | 整型字面量                   |
| CONSTANT_Float_info              | 4    | 浮点型字面量                 |
| CONSTANT_Long_info               | 5    | 长整型字面量                 |
| CONSTANT_Double_info             | 6    | 双精度浮点型字面量           |
| CONSTANT_Class_info              | 7    | 类或接口的符号引用           |
| CONSTANT_String_info             | 8    | 字符串类型字面量             |
| CONSTANT_Fieldref_info           | 9    | 字段的符号引用               |
| CONSTANT_Methodref_info          | 10   | 类中方法的符号引用           |
| CONSTANT_InterfaceMethodref_info | 11   | 接口中方法的符号引用         |
| CONSTANT_NameAndType_info        | 12   | 字段或方法的部分符号引用     |
| CONSTANT_MethodHandle_info       | 15   | 表示方法句柄                 |
| CONSTANT_MethodType_info         | 16   | 表示方法类型                 |
| CONSTANT_Dynamic_info            | 17   | 表示一个动态计算常量         |
| CONSTANT_InvokeDynamic_info      | 18   | 表示一个动态方法调用点       |
| CONSTANT_Module_info             | 19   | 表示一个模块                 |
| CONSTANT_Package_info            | 20   | 表示一个模块开发或者到处的包 |

###  6.1.3 类访问标志  

**在常量池结束之后，紧接着的2个字节代表访问标志（access_flags）**，这个标志用于识别一些类或者接口层次的访问信息， 包括： 这个Class是类还是接口； 是否定义为public类型； 是否定义为abstract类型； 如果是类的话， 是否被声明为final； 等等。 具体的标志位以及标志的含义见下表：

| 标志名称       | 标志值 | 含义                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 是否为public类型                                             |
| ACC_FINAL      | 0x0010 | 是否被声明为final，只有类可以设置                            |
| ACC_SUPER      | 0x0020 | 是否允许invokespecial字节码指令的新语义，JDK1.0.2之后编译出来的类，这个标志都为真 |
| ACC_INTERFACE  | 0x0200 | 标识这是一个接口                                             |
| ACC_ABSTRACT   | 0x0400 | 是否为abstract类型，对于接口或抽象类来说，此标志值为真，其他类型为假 |
| ACC_SYNTHETIC  | 0x1000 | 标识这个类并非由用户代码产生                                 |
| ACC_ANNOTATION | 0x2000 | 标识这是个注解                                               |
| ACC_EUM        | 0x4000 | 标识这是个枚举                                               |
| ACC_MODULE     | 0x8000 | 标识这是个模块                                               |

### 6.1.4 类索引、 父类索引与接口索引集合  

**类索引、 父类索引和接口索引集合都按顺序排列在访问标志之后**， 类索引和父类索引用两个u2类型的索引值表示， 它们各自指向一个类型为CONSTANT_Class_info的类描述符常量， 通过CONSTANT_Class_info类型的常量中的索引值可以找到定义在CONSTANT_Utf8_info类型的常量中的全限定名字符串。  

![](images\JVM03.png)

### 6.1.5 字段表集合 

**字段表（field_info）**用于描述接口或者类中声明的变量。Java中的“字段”（Field）包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。

字段可以包括的修饰符有字段的作用域（public、private protected修饰符）、是实例变量还是类变量（static修饰符）、可变性（final）、并发可见性（volatile修饰符，是否强制从主内存读写）、可否被序列化（transient修饰符）、字段数据类型（基本类型、对象、数组）、字段名称。字段表结构如下：

| 类型 | 名称             | 数量             |
| ---- | ---------------- | ---------------- |
| u2   | access_flags     | 1                |
| u2   | name_index       | 1                |
| u2   | descriptor_index | 1                |
| u2   | attributes_count | 1                |
| u2   | attributes       | attributes_count |

字段修饰符放在access_flags项目中， 它与类的访问标志access_flags是非常类似的， 都是一个u2的数据类型， 其中可以设置的标志位和含义如下表所示：

| 标志名称      | 标志值 | 含义                     |
| ------------- | ------ | ------------------------ |
| ACC_PUBLIC    | 0x0001 | 字段是否public           |
| ACC_PRIVATE   | 0x0002 | 字段是否private          |
| ACC_PROTECTED | 0x0004 | 字段是否protected        |
| ACC_STATIC    | 0x0008 | 字段是否static           |
| ACC_FINAL     | 0x0010 | 字段是否final            |
| ACC_VOLATILE  | 0x0040 | 字段是否volatile         |
| ACC-TRANSIENT | 0x0080 | 字段是否transient        |
| ACC_SYNTHETIC | 0x1000 | 字段是否由编译器自动产生 |
| ACC_ENUM      | 0x4000 | 字段是否enum             |

 由于语法规则的约束，ACC_PUBLIC、ACC_PRIVATE、ACC_PROTECTED三个标志最多只能选择其一，ACC_FINAL、ACC_VOLATILE不能同时选择。接口之中的字段必须有ACC_PUBLIC、ACC_STATIC、ACC_FINAL标志，这些都是由Java本身的语言规则所导致的。 

跟随access_flags标志的是两项索引值： name_index和descriptor_index。 它们都是对常量池项的引用， 分别代表着字段的简单名称以及字段和方法的描述符。   

**简单名称**：指没有类型和参数修饰的方法或者字段名称， 比如一个类中的inc()方法和m字段的简单名称分别就是“inc”和“m”。  

**描述符**：用来描述字段的数据类型、 方法的参数列表（包括数量、 类型以及顺序） 和返回值。 根据描述符规则， 基本数据类型（byte、 char、 double、 float、 int、 long、 short、 boolean） 以及代表无返回值的void类型都用一个大
写字符来表示， 而对象类型则用字符L加对象的全限定名来表示， 详见下表：  

| 标识字符 | 含义                            |
| -------- | ------------------------------- |
| B        | byte类型                        |
| C        | char类型                        |
| D        | double类型                      |
| F        | float类型                       |
| I        | int类型                         |
| J        | long类型                        |
| S        | short类型                       |
| Z        | boolean类型                     |
| V        | 特殊类型void                    |
| L        | 对象类型，例如Ljava/lang/Object |

对于数组类型， 每一维度将使用一个前置的“[”字符来描述， 如一个定义为“java.lang.String[][]”类型的二维数组将被记录成“[[Ljava/lang/String；”，一个整型数组“int[]”将被记录成" [I "。

![](images\JVM04.png)

如上图所示，字段表集合从地址0x000000F8开始，第一个u2类型的数据为容量计数器fields_count， 其值为0x0001， 说明这个类只有一个字段表数据。 紧跟着容量计数器的是access_flags标志， 值为0x0002， 代表private修饰符的ACC_PRIVATE标志位为真（ACC_PRIVATE标志的值为0x0002）。 代表字段名称的name_index的值为0x0005， 从常量表中可查得第五项常量是一个CONSTANT_Utf8_info类型的字符串， 其值为“m”， 代表字段描述符的descriptor_index的值为0x0006， 指向常量池的字符串“I”。 根据这些信息， 我们可以推断出原代码定义的字段为“private int m； ”。 

**注意**：在Java语言中字段是无法重载的， 两个字段的数据类型、 修饰符不管是否相同， 都必须使用不一样的名称， 但是对于Class文件格式来讲， 只要两个字段的描述符不是完全相同， 那字段重名就是合法的。   

### 6.1.6 方法表集合  

方法表的结构如同字段表一样， 依次包括**访问标志(access_flags) 、 名称索引(name_index) 、 描述符索引(descriptor_index) 、 属性表集合(attributes)**几项， 如下表所示：

| 类型           | 名称             | 数量             |
| -------------- | ---------------- | ---------------- |
| u2             | access_flags     | 1                |
| u2             | name_index       | 1                |
| u2             | descriptor_index | 1                |
| u2             | attributes_count | 1                |
| attribute_info | attributes       | attributes_count |

 对于方法表， 访问标志位access_flags的取值可参见下表：

| 标志名称         | 标志值 | 含义                             |
| ---------------- | ------ | -------------------------------- |
| ACC_PUBLIC       | 0x0001 | 方法是否public                   |
| ACC_PRIVATE      | 0x0002 | 方法是否private                  |
| ACC_PROTECTED    | 0x0004 | 方法是否protected                |
| ACC_STATIC       | 0x0008 | 方法是否static                   |
| ACC_FINAL        | 0x0010 | 方法是否final                    |
| ACC_SYNCHRONIZED | 0x0020 | 方法是否synchronized             |
| ACC-BRIDGE       | 0x0040 | 方法是不是由编译器产生的桥接方法 |
| ACC-VARARGS      | 0x0080 | 方法是否接收不定参数             |
| ACC_NATIVE       | 0x0100 | 方法是否native                   |
| ACC_ABSTRACT     | 0x0400 | 方法是否abstract                 |
| ACC_STRICT       | 0x0800 | 方法是否strictfp                 |
| ACC-SYNTHETIC    | 0x1000 | 方法是否编译器自动产生           |

在Java语言中，要**重载（Overload）一个方法**，除了要与原方法具有相同的简单名称之外，还要求**必须拥有一个与原方法不同的特征签名**。**特征签名**是指一个方法中各个参数在常量池中的字段符号引用的集合，也正是因为**返回值不会包含在特征签名之中**，所以Java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的。   

### 6.1.7 属性表集合  

对于每一个属性，它的名称都要从常量池中引用一个CONSTANT_Utf8_info类型的常量来表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可。一个符合规则的属性表应该满足下表所定义的结构：

| 类型 | 名称                 | 数量             |
| ---- | -------------------- | ---------------- |
| u2   | arrtibute_name_index | 1                |
| u4   | arrtibute_length     | 1                |
| u1   | info                 | arrtibute_length |

#### 1. Code属性

Java程序方法体里面的代码经过Javac编译器处理之后， 最终变为字节码指令存储在Code属性内。结构如下：

| 类型           | 名称                   | 数量                   |
| -------------- | ---------------------- | ---------------------- |
| u2             | arrtibute_name_index   | 1                      |
| u4             | arrtibute_length       | 1                      |
| u2             | max_stack              | 1                      |
| u2             | max_locals             | 1                      |
| u4             | code_length            | 1                      |
| u1             | code                   | code_length            |
| u2             | exception_table_length | 1                      |
| exception_info | exception_table        | exception_table_length |
| u2             | arrtibutes_count       | 1                      |
| arrtibute_info | arrtibutes             | arrtibutes_count       |

attribute_name_index是一项指向CONSTANT_Utf8_info型常量的索引，此常量值**固定为“Code”**，它代表了该属性的属性名称，attribute_length指示了属性值的长度，由于属性名称索引与属性长度一共为6个字节，所以属性值的长度固定为整个属性表长度减去6个字节。max_stack代表了操作数栈（Operand Stack）深度的最大值。max_locals代表了局部变量表所需的存储空间。   

Code属性是Class文件中最重要的一个属性， 如果把一个Java程序中的信息分为代码（Code， 方法体里面的Java代码） 和元数据（Metadata， 包括类、 字段、 方法定义及其他信息） 两部分， 那么在整个Class文件里， Code属性用于描述代码， 所有的其他数据项目都用于描述元数据。  

#### 2. 异常表集合

**在字节码指令之后的是这个方法的显式异常处理表**（简称“异常表”） 集合， 异常表对于Code属性来说并不是必须存在的，如存在，结构应如下表所示：

| 类型 | 名称       | 数量 |
| ---- | ---------- | ---- |
| u2   | start_pc   | 1    |
| u2   | end_pc     | 1    |
| u2   | handler_pc | 1    |
| u2   | catch_type | 1    |

《Java虚拟机规范》 中明确要求**Java语言的编译器应当选择使用异常表而不是通过跳转指令来实现Java异常及finally处理机制。**

```java
public int inc() {
    int x;
    try {
    	x = 1;
    	return x;
    } catch (Exception e) {
        x = 2;
        return x;
    } finally {
    	x = 3;
    }
}
```

编译后的ByteCode字节码及异常表：

```java
Code:
Stack=1, Locals=5, Args_size=1
0: iconst_1 // try块中的x=1
1: istore_1
2: iload_1 // 保存x到returnValue中， 此时x=1
3: istore 4
5: iconst_3 // finaly块中的x=3
6: istore_1
7: iload 4 // 将returnValue中的值放到栈顶， 准备给ireturn返回
9: ireturn
10: astore_2 // 给catch中定义的Exception e赋值， 存储在变量槽 2中
11: iconst_2 // catch块中的x=2
12: istore_1
13: iload_1 // 保存x到returnValue中， 此时x=2
14: istore 4
16: iconst_3 // finaly块中的x=3
17: istore_1
18: iload 4 // 将returnValue中的值放到栈顶， 准备给ireturn返回
20: ireturn
21: astore_3 // 如果出现了不属于java.lang.Exception及其子类的异常才会走到这里
22: iconst_3 // finaly块中的x=3
23: istore_1
24: aload_3 // 将异常放置到栈顶， 并抛出
25: athrow
Exception table:
from to target type
0 5 10 Class java/lang/Exception
0 5 21 any
10 16 21 any
```

字节码中第0～4行所做的操作就是将整数1赋值给变量x， 并且将此时x的值复制一份副本到最后一个本地变量表的变量槽中（这个变量槽里面的值在ireturn指令执行前将会被重新读到操作栈顶， 作为方法返回值使用。 为了讲解方便， 笔者给这个变量槽起个名字： returnValue） 。 如果这时候没有出现异常， 则会继续走到第5～9行， 将变量x赋值为3， 然后将之前保存在returnValue中的整数1读入到操作栈顶， 最后ireturn指令会以int形式返回操作栈顶中的值， 方法结束。 如果出现了异常， PC寄存器指针转到第10行， 第10～20行所做的事情是将2赋值给变量x， 然后将变量x此时的值赋给returnValue， 最后再将变量x的值改为3。 方法返回前同样将returnValue中保留的整数2读到了操作栈顶。 从第21行开始的代码， 作用是将变量x的值赋为3， 并将栈顶的异常抛出， 方法结束。 

#### 3. Exceptions属性 

Exceptions属性是在方法表中与Code属性平级的一项属性，作用是列举出**方法中可能抛出的受查异常(Checked Excepitons)**，也就是方法描述时在throws关键字后面列举的异常。 它的结构见下表：

| 类型 | 名称                  | 数量                 |
| ---- | --------------------- | -------------------- |
| u2   | arrtibute_name_index  | 1                    |
| u4   | arrtibute_length      | 1                    |
| u2   | number_of_exceptions  | 1                    |
| u2   | exception_index_table | number_of_exceptions |

此属性中的number_of_exceptions项表示方法可能抛出number_of_exceptions种受查异常， 每一种受查异常使用一个exception_index_table项表示； exception_index_table是一个指向常量池中CONSTANT_Class_info型常量的索引， 代表了该受查异常的类型。  

#### 4.LineNumberTable属性  

LineNumberTable属性用于描述Java源码行号与字节码行号（字节码的偏移量） 之间的对应关系，属性的结构如下表 ：

| 类型             | 名称                     | 数量                     |
| ---------------- | ------------------------ | ------------------------ |
| u2               | attribute_name_index     | 1                        |
| u4               | attribute_length         | 1                        |
| u2               | line_number_table_length | 1                        |
| line_number_info | line_number_table        | line_number_table_length |

 line_number_info表包含start_pc和line_number两个u2类型的数据项， 前者是字节码行号， 后者是Java源
码行号。 

#### 5.LocalVariableTable及LocalVariableTypeTable属性  

**LocalVariableTable**属性用于描述栈帧中**局部变量表的变量与Java源码中定义的变量之间的关系**，属性的结构如下表：

| 类型                | 名称                        | 数量                        |
| ------------------- | --------------------------- | --------------------------- |
| u2                  | attribute_name_index        | 1                           |
| u4                  | attribute_length            | 1                           |
| u2                  | local_variable_table_length | 1                           |
| local_variable_info | local_variable_table        | local_variable_table_length |

 local_variable_info项目代表了一个栈帧与源码中的局部变量的关联， 结构如下表：  

| 类型 | 名称             | 数量 |
| ---- | ---------------- | ---- |
| u2   | start_pc         | 1    |
| u2   | length           | 1    |
| u2   | name_index       | 1    |
| u2   | descriptor_index | 1    |
| u2   | index            | 1    |

start_pc和length属性分别代表了这个局部变量的生命周期开始的字节码偏移量及其作用范围覆盖的长度， 两者结合起来就是这个局部变量在字节码之中的作用域范围。name_index和descriptor_index都是指向常量池中CONSTANT_Utf8_info型常量的索引， 分别代表了局部变量的名称以及这个局部变量的描述符。index是这个局部变量在栈帧的局部变量表中变量槽的位置。  

#### 6. ConstantValue属性 

ConstantValue**属性的作用是通知虚拟机自动为静态变量赋值**。只有被static关键字修饰的变量（类变量） 才可以使用这项属性，属性结构如下表所示：

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |
| u2   | constantvalue_index  | 1    |

可以看出ConstantValue属性是一个定长属性， 它的attribute_length数据项值必须固定为2。 constantvalue_index数据项代表了常量池中一个字面量常量的引用， 根据字段类型的不同， 字面量可以是CONSTANT_Long_info、 CONSTANT_Float_info、 CONSTANT_Double_info、CONSTANT_Integer_info和CONSTANT_String_info常量中的一种。  

#### 7. InnerClasses属性 

InnerClasses属性用于记录内部类与宿主类之间的关联。 如果一个类中定义了内部类， 那编译器将会为它以及它所包含的内部类生成InnerClasses属性。 InnerClasses属性的结构如下表所示：

| 类型             | 名称                 | 数量              |
| ---------------- | -------------------- | ----------------- |
| u2               | attribute_name_index | 1                 |
| u4               | attribute_length     | 1                 |
| u2               | number_of_classes    | 1                 |
| inner_class_info | inner_classes        | number_of_classes |

number_of_classes代表需要记录多少个内部类信息， 每一个内部类的信息都由一个inner_classes_info表进行描述。 inner_classes_info表的结构如下：

| 类型 | 名称                     | 数量 |
| ---- | ------------------------ | ---- |
| u2   | inner_class_info_index   | 1    |
| u2   | outer_class_info_index   | 1    |
| u2   | inner_name_index         | 1    |
| u2   | inner_class_access_flags | 1    |

inner_class_info_index和outer_class_info_index都是指向常量池中CONSTANT_Class_info型常量的索引， 分别代表了内部类和宿主类的符号引用。inner_name_index是指向常量池中CONSTANT_Utf8_info型常量的索引， 代表这个内部类的名称，如果是匿名内部类， 这项值为0。inner_class_access_flags是内部类的访问标志， 类似于类的access_flags， 它的取值范围如下表所示：

| 标志名称       | 标志值 | 含义                     |
| -------------- | ------ | ------------------------ |
| ACC_PUBLIC     | 0x0001 | 内部类是否public         |
| ACC_PRIVATE    | 0x0002 | 内部类是否private        |
| ACC_PROTECTED  | 0x0004 | 内部类是否protected      |
| ACC_STATIC     | 0x0008 | 内部类是否static         |
| ACC_FINAL      | 0x0010 | 内部类是否final          |
| ACC_INTERFACE  | 0x0020 | 内部类是否为接口         |
| ACC_ABSTRACT   | 0x0400 | 内部类是否为abstract     |
| ACC_SYNTHETIC  | 0x1000 | 字段是否由编译器自动产生 |
| ACC_ANNOTATION | 0x2000 |                          |
| ACC_ENUM       | 0x4000 | 字段是否enum             |

#### 8.StackMapTable属性

StackMapTable属性中包含零至多个栈映射帧（Stack Map Frame），每个栈映射帧都显式或隐式地代表了一个字节码偏移量， 用于表示执行到该字节码时局部变量表和操作数栈的验证类型。 类型检查验证器会通过检查目标方法的局部变量和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束。 StackMapTable属性的结构如下表所示。

| 类型            | 名称                    | 数量              |
| --------------- | ----------------------- | ----------------- |
| u2              | attribute_name_index    | 1                 |
| u4              | attribute_length        | 1                 |
| u2              | number_of_entries       | 1                 |
| stack_map_frame | stack_map_frame_entries | number_of_entries |

如果方法的Code属性中没有附带StackMapTable属性， 那就意味着它带有一个隐式的StackMap属性， 这个StackMap属性的作用等同于number_of_entries值为0的StackMapTable属性。 一个方法的Code属性最多只能有一个StackMapTable属性， 否则将抛出ClassFormatError异常。 

#### 9.BootstrapMethods属性  

BootstrapMethods属性在JDK 7时增加到Class文件规范之中， 它是一个复杂的变长属性， 位于类文件的属性表中。 这个属性用于保存invokedynamic指令引用的引导方法限定符。  BootstrapMethods属性的结构如下表所示：

| 类型             | 名称                     | 数量                     |
| ---------------- | ------------------------ | ------------------------ |
| u2               | attribute_name_index     | 1                        |
| u4               | attribute_length         | 1                        |
| u2               | number_bootstrap_methods | 1                        |
| bootstrap_method | bootstrap_methods        | number_bootstrap_methods |

  引用到的bootstrap_method结构如下表所示：

| 类型 | 名称                    | 数量                    |
| ---- | ----------------------- | ----------------------- |
| u2   | bootstrap_method_ref    | 1                       |
| u2   | num_bootstrap_arguments | 1                       |
| u2   | bootstrap_arguments     | num_bootstrap_arguments |

 BootstrapMethods属性里， num_bootstrap_methods项的值给出了bootstrap_methods[]数组中的引导方法限定符的数量。 而**bootstrap_methods[]数组**的每个成员包含了一个指向常量池CONSTANT_MethodHandle结构的索引值， 它代表了一个引导方法。   

bootstrap_methods[]数组的每个成员必须包含以下三项内容：

> **bootstrap_method_ref**： 值必须是一个对常量池的有效索引。 常量池在该索引处的值必须是一个CONSTANT_MethodHandle_info结构。
>
> **num_bootstrap_arguments**： num_bootstrap_arguments项的值给出了bootstrap_argu-ments[]数组成员的数量。
>
> **bootstrap_arguments[]**： 每个成员必须是一个对常量池的有效索引。常量池在该索引出必须是下列结构之一： CONSTANT_String_info、 CONSTANT_Class_info、CONSTANT_Integer_info、CONSTANT_Long_info、 CONSTANT_Float_info、CONSTANT_Double_info、 CONSTANT_MethodHandle_info或CONSTANT_MethodType_info。  

#### 10.MethodParameters属性 

MethodParameters是在JDK 8时新加入到Class文件格式中的， 它是一个用在方法表中的变长属性，作用是记录方法的各个形参名称和信息。结构如下：

| 类型      | 名称                 | 数量             |
| --------- | -------------------- | ---------------- |
| u2        | attribute_name_index | 1                |
| u4        | attribute_length     | 1                |
| u1        | parameters_count     | 1                |
| parameter | parameters           | parameters_count |

引用到的parameter结构如下：

| 类型 | 名称         | 数量 |
| ---- | ------------ | ---- |
| u2   | name_index   | 1    |
| u2   | access_flags | 1    |

name_index是一个指向常量池CONSTANT_Utf8_info常量的索引值， 代表了该参数的名称。 

access_flags是参数的状态指示器， 它可以包含以下三种状态中的一种或多种：
	0x0010（ACC_FINAL） ： 表示该参数被final修饰。
	0x1000（ACC_SYNTHETIC） ： 表示该参数并未出现在源文件中， 是编译器自动生成的。
	0x8000（ACC_MANDATED） ： 表示该参数是在源文件中隐式定义的。 Java语言中的典型场景是this关键字。  

## 6.2 字节码指令简介  

Java虚拟机的指令由一个字节长度的、 代表着某种特定操作含义的数字（称为**操作码**， **Opcode**）以及跟随其后的零至多个代表此操作所需的参数（称为**操作数**，**Operand**） 构成。Java虚拟机操作码的长度为一个字节（即0～255） ， 因此指令集的操作码总数不能够超过256条 。

### 6.2.1 字节码与数据类型  

在Java虚拟机的指令集中， 大多数指令都包含其操作所对应的数据类型信息。i代表对int类型的数据操作，l代表long，s代表short，b代表byte，c代表char，f代表float，d代表double，a代表reference。

Java虚拟机指令集所支持的数据类型如下：

![](images\JVM05.png)

![](images\JVM06.png)

编译器会在编译期或运行期对于boolean、byte、short和char类型数据的大多数操作， 实际上都是使用相应的对int类型作为运算类型（Computational Type）来进行的。  

### 6.2.2 加载和存储指令 

加载和存储指令用于将数据在栈帧中的局部变量表和操作数栈（见第2章关于内存区域的介绍） 之间来回传输， 这类指令包括：

1. 将一个局部变量加载到操作栈： iload、lload、fload、dload、aload
2. 将一个数值从操作数栈存储到局部变量表： istore、lstore、fstore、dstore、astore
3. 将一个常量加载到操作数栈： bipush、 sipush、 ldc、 ldc_w、 ldc2_w、 aconst_null、 iconst_m1、iconst_<i>、 lconst<l>、 fconst<f>、 dconst<d>
4. 扩充局部变量表的访问索引的指令： wide    

指令助记符中有一部分是以尖括号结尾的，这些指令助记符实际上代表了一组指令（例如iload_<n>， 它代表了iload_0、 iload_1、 iload_2和iload_3这几条指令）。

### 6.2.3 运算指令 

1. 加法指令： iadd、 ladd、 fadd、 dadd
2. 减法指令： isub、 lsub、 fsub、 dsub
3. 乘法指令： imul、 lmul、 fmul、 dmul
4. 除法指令： idiv、 ldiv、 fdiv、 ddiv
5. 求余指令： irem、 lrem、 frem、 drem
6. 取反指令： ineg、 lneg、 fneg、 dneg
7. 位移指令： ishl、 ishr、 iushr、 lshl、 lshr、 lushr
8. 按位或指令： ior、 lor
9. 按位与指令： iand、 land
10. 按位异或指令： ixor、 lxor
11. 局部变量自增指令： iinc
12. 比较指令： dcmpg、 dcmpl、 fcmpg、 fcmpl、 lcmp  

### 6.2.4 类型转换指令  

类型转换指令可以将两种不同的数值类型相互转换，Java虚拟机直接支持（即转换时无须显式的转换指令） 以下数值类型的**宽化类型转换**（**WideningNumeric Conversion**， 即小范围类型向大范围类型的安全转换） ：

1. int -> long、 float、double
2. long -> float、 double
3. float -> double

处理**窄化类型转换**（**Narrowing Numeric Conversion**）时，就必须显式地使用转换指令来完成，这些转换指令包括i2b、i2c、i2s、l2i、f2i、f2l、d2i d2l和d2f。

### 6.2.5 对象创建与访问指令  

1. 创建类实例的指令： new
2. 创建数组的指令： newarray、 anewarray、 multianewarray
3. ·访问类字段（static字段， 或者称为类变量）和实例字段（非static字段， 或者称为实例变量）的指令：getfield、 putfield、 getstatic、 putstatic
4. 把一个数组元素加载到操作数栈的指令： baload、 caload、 saload、 iaload、 laload、 faload、
   daload、 aaload
5. 将一个操作数栈的值储存到数组元素中的指令： bastore、 castore、 sastore、 iastore、 fastore、
   dastore、 aastore
6. 取数组长度的指令： arraylength
7. 检查类实例类型的指令： instanceof、 checkcast  

### 6.2.6 操作数栈管理指令 

1. 将操作数栈的栈顶一个或两个元素出栈： pop、 pop2
2. 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶： dup、 dup2、 dup_x1、dup2_x1、 dup_x2、 dup2_x2
3. 将栈最顶端的两个数值互换： swap  

### 6.2.7 控制转移指令  

控制转移指令包括：

1. 条件分支： ifeq、 iflt、 ifle、 ifne、 ifgt、 ifge、 ifnull、 ifnonnull、 if_icmpeq、 if_icmpne、 if_icmplt、if_icmpgt、 if_icmple、 if_icmpge、 if_acmpeq和if_acmpne
2. 复合条件分支： tableswitch、 lookupswitch
3. 无条件分支： goto、 goto_w、 jsr、 jsr_w、 ret  

### 6.2.8 方法调用和返回指令 

1. invokevirtual指令： 用于调用对象的实例方法， 根据对象的实际类型进行分派（虚方法分派） ，这是Java语言中最常见的方法分派方式。
2. invokeinterface指令： 用于调用接口方法， 它会在运行时搜索一个实现了这个接口方法的对象， 找出适合的方法进行调用。
3. invokespecial指令： 用于调用一些需要特殊处理的实例方法， 包括实例初始化方法、 私有方法和父类方法。
4. invokestatic指令： 用于调用类静态方法（static方法） 。
5. invokedynamic指令： 用于在运行时动态解析出调用点限定符所引用的方法。 并执行该方法。 前面四条调用指令的分派逻辑都固化在Java虚拟机内部， 用户无法改变， 而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。  

方法返回指令是根据返回值的类型区分的， 包括ireturn（当返回值是boolean、 byte、 char、 short和int类型时使用） 、 lreturn、 freturn、 dreturn和areturn， 另外还有一条return指令供声明为void的方法、 实例初始化方法、 类和接口的类初始化方法使用。

### 6.2.9 同步指令  

Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步， 这两种同步结构都是使用管程（Monitor， 更常见的是直接将它称为“锁”） 来实现的。 

虚拟机可以从方法常量池中的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否被声明为同步方法。 当方法调用时， 调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置， 如果设置了， 执行线程就要求先成功持有管程， 然后才能执行方法， 最后当方法完成（无论是正常完成还是非正常完成） 时释放管程。   

***

# 七、虚拟机加载机制

虚拟机的类加载机制：Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型。

## 7.1 类加载时机

![](images\JVM01.png)
一个类从被加载到虚拟机内存到卸载出内存为止，它的整个生命周期将会经历**加载(Loading)、验证(Verification)、准备(Preparation)、解析(Resolution)、初始化(Initialization)、使用(Using)和卸载(Unloading)**七个阶段，其中**验证、准备、解析三个部分统称为连接(Linking)**。

其中：加载、 验证、 准备、 初始化和卸载这五个阶段的顺序是确定的。

>对于初始化阶段， 《Java虚拟机规范》严格规定了有且只有六种情况必须立即对类进行“初始化”：  
>1) **遇到new、getstatic、putstatic或invokestatic这四条字节码指令**时，如果类型没有进行过初始化，则需要先触发其初始化阶段。  
>2) **使用java.lang.reflect包的方法对类型进行反射调用**的时候，如果类型没有进行过初始化，则需要先触发其初始化。  
>3) 当**初始化类**的时候， 如果发现其**父类还没有进行过初始化**， 则需要先触发其父类的初始化。  
>4) 当虚拟机启动时，用户需要指定一个要执行的主类(**包含main()方法的那个类**)，虚拟机会先初始化这个主类。  
>5) 如果一个java.lang.invoke.MethodHandle实例最后的解析结果为**REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄**，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。 
>6) 当一个接口中定义了JDK 8新加入的默认方法(**被default关键字修饰的接口方法**)时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

通过子类引用父类的静态字段，不会导致子类初始化，实例代码如下:

```java
public class SuperClass {
	static {
		System.out.println("SuperClass init!");
	} 
	public static int value = 123;
} 
public class SubClass extends SuperClass {
	static {
		System.out.println("SubClass init!");
	}
} 

/**
  * 非主动使用类字段演示
  **/
public class NotInitialization {
	public static void main(String[] args) {
		System.out.println(SubClass.value);
	}
}
```
上述代码运行之后，只会输出“SuperClass init！”，对于静态字段，只有直接定义这个字段的类才会被初始化， 因此**通过其子类来引用父类中定义的静态字段， 只会触发父类的初始化而不会触发子类的初始化**。

通过数组定义来引用类， 不会触发此类的初始化，示例如下：

```java
public class NotInitialization {
	public static void main(String[] args) {
		SuperClass[] sca = new SuperClass[10];
	}
}
```
它是一个由虚拟机自动生成的、直接继承于java.lang.Object的子类，创建动作由字节码指令newarray触发。

常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化，示例如下：

```java
public class ConstClass {
	static {
		System.out.println("ConstClass init!");
	} 
	public static final String HELLOWORLD = "hello world";
} 
/**
  * 非主动使用类字段演示
  **/
public class NotInitialization {
	public static void main(String[] args) {
		System.out.println(ConstClass.HELLOWORLD);
	}
}
```
## 7.2 类加载的过程

### 7.2.1 加载

在加载阶段， Java虚拟机需要完成以下三件事情：  

1） 通过一个类的全限定名来获取定义此类的二进制字节流。  

2） 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。  

3） 在内存中生成一个代表这个类的java.lang.Class对象， 作为方法区这个类的各种数据的访问入口。  

加载阶段既可以使用Java虚拟机里内置的引导类加载器来完成，也可以由用户自定义的类加载器去完成。对于数组类而言，本身不通过类加载器创建， 它是由Java虚拟机直接在内存中动态构造出来的。 

但数组类的元素类型（数组去掉所有维度的类型）最终还是要靠类加载器来完成加载，一个数组类（下面简称为C）创建过程遵循以下规则： 

>1.数组的组件类型（ Component Type， 指的是数组去掉一个维度的类型， 注意和前面的元素类型区分开来） 是引用类型， 那就递归采用本节中定义的加载过程去加载这个组件类型， 数组C将被标识在加载该组件类型的类加载器的类名称空间上。  
>
>2.如果数组的组件类型不是引用类型（ 例如int[]数组的组件类型为int），Java虚拟机将会把数组C标记为与引导类加载器关联。  
>
>3.**数组类的可访问性与它的组件类型的可访问性一致**， 如果组件类型不是引用类型， 它的数组类的可访问性将默认为public， 可被所有的类和接口访问到。

### 7.2.2 验证

验证是连接阶段的第一步，这一阶段的目的是确保Class文件的字节流中包含的信息符合《Java虚拟机规范》 的全部约束要求，保证这些信息被当作代码运行后不会危害虚拟机自身的安全。

验证阶段大致上会完成下面四个阶段的检验动作： 文件格式验证、 元数据验证、 字节码验证和符号引用验证。

#### 1.文件格式验证

验证字节流是否符合Class文件格式的规范， 并且能被当前版本的虚拟机处理。例如是否以魔数0xCAFEBABE开头、主、次版本号是否在当前Java虚拟机接受范围之内等等。

主要目的是**保证输入的二进制字节流能正确地解析并存储于方法区之内**， 格式上符合描述一个Java类型信息的要求。

#### 2.元数据验证

对字节码描述的信息进行语义分析，以保证其描述的信息符合《Java语言规范》的要求，例如：这个类是否有父类（除了java.lang.Object之外， 所有的类都应当有父类）、这个类的父类是否继承了不允许被继承的类（被final修饰的类）等等。

主要目的是**对类的元数据信息进行语义校验**，保证不存在与《Java语言规范》定义相悖的元数据信息。

#### 3.字节码验证

主要目的是通过数据流分析和控制流分析，确定程序语义是合法的、符合逻辑的。在第二阶段对元数据信息中的数据类型校验完毕以后，这阶段就要对**类的方法体（Class文件中的Code属性）进行校验分析**，保证被校验类的方法在运行时不会做出危害虚拟机安全的行为。

例如：保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作， 例如不会出现类似于“在操作栈放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中”这样的情况；保证任何跳转指令都不会跳转到方法体以外的字节码指令上等等。

#### 4.符号引用验证

符号引用验证可以看作是对类自身以外（常量池中的各种符号引用）的各类信息进行匹配性校验，通俗来说就是，该类是否缺少或者被禁止访问它依赖的某些外部类、方法、字段等资源。例如：符号引用中通过字符串描述的全限定名是否能找到对应的类、在指定类中是否存在符合方法的字段描述符及简单名称所描述的方法和字段等等。

符号引用验证的主要目的是确保解析行为能正常执行， 如果无法通过符号引用验证， Java虚拟机将会抛出一个java.lang.IncompatibleClassChangeError的子类异常， 典型的如：java.lang.IllegalAccessError、 java.lang.NoSuchFieldError、 java.lang.NoSuchMethodError等。  

### 7.2.3  准备  

**准备阶段是正式为类中定义的变量（即静态变量， 被static修饰的变量） 分配内存并设置类变量初始值的阶段。**

这时候进行内存分配的仅包括类变量， 而不包括实例变量， 实例变量将会在对象实例化时随着对象一起分配在Java堆中。 其次是这里所说的初始值“通常情况”下是数据类型的零值， 假设一个类变量的定义为：

```java
public static int value = 123;
```

变量value在准备阶段过后的初始值为0而不是123， 因为这时尚未开始执行任何Java方法， 而把value赋值为123的putstatic指令是程序被编译后， 存放于类构造器<clinit>()方法之中， 所以把value赋值为123的动作要到类的初始化阶段才会被执行。   

### 7.2.4 解析 

**解析阶段是Java虚拟机将常量池内的符号引用替换为直接引用的过程。**

**符号引用**（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。 符号引用与虚拟机实现的内存布局无关，引用的目标并不一定是已经加载到虚拟机内存当中的内容。  

**直接引用**（Direct References）：直接引用是可以直接指向目标的指针、相对偏移量或者是一个能间接定位到目标的句柄。  

除invokedynamic指令以外，虚拟机实现可以对第一次解析的结果进行缓存， 譬如在运行时直接引用常量池中的记录，并把常量标识为已解析状态， 从而避免解析动作重复进行。 对于invokedynamic指令， 必须等到程序实际运行到这条指
令时， 解析动作才能进行。   

解析动作主要针对类或接口、 字段、 类方法、 接口方法、 方法类型、 方法句柄和调用点限定符这7类符号引用进行， 分别对应于常量池的**CONSTANT_Class_info**、 **CON-STANT_Fieldref_info**、**CONSTANT_Methodref_info**、 **CONSTANT_InterfaceMethodref_info**、**CONSTANT_MethodType_info**、 **CONSTANT_MethodHandle_info**、 **CONSTANT_Dyna-mic_info**和**CONSTANT_InvokeDynamic_info** 8种常量类型。   

### 7.2.5 初始化  

初始化阶段就是执行类构造器<clinit>()方法的过程。   

- [ ] <clinit>()方法是由编译器自动收集类中的**所有类变量的赋值动作和静态语句块(static{}块)中的语句**合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序决定的， **静态语句块中只能访问到定义在静态语句块之前的变量**， **定义在它之后的变量， 在前面的静态语句块可以赋值， 但是不能访问**。

```java
public class Test {
    static {
        i = 0;
        System.out.print(i); // 这句编译器会提示“非法向前引用”
    }
	static int i = 1;
}
```

- [ ] <clinit>()方法与类的构造函数（即在虚拟机视角中的实例构造器<init>()方法） 不同， 它不需要显式地调用父类构造器， Java虚拟机会保证在子类的<clinit>()方法执行前， 父类的<clinit>()方法已经执行完毕。 因此在Java虚拟机中第一个被执行的<clinit>()方法的类型肯定是java.lang.Object。  

```java
static class Parent {
    public static int A = 1;
    static {
        A = 2;
    }
}

static class Sub extends Parent {
    public static int B = A;
}

public static void main(String[] args) {
    System.out.println(Sub.B);
}
```

- [ ] 父类的<clinit>()方法先执行， 也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作，所以上面代码中字段B的值将会是2而不是1。  

- [ ] <clinit>()方法对于类或接口来说并不是必需的， 如果一个类中没有静态语句块， 也没有对变量的赋值操作， 那么编译器可以不为这个类生成<clinit>()方法。

- [ ] 接口中不能使用静态语句块， 但仍然有变量初始化的赋值操作， 因此接口与类一样都会生成<clinit>()方法。 但执行接口的<clinit>()方法不需要先执行父接口的<clinit>()方法，只有当父接口中定义的变量被使用时， 父接口才会被初始化。 此外， 接口的实现类在初始化时也一样不会执行接口的<clinit>()方法。

- [ ] Java虚拟机必须保证一个类的<clinit>()方法在多线程环境中被正确地加锁同步， 如果多个线程同时去初始化一个类， 那么只会有其中一个线程去执行这个类的<clinit>()方法， 其他线程都需要阻塞等待， 直到活动线程执行完毕<clinit>()方法。

  注意：其他线程虽然会被阻塞， 但如果执行＜clinit＞()方法的那条线程退出＜clinit＞()方法后， 其他线程唤醒后则不会再次进入＜clinit＞()方法。 **同一个类加载器下， 一个类型只会被初始化一次。**  

## 7.3 类加载器 

通过一个类的全限定名来获取描述该类的二进制字节流，Java虚拟机设计团队有意把这个动作放到Java虚拟机外部去实现， 以便让应用程序自己决定如何去获取所需的类。 实现这个动作的代码被称为“类加载器”（Class Loader） 。 

### 7.3.1 类与类加载器   

任意一个类， 都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性。

### 7.3.2 双亲委派模型  

对于Java虚拟机而言，只有两种不同的类加载器： 一种是**启动类加载器(Bootstrap ClassLoader)**，这个类加载器使用C++语言实现， 是虚拟机自身的一部分； 另外一种就是其他所有的类加载器， 这些类加载器都由Java语言实现， 独立存在于虚拟机外部， 并且全都继承自抽象类java.lang.ClassLoader。  

对于Java开发人员而言，Java一直保持着三层类加载器、双亲委派的类加载架构。

#### 1.三层类加载器

**启动类加载器(Bootstrap Class Loader)** ：这个类加载器负责加载存放在**<JAVA_HOME>\lib**目录， 或者被-Xbootclasspath参数所指定的路径中存放的， 而且是Java虚拟机能够识别的（按照文件名识别， 如rt.jar、 tools.jar， 名字不符合的类库即使放在lib目录中也不会被加载） 类库加载到虚拟机的内存中。   

**扩展类加载器(Extension Class Loader)**： 这个类加载器是在类sun.misc.Launcher$ExtClassLoader中以Java代码的形式实现的。 它负责加载**<JAVA_HOME>\lib\ext目录**中， 或者被**java.ext.dirs系统变量所指定的路径**中所有的类库。  

**应用程序类加载器(Application Class Loader)**： 这个类加载器由sun.misc.Launcher$AppClassLoader来实现。它负责加载用户类路径（ClassPath） 上所有的类库。

#### 2.双亲委派机制

![](images\JVM02.png)

图中展示的各种类加载器之间的层次关系被称为类加载器的“双亲委派模型(Parents DelegationModel)”。  

类加载器之间的父子关系一般不是以继承（Inheritance） 的关系来实现的， 而是通常使用组合（Composition） 关系来复用父加载器的代码。  

**双亲委派模型的工作过程**： 如果一个类加载器收到了类加载的请求， 它首先不会自己去尝试加载这个类， 而是把这个请求委派给父类加载器去完成， 每一个层次的类加载器都是如此， 因此所有的加载请求最终都应该传送到最顶层的启动类加载器中， 只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类） 时， 子加载器才会尝试自己去完成加载。  

使用双亲委派模型来组织类加载器之间的关系， 一个显而易见的好处就是Java中的类随着它的类加载器一起具备了一种带有优先级的层次关系。  

双亲委派模型的实现 ：

```java
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
{
    // 首先， 检查请求的类是否已经被加载过了
    Class c = findLoadedClass(name);
    if (c == null) {
        try {
            if (parent != null) {
            	c = parent.loadClass(name, false);
            } else {
            	c = findBootstrapClassOrNull(name);
            }
        }catch (ClassNotFoundException e) {
        // 如果父类加载器抛出ClassNotFoundException
        // 说明父类加载器无法完成加载请求
        }
        if (c == null) {
            // 在父类加载器无法加载时
            // 再调用本身的findClass方法来进行类加载
            c = findClass(name);
        }
    }
    if (resolve) {
    	resolveClass(c);
    }
    return c;
}
```

**代码逻辑**：先检查请求加载的类型是否已经被加载过， 若没有则调用父加载器的loadClass()方法， 若父加载器为空则默认使用启动类加载器作为父加载器。 假如父类加载器加载失败，抛出ClassNotFoundException异常的话， 才调用自己的findClass()方法尝试进行加载。  

## 7.4Java模块化系统  

这部分暂时没有细看，后续补充

# 八、虚拟机字节码执行引擎  

执行引擎在执行字节码的时候， 通常会有解释执行（通过解释器执行） 和编译执行（通过即时编译器产生本地代码执行） 两种选择， 也可能两者兼备， 还可能会有同时包含几个不同级别的即时编译器一起工作的执行引擎。  

## 8.1 运行时栈帧结构  

Java虚拟机以方法作为最基本的执行单元，“栈帧”（Stack Frame）则是用于支持虚拟机进行方法调用和方法执行背后的数据结构，它也是虚拟机运行时数据区中的虚拟机栈（Virtual MachineStack）的栈元素。每一个栈帧都包括了**局部变量表、 操作数栈、 动态连接、 方法返回地址和一些额外的附加信息**。    

### 8.1.1 局部变量表  

局部变量表(Local Variables Table)是一组变量值的存储空间， 用于**存放方法参数和方法内部定义的局部变量**。在Java程序被编译为Class文件时， 就在方法的Code属性的max_locals数据项中确定了该方法所需分配的局部变量表的最大容量。  

对于64位的数据类型，Java虚拟机会以高位对齐的方式为其分配两个连续的变量槽空间。 Java语言中明确的64位的数据类型只有long和double两种。

Java虚拟机通过索引定位的方式使用局部变量表， 索引值的范围是从0开始至局部变量表最大的变量槽数量。 当一个方法被调用时，Java虚拟机会使用局部变量表来完成参数值到参数变量列表的传递过程，即实参到形参的传递。 如果执行的是实例方法(没有被static修饰的方法)， 那局部变量表中**第0位索引的变量槽默认是用于传递方法所属对象实例的引用**， 在方法中可以通过关键字“this”来访问到这个隐含的参数。 **局部变量表中的变量槽是可以重用的。**      

### 8.1.2 操作数栈  

操作数栈(Operand Stack) 也常被称为操作栈，它是一个后入先出(Last In First Out， **LIFO**)栈。在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容， 也就是出栈和入栈操作。 操作数栈中元素的数据类型必须与字节码指令的序列严格匹配， 在编译程序代码的时候， 编译器必须要严格保证这一点， 在类校验阶段的数据流分析中还要再次验证这一点。    

在概念模型中，两个不同栈帧作为不同方法的虚拟机栈的元素，是完全相互独立的。 但大部分虚拟机都进行一个优化处理，让下面栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起。如下图所示：

<img src="images\JVM17.png" style="zoom:67%;" />

### 8.1.3 动态连接

每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接(Dynamic Linking)。  这些符号引用一部分会**在类加载阶段或者第一次使用的时候就被转化为直接引用**， 这种转化被称为**静态解析**。另外一部分将**在每一次运行期间都转化为直接引用**， 这部分就称为**动态连接**。   

### 8.1.4 方法返回地址  

当一个方法开始执行后， 只有两种方式退出：

- 执行引擎遇到任意一个方法返回的字节码指令，这时候可能会有返回值传递给上层的方法调用者（调用当前方法的方法称为调用者或者主调方法）， 方法是否有返回值以及返回值的类型将根据遇到何种方法返回指令来决定， 这种退出方法的方式称为“**正常调用完成**”（Normal Method Invocation Completion）。  

- 在方法执行的过程中遇到了异常， 并且这个异常没有在方法体内得到妥善处理。 无论是Java虚拟机内部产生的异常， 还是代码中使用athrow字节码指令产生的异常， 只要在本方法的异常表中没有搜索到匹配的异常处理器， 就会导致方法退出， 这种退出方法的方式称为“**异常调用完成**（Abrupt Method Invocation Completion） ”。   

方法退出的过程实际上等同于把当前栈帧出栈， 因此退出时可能执行的操作有： 恢复上层方法的局部变量表和操作数栈， 把返回值(如果有的话) 压入调用者栈帧的操作数栈中，调整PC计数器的值以指向方法调用指令后面的一条指令。  

## 8.2 方法调用

### 8.2.1 解析

调用目标在程序代码写好、 编译器进行编译那一刻就已经确定下来。 这类方法的调用被称为解析（Resolution）。

在Java虚拟机支持以下5条方法调用字节码指令， 分别是：

- **invokestatic**。 用于调用静态方法。

- **invokespecial**。 用于调用实例构造器<init>()方法、 私有方法和父类中的方法。

- **invokevirtual**。 用于调用所有的虚方法。

- **invokeinterface**。 用于调用接口方法， 会在运行时再确定一个实现该接口的对象。

- **invokedynamic**。 先在运行时动态解析出调用点限定符所引用的方法， 然后再执行该方法。 前面4条调用指令， 分派逻辑都固化在Java虚拟机内部， 而invokedynamic指令的分派逻辑是由用户设定的引导方法来决定的。

能被invokestatic和invokespecial指令调用的方法， 都可以在解析阶段中确定唯一的调用版本，符合这个条件的方法共有**静态方法、 私有方法、 实例构造器、 父类方法**4种， 再加上**被final修饰的方法**（尽管它使用invokevirtual指令调用） ， 这5种方法调用会在类加载的时候就可以把符号引用解析为该方法的直接引用。 这些方法统称为“**非虚方法**”（Non-Virtual Method） ， 与之相反， 其他方法就被称为“**虚方法**”（Virtual Method） 。  

### 8.2.2 分派  

#### 1.静态分派  

```java
public class StaticDispatch {
    static abstract class Human {
    } 

    static class Man extends Human {
    } 

    static class Woman extends Human {
    } 

    public void sayHello(Human guy) {
        System.out.println("hello,guy!");
    } 

    public void sayHello(Man guy) {
        System.out.println("hello,gentleman!");
    } 

    public void sayHello(Woman guy) {
        System.out.println("hello,lady!");
    } 

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        StaticDispatch sr = new StaticDispatch();
        sr.sayHello(man);
        sr.sayHello(woman);
    }
}
```

以上代码的输出为：

```java
hello,guy!
hello,guy!
```

为什么虚拟机会选择执行参数类型为Human的重载版本？ 

```java
Human man = new Man();
```

上面代码中的“Human”为变量的“**静态类型**”（Static Type）， 或者叫“**外观类型**”（Apparent Type），后面的“Man”则被称为变量的“**实际类型**”（Actual Type） 或者叫“**运行时类型**”（Runtime Type）。静态类型和实际类型在程序中都可能会发生变化，区别是静态类型的变化仅仅在使用时发生，变量本身的静态类型不会被改变，并且最终的静态类型是在编译期可知的；而实际类型变化的结果在运行期才可确定。

回到最开始部分的代码，main()里面的两次sayHello()方法调用， 在方法接收者已经确定是对象“sr”的前提下， 使用哪个重载版本， 就完全取决于传入参数的数量和数据类型。 代码中故意定义了两个静态类型相同， 而实际类型不同的变量， 但虚拟机（或者准确地说是编译器） 在重载时是通过参数的静态类型而不是实际类型作为判定依据的。   

所有依赖静态类型来决定方法执行版本的分派动作， 都称为静态分派。 静态分派的最典型应用表现就是方法重载。 静态分派发生在编译阶段， 因此确定静态分派的动作实际上不是由虚拟机来执行的。

#### 2.动态分派  

```java
public class DynamicDispatch {
    static abstract class Human {
    	protected abstract void sayHello();
    } 
    
    static class Man extends Human {
        @Override
        protected void sayHello() {
        System.out.println("man say hello");
    	}
    } 
    
    static class Woman extends Human {
        @Override
        protected void sayHello() {
        	System.out.println("woman say hello");
        }
    } 
    
    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }
}
```

显然这里选择调用的方法版本是不可能再根据静态类型来决定的，因为静态类型同样都是Human的两个变量man和woman在调用sayHello()方法时产生了不同的行为，甚至变量man在两次调用中还执行了两个不同的方法。 

通过字节码去分析：

```java
public static void main(java.lang.String[]);
    Code:
        Stack=2, Locals=3, Args_size=1
        0: new #16; //class org/fenixsoft/polymorphic/DynamicDispatch$Man
        3: dup
        4: invokespecial #18; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Man."<init>":()V
        7: astore_1
        8: new #19; //class org/fenixsoft/polymorphic/DynamicDispatch$Woman
        11: dup
        12: invokespecial #21; //Method org/fenixsoft/polymorphic/DynamicDispatch$Woman."<init>":()V
        15: astore_2
        16: aload_1
        17: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
        20: aload_2
        21: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
        24: new #19; //class org/fenixsoft/polymorphic/DynamicDispatch$Woman
        27: dup
        28: invokespecial #21; //Method org/fenixsoft/polymorphic/DynamicDispatch$Woman."<init>":()V
        31: astore_1
        32: aload_1
        33: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
        36: return
```

invokevirtual指令的运行时解析过程大致分为以下几步：

1. 找到操作数栈顶的第一个元素所指向的对象的实际类型， 记作C。
2. 如果在类型C中找到与常量中的描述符和简单名称都相符的方法， 则进行访问权限校验， 如果通过则返回这个方法的直接引用， 查找过程结束； 不通过则返回java.lang.IllegalAccessError异常。
3. 否则， 按照继承关系从下往上依次对C的各个父类进行第二步的搜索和验证过程。
4. 如果始终没有找到合适的方法， 则抛出java.lang.AbstractMethodError异常。  

invokevirtual指令执行的第一步就是在运行期确定接收者的实际类型， 所以两次调用中的invokevirtual指令并不是把常量池中方法的符号引用解析到直接引用上就结束了， 还会根据方法接收者的实际类型来选择方法版本， 这个过程就是Java语言中方法重写的本质。 我们把这种**在运行期根据实际类型确定方法执行版本的分派过程称为动态分派**。  

```java
public class FieldHasNoPolymorphic {
    static class Father {
        public int money = 1;
        public Father() {
            money = 2;
            showMeTheMoney();
    	} 
        public void showMeTheMoney() {
        	System.out.println("I am Father, i have $" + money);
        }
    } 
    static class Son extends Father {
        public int money = 3;
        public Son() {
        	money = 4;
            showMeTheMoney();
        } 
        public void showMeTheMoney() {
    		System.out.println("I am Son, i have $" + money);
    	}
    } 
    
    public static void main(String[] args) {
        Father gay = new Son();
        System.out.println("This gay has $" + gay.money);
    }
}
```

输出的结果为：

```java
I am Son, i have $0
I am Son, i have $4
This gay has $2
```

输出两句都是“I am Son”，这是因为Son类在创建的时候，首先隐式调用了Father的构造函数，而Father构造函数中对showMeTheMoney()的调用是一次**虚方法调用**，实际执行的版本是Son::showMeTheMoney()方法。同样的，由于执行Father构造函数时son的money字段还未进行初始化，所以Father构造函数执行showMeTheMoney()时，son的money字段为0。后续son的构造函数执行时，money字段被赋值为4。

#### 3.虚拟机动态分派的实现

如今（直至Java 12和预览版的Java 13）的Java语言是一门静态多分派、动态单分派的语言。

动态分派是执行非常频繁的动作， 而且动态分派的方法版本选择过程需要运行时在接收者类型的方法元数据中搜索合适的目标方法， 因此， Java虚拟机实现基于执行性能的考虑， 真正运行时一般不会如此频繁地去反复搜索类型元数据。 

面对这种情况，一种基础而且常见的优化手段是为类型在方法区中建立一个**虚方法表**（Virtual Method Table， 也称为vtable，与此对应的，在invokeinterface执行时也会用到**接口方法表**——Interface Method Table， 简称itable） ，使用虚方法表索引来代替元数据查找以提高性能。

<img src="images\JVM18.png" style="zoom:67%;" />

如果某个方法在子类中没有被重写， 那子类的虚方法表中的地址入口和父类相同方法的地址入口是一致的， 都指向父类的实现入口。 如果子类中重写了这个方法， 子类虚方法表中的地址也会被替换为指向子类实现版本的入口地址。   

查虚方法表是分派调用的一种优化手段， 由于Java对象里面的方法默认（即不使用final修饰） 就是虚方法， 虚拟机除了使用虚方法表之外， 为了进一步提高性能， 还会使用类型继承关系分析（Class Hierarchy Analysis， CHA） 、 守护内联（Guarded Inlining） 、 内联缓存（Inline Cache） 等多种非稳定的激进优化来争取更大的性能空间。

