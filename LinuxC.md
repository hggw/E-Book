# 一、C语言入门

## 1 自定义函数

**函数原型**  

>声明了一个函数的名字、参数类型和个数、返回值类型，这称为函数原型  

**函数声明**  
>在代码中可以单独写一个函数原型，后面加;号结束，而不写函数体  

**函数定义**  
>只有带函数体的声明才叫定义  

一个较为简单的例子
    #include <stdio.h>

	void newline(void){
		printf("\n");
	}
	
	void threeline(void){
		newline();
		newline();
		newline();
	}
	
	int main(void){
		printf("Three lines:\n");
		threeline();
		printf("Another three lines.\n");
		threeline();
		return 0;
	}

在上面的例子中，main调用threeline，threeline再调用newline，要保证每个函数的原型出现在调用之前，就只能按先newline，再threeline，再main的顺序定义。  

如果使用不带函数体的声明，则可以改变函数的定义顺序：  

    #include <stdio.h>
    
    int main(void){
    	printf("Three lines:\n");
    	threeline();
    	printf("Another three lines.\n");
    	threeline();
    	return 0;
    }
    
    void newline(void){
    	printf("\n");
    }
    
    void threeline(void){
    	newline();
    	newline();
    	newline();
    }

编译时会报警告：  

    $ gcc main.c  
    main.c:17: warning: conflicting types for ‘threeline’  
    main.c:6: warning: previous implicit declaration of ‘threeline’ was here  

在main函数中调用threeline时并没有声明它，编译器认为此处隐式声明了int threeline(void)。然后为这个调用生成相应的指令，隐式声明的参数类型和个数根据函数调用代码来确定，隐式声明的返回值类型总是int。然后编译器接着往下看，看到threeline函数的原型是void threeline(void)，和先前的隐式声明的返回值类型不符，所以才报这个警告。

## 2 goto语句

如果在一个嵌套循环中遇到某个错误条件需要立即跳到循环之外的某个地方做出错处理，就可以用goto语句。  

goto语句可以从程序中的任何地方，无条件跳转到任何其它地方，只要给那个地方起个标号就行，唯一的限制是goto只能跳到同一个函数的某个标号处，而不能跳到别的函数里。

    for (...)
    for (...) {
    	...
    	if (出现错误条件)
    		goto error;
    }
    error:
    	出错处理;
这里的error:叫做标号（Label）。  

通常goto语句只用于在函数末尾做出错处理（例如释放先前分配的资源、恢复先前改动过的全局变量等），函数中任何地方出现了错误条件都可以立即跳到函数末尾，处理完之后函数返回。比较上面两种写法，用goto语句还是方便很多。但是除了这个用途之外，在任何场合都不要轻易考虑使用goto语句。

## 3 结构体

### 3.1 定义结构体

​    struct complex_struct {
​		double x, y;
​	};
这样定义了complex_struct这个标识符，这种标识符在C语言中称为Tag，整个可以看作一个类型名，就像int或double一样，只不过它是一个复合类型，如果用这个类型名来定义变量，可以这样写：  

    struct complex_struct {
    	double x, y;
    } z1, z2;

也可以这样定义：  

    struct complex_struct z3, z4;
### 3.2 访问结构体

结构体内的成员可以用.运算符（.号，Period）来访问。

    struct complex_struct z = { 3.0, 4.0 };

Initializer中的数据依次赋给结构体的成员。如果Initializer中的数据比结构体的成员多，编译器会报错，但如果只是末尾多个逗号不算错。如果Initializer中的数据比结构体的成员少，未指定的成员将用0来初始化，就像未初始化的全局变量一样。  

结构体可以当作函数的参数和返回值来传递：  

    struct complex_struct add_complex(struct complex_struct z1, struct complex_struct z2)
    {
    	z1.x = z1.x + z2.x;
    	z1.y = z1.y + z2.y;
    	return z1;
    }
这个函数实现了两个复数相加，如果在main函数中这样调用：  

    struct complex_struct z = { 3.0, 4.0 };
    z = add_complex(z, z);
调用传参的过程如下图所示：

![avatar](images\LC01.png)

### 3.3 数据类型标志

complex_struct结构体由一个数据类型标志和两个浮点数组成，如果数据类型标志为0，那两个浮点数就表示直角座标，如果数据类型标志为1，那两个浮点数就表示极座标。这样，直角座标和极座标的数据都可以适配（Adapt）到complex_struct结构体中，无需转换和损失精度：  

    enum coordinate_type { RECTANGULAR, POLAR };
    struct complex_struct {
    	enum coordinate_type t;
    	double a, b;
    };
enum coordinate_type表示一个枚举（Enumeration）类型。枚举类型的成员是常量，它们的值编译器自动分配，例如定义了上面的枚举类型之后，RECTANGULAR就表示常量0，POLAR就表示常量1。如果不希望从0开始分配，可以这样定义：  

    enum coordinate_type { RECTANGULAR = 1, POLAR };
这样，RECTANGULAR就表示常量1，而POLAR就表示常量2，这些常量的类型就是int。需要注意的是，结构体的成员名和变量名不在同一命名空间，但枚举的成员名和变量名在同一命名空间，所以会出现命名冲突。

## 4 数组

​    int count[4];
​	// 定义由4个结构体元素组成的数组
​	struct Complex {
​		double x, y;
​	} a[4];
​	//定义一个包含数组成员的结构体
​	struct {
​		double x, y;
​		int count[4];
​	} s;
对于数组类型有一条特殊规则：数组名做右值使用时，自动转换成指向数组首元素的指针。所以上面的函数调用其实是传一个指针类型的参数，而不是数组类型的参数。
字符串字面值也可以像数组名一样使用，可以加下标访问其中的字符： 

    char c = "Hello, world.\n"[0];
通过下标修改其中的字符却是不允许的：

    "Hello, world.\n"[0] = 'A';

str的后四个元素没有指定，自动初始化为0，即'\0'字符。注意，虽然字符串字面值"Hello"是只读的，但用它初始化的数组str却是可读可写的。

	char str[10] = "Hello";
	char str[10] = { 'H', 'e', 'l', 'l', 'o', '\0' };
如果用于初始化的字符串字面值比数组还长：

    char str[10] = "Hello, world.\n";
printf函数的格式化字符串中可以用%s表示字符串的占位符。但现在字符串可以保存在一个数组里面，用%s来打印就很有必要了：

    printf("string: %s\n", str);

printf会从数组str的开头一直打印到'\0'字符为止（'\0'本身不打印）。如果数组str中没有'\0'，那么printf就会打印出界，有时候打印出乱码，有时候看起来没错误，有时候引起程序崩溃。
###4.1 多维数组
多维数组的初始化：

    int a[3][2] = { 1, 2, 3, 4, 5 };
    // 用嵌套Initializer初始化
    int a[][2] = { { 1, 2 },{ 3, 4 },{ 5, } };

多维数组第一维的长度可以由编译器自动计算，不需要指定，其余各维都必须明确指定长度。如果是字符数组，也可以嵌套使用字符串字面值做Initializer。

## 5 编码风格

1、关键字if, while, for与其后的控制表达式的(括号之间插入一个空格分隔，但括号内的表达式应紧贴括号。  
2、双目运算符的两侧插入一个空格分隔，单目运算符和操作数之间不加空格，例如i␣=␣i␣+␣1、++i、!(i␣<␣1)、-x、&a[1]等。  
3、后缀运算符和操作数之间也不加空格，例如取结构体成员s.a、函数调用foo(arg1)、取数组成员a[i]。  
4、,号和;号之后要加空格，这是英文的书写习惯，例如for␣(i␣=␣1;␣i␣<␣10;␣i++)、foo(arg1,␣arg2)。  
5、以上关于双目运算符和后缀运算符的规则不是严格要求，有时候为了突出优先级也可以写得更紧凑一些，例如for␣(i=1;␣i<10;␣i++)、distance␣=␣sqrt(x*x␣+␣y*y)等。但是省略的空格一定不要误导了读代码的人，例如a||b␣&&␣c很容易让人理解成错误的优先级。  
6、由于标准的Linux终端是24行80列的，接近或大于80个字符的较长语句要折行写，折行后用空格和上面的表达式或参数对齐，例如：  

    if␣(sqrt(x*x␣+␣y*y) > 5.0
        &&␣x␣<␣0.0
        &&␣y␣>␣0.0)
    foo(sqrt(x*x␣+␣y*y),
        a[i-1]␣+␣b[i-1]␣+␣c[i-1])
7、较长的字符串可以断成多个字符串然后分行书写。  

    printf("This is such a long sentence that "
       "it cannot be held within a line\n");

### 5.1 内核关于缩进的规则

1、要用缩进体现出语句块的层次关系，使用Tab字符缩进，不能用空格代替Tab。在标准的Linux终端上，一个Tab看起来是8个空格的宽度，有些编辑器可以设置一个Tab看起来是几个空格的宽度，建议设成8，这样大的缩进使代码看起来非常清晰。规定不能用空格代替Tab主要是不希望空格和Tab混在一起做缩进，如果混在一起用了，在某些编辑器里把Tab的宽度改了就会看起来非常混乱。  
2、if/else、while、do/while、for、switch这些可以带语句块的语句，语句块的{和}应该和关键字写在一起，用空格隔开，而不是单独占一行。  
3、函数定义的{和}单独占一行，这一点和语句块的规定不同。  
4、switch和语句块里的case、default对齐写，也就是说语句块里的case、default相对于switch不往里缩进。自己命名的标号（用于goto）必须顶头写不缩进，而不管标号下的语句缩进到第几层。  
5、代码中每个逻辑段落之间应该用一个空行分隔开。例如每个函数定义之间应该插入一个空行，头文件、全局变量定义和函数定义之间也应该插入空行。  
6、一个函数的语句列表如果很长，也可以根据相关性分成若干组，用空行分隔，这条规定不是严格要求，一般变量定义语句组成一组，后面要加空行，return之前要加空行。

### 5.2 注释

单行注释应用空格把界定符和文字分开。多行注释最常见的是这种形式：

	/*␣单行注释␣*/
	/*
	␣*␣多行注释
	␣*␣多行注释
	␣*/

使用注释的场合主要有：

1、整个源文件的顶部注释。说明此模块的相关信息，例如文件名、作者和版本历史等，顶头写不缩进。  
2、函数注释。说明此函数的功能、参数、返回值、错误码等，写在函数定义上侧，和此函数定义之间不留空行，顶头写不缩进。  
3、相对独立的语句组注释。对这一组语句做特别说明，写在语句组上侧，和此语句组之间不留空行，与当前语句组的缩进一致。注意，说明语句组的注释一定要写在语句组上面，不能写在语句组下面。  
4、代码行右侧的简短注释。对当前代码行做特别说明，一般为单行注释，和代码之间至少用一个空格隔开，一个源文件中所有的右侧注释最好能上下对齐。  
5、复杂的结构体定义比函数更需要注释。  
6、复杂的宏定义和变量定义也需要注释。

### 5.3 标识符命名

1、标识符的命名要清晰明了，可以使用完整的单词和大家易于理解的缩写。短的单词可以通过去元音形成缩写，较长的单词可以取单词的头几个字母形成缩写，也可以采用大家基本认同的缩写。例如count—>cnt，block->blk，length->len，window->win。

2、内核风格规定变量、函数和类型采用全小写加下划线的方式命名，常量（宏定义和枚举常量）采用全大写加下划线的方式命名。上面举例的函数名radix_tree_insert、类型名struct radix_tree_root、常量名RADIX_TREE_MAP_SHIFT等。

3、全局变量和全局函数的命名一定要详细，不惜多用几个单词多写几个下划线，例如函数名radix_tree_insert，因为它们在整个项目的许多源文件中都会用到，必须让使用者明确这个变量或函数是干什么用的。局部变量和只在一个源文件中调用的内部函数的命名可以简略一些，但不能太短，不要使用单个字母做变量名，只有一个例外：用i、j、k做循环变量是可以的。

### 5.4 函数设计

函数设计应遵循以下原则：

1、实现一个函数只是为了做好一件事情，不要把函数设计成用途广泛、面面俱到的，这样的函数肯定会超长，而且往往不可重用，维护困难。

2、函数内部的缩进层次不宜过多，一般以少于4层为宜。如果缩进层次太多就说明设计得太复杂了，应该考虑分割成更小的函数来调用（这称为Helper Function）。

3、函数不要写得太长，建议在24行的标准终端上不超过两屏。

4、执行函数就是执行一个动作，函数名通常应包含动词。

5、比较重要的函数定义上面必须加注释，说此函数的功能、参数、返回值、错误码等。

6、一个函数的局部变量数量控制在5到10个，再多的话应该考虑分割函数。

## 6 gdb——这部分目前用不上，先跳过

## 7 栈与队列

### 7.1 深度优先搜索

算法的伪代码如下：  

    将起点标记为已走过并压栈;
    while (栈非空) {
    	从栈顶弹出一个点p;
    	if (p这个点是终点)
    		break;
    	否则沿右、下、左、上四个方向探索相邻的点，if (和p相邻的点有路可走，并且还没走过)
    		将相邻的点标记为已走过并压栈，它的前趋就是p点;
    }
    if (p点是终点) {
    	打印p点的座标;
    	while (p点有前趋) {
    		p点=p点的前趋;
    		打印p点的座标;
    	}
    } else
    	没有路线可以到达终点;

用深度优先搜索解迷宫问题：

    #include <stdio.h>
    
    #define MAX_ROW 5
    #define MAX_COL 5
    
    struct point { int row, col; } stack[512];
    int top = 0;
    
    void push(struct point p)
    {
    	stack[top] = p;
    	top++;
    }
    
    struct point pop(void)
    {
    	top--;
    	return stack[top];
    }
    
    int is_empty(void)
    {
    	return top == 0;
    }
    
    int maze[MAX_ROW][MAX_COL] = {
    	0, 1, 0, 0, 0,
    	0, 1, 0, 1, 0,
    	0, 0, 0, 0, 0,
    	0, 1, 1, 1, 0,
    	0, 0, 0, 1, 0,
    };
    
    void print_maze(void)
    {
    	int i, j;
    	for (i = 0; i < MAX_ROW; i++) {
    		for (j = 0; j < MAX_COL; j++)
    			printf("%d ", maze[i][j]);
    		putchar('\n');
    	}
    	printf("*********\n");
    }
    
    struct point predecessor[MAX_ROW][MAX_COL] = {
    	{{-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}},
    	{{-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}},
    	{{-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}},
    	{{-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}},
    	{{-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}, {-1,-1}},
    };
    
    void visit(int row, int col, struct point pre)
    {
    	struct point visit_point = { row, col };
    	maze[row][col] = 2;
    	predecessor[row][col] = pre;
    	push(visit_point);
    }
    
    int main(void)
    {
    	struct point p = { 0, 0 };
    
    	maze[p.row][p.col] = 2;
    	push(p);	
    	
    	while (!is_empty()) {
    		p = pop();
    		if (p.row == MAX_ROW - 1  /* goal */
    		    && p.col == MAX_COL - 1)
    			break;
    		if (p.col+1 < MAX_COL     /* right */
    		    && maze[p.row][p.col+1] == 0)
    			visit(p.row, p.col+1, p);
    		if (p.row+1 < MAX_ROW     /* down */
    		    && maze[p.row+1][p.col] == 0)
    			visit(p.row+1, p.col, p);
    		if (p.col-1 >= 0          /* left */
    		    && maze[p.row][p.col-1] == 0)
    			visit(p.row, p.col-1, p);
    		if (p.row-1 >= 0          /* up */
    		    && maze[p.row-1][p.col] == 0)
    			visit(p.row-1, p.col, p);
    		print_maze();
    	}
    	if (p.row == MAX_ROW - 1 && p.col == MAX_COL - 1) {
    		printf("(%d, %d)\n", p.row, p.col);
    		while (predecessor[p.row][p.col].row != -1) {
    			p = predecessor[p.row][p.col];
    			printf("(%d, %d)\n", p.row, p.col);
    		}
    	} else
    		printf("No path!\n");
    
    	return 0;
    }
这次堆栈里的元素是结构体类型的，用来表示迷宫中一个点的x和y座标。我们用一个新的数据结构保存走迷宫的路线，每个走过的点都有一个前趋（Predecessor）的点，表示是从哪儿走到当前点的，比如predecessor[4][4]是座标为(3, 4)的点，就表示从(3, 4)走到了(4, 4)，一开始predecessor的各元素初始化为无效座标(-1, -1)。在迷宫中探索路线的同时就把路线保存在predecessor数组中，已经走过的点在maze数组中记为2防止重复走，最后找到终点时就根据predecessor数组保存的路线从终点打印到起点。

### 7.2 队列与广度优先搜索

用广度优先搜索解迷宫问题:

	#include <stdio.h>
	
	#define MAX_ROW 5
	#define MAX_COL 5
	
	struct point { int row, col, predecessor; } queue[512];
	int head = 0, tail = 0;
	
	void enqueue(struct point p)
	{
		queue[tail] = p;
		tail++;
	}
	
	struct point dequeue(void)
	{
		head++;
		return queue[head-1];
	}
	
	int is_empty(void)
	{
		return head == tail;
	}
	
	int maze[MAX_ROW][MAX_COL] = {
		0, 1, 0, 0, 0,
		0, 1, 0, 1, 0,
		0, 0, 0, 0, 0,
		0, 1, 1, 1, 0,
		0, 0, 0, 1, 0,
	};
	
	void print_maze(void)
	{
		int i, j;
		for (i = 0; i < MAX_ROW; i++) {
			for (j = 0; j < MAX_COL; j++)
				printf("%d ", maze[i][j]);
			putchar('\n');
		}
		printf("*********\n");
	}
	
	void visit(int row, int col)
	{
		struct point visit_point = { row, col, head-1 };
		maze[row][col] = 2;
		enqueue(visit_point);
	}
	
	int main(void)
	{
		struct point p = { 0, 0, -1 };
	
		maze[p.row][p.col] = 2;
		enqueue(p);
		
		while (!is_empty()) {
			p = dequeue();
			if (p.row == MAX_ROW - 1  /* goal */
			    && p.col == MAX_COL - 1)
				break;
			if (p.col+1 < MAX_COL     /* right */
			    && maze[p.row][p.col+1] == 0)
				visit(p.row, p.col+1);
			if (p.row+1 < MAX_ROW     /* down */
			    && maze[p.row+1][p.col] == 0)
				visit(p.row+1, p.col);
			if (p.col-1 >= 0          /* left */
			    && maze[p.row][p.col-1] == 0)
				visit(p.row, p.col-1);
			if (p.row-1 >= 0          /* up */
			    && maze[p.row-1][p.col] == 0)
				visit(p.row-1, p.col);
			print_maze();
		}
		if (p.row == MAX_ROW - 1 && p.col == MAX_COL - 1) {
			printf("(%d, %d)\n", p.row, p.col);
			while (p.predecessor != -1) {
				p = queue[p.predecessor];
				printf("(%d, %d)\n", p.row, p.col);
			}
		} else
			printf("No path!\n");
	
		return 0;
	}

![avatar](images\LC02.png)

这个算法的特点是沿各个方向同时展开搜索，每个可以走通的方向轮流往前走一步，这称为广度优先搜索（BFS，Breadth First Search）。

广度优先每次都从各个方向探索一步，将前线推进一步，图中的虚线就表示这个前线，队列中的元素总是由前线的点组成的。广度优先搜索可以找到从起点到终点的最短路径，而深度优先搜索找到的不一定是最短路径。
###7.3 环形队列
环形队列（Circular Queue）。把queue数组想像成一个圈，head和tail指针仍然是一直增大的，当指到数组末尾时就自动回到数组开头。从head到tail之间是队列的有效元素，从tail到head之间是空的存储位置，head追上tail就表示队列空了，tail追上head就表示队列的存储空间满了。如下图所示：

![avatar](images\LC03.png)

# 二、C语言本质

## 1 数据类型详解

### 1.1 类型转换

在一个表达式中，凡是可以使用int或unsigned int类型做右值的地方也都可以使用有符号或无符号的char型、short型和Bit-field。如果原始类型的取值范围都能用int型表示，则其值被提升为int型，如果表示不了就提升为unsigned int型，这称为Integer Promotion。  

1、如果一个函数的形参类型未知，那么调用函数时要对相应的实参做Integer Promotion，此外，相应的实参如果是float型的也要被提升为double型，这条规则称为Default Argument Promotion。

2、算术运算中的类型转换。两个算术类型的操作数做算术运算，比如a * b，如果两边操作数的类型不同，编译器会自动做类型转换，使两边类型相同之后才做运算，这称为Usual Arithmetic Conversion。

3、单目运算符+、-、~只有一个操作数，移位运算符<<、>>两边的操作数类型不要求一致，因此这些运算不需要做Usual Arithmetic Conversion，但也需要做Integer Promotion。

### 1.2 Usual Arithmetic Conversion的规则

>1、如果有一边的类型是long double，则把另一边也转成long double。

>2、如果有一边的类型是double，则把另一边也转成double。

>3、如果有一边的类型是float，则把另一边也转成float。

如果两边都是整数类型，先对a和b做Integer Promotion，转换后的类型若仍不相同，则需要继续转换。首先规定char、short、int、long、long long的转换级别（Integer Conversion Rank）逐级提高，同一类型的有符号和无符号数具有相同的Rank，然后有如下转换规则：

>1、如果两边都是有符号数，或者都是无符号数，那么较低Rank的类型转换成较高Rank的类型。例如unsigned int和unsigned long做算术运算时都转成unsigned long。

>2、如果一边是无符号数另一边是有符号数，无符号数的Rank不低于有符号数的Rank，则把有符号数转成另一边的无符号类型。例如unsigned long和int做算术运算时都转成unsigned long，unsigned long和long做算术运算时也都转成unsigned long。

>3、一边是无符号数另一边是有符号数，并且无符号数的Rank低于有符号数的Rank。这时又分为两种情况：

>1) 如果这个有符号数类型能够覆盖这个无符号数类型的取值范围，则把无符号数转成另一边的有符号类型。例如遵循LP64的平台上unsigned int和long在做算术运算时都转成long。

>2) 否则，也就是这个符号数类型不足以覆盖这个无符号数类型的取值范围，则把两边都转成两者之中较高Rank的无符号类型。例如遵循ILP32的平台上unsigned int和long在做算术运算时都转成unsigned long。
### 1.3 由赋值产生的类型转换

1) 如果赋值或初始化时等号两边的类型不相同，则编译器会把等号右边的类型转换成等号左边的类型再做赋值。
2) 函数调用传参的过程相当于定义形参并且用实参对其做初始化，函数返回的过程相当于定义一个临时变量并且用return的表达式对其做初始化，所以由赋值产生的类型转换也适用于这两种情况。

例如一个函数的原型是int foo(int, int);，则调用foo(3.1, 4.2)时会自动把两个double型的实参转成int型赋给形参，如果这个函数定义中有返回语句return 1.2;，则返回值1.2会自动转成int型再返回。
###1.4 强制类型转换
可以通过类型转换运算符（Cast Operator）规定某个值要转换成何种类型，这称为显式类型转换（Explicit Conversion）或强制类型转换（Type Cast）。

例如计算表达式(double)3 + i，首先将整数3强制转换成double型3.0，然后和整型变量i相加，这时适用Usual Arithmetic Conversion规则，首先把i也转成double型，然后两者相加，最后整个表达式的值也是double型的。这里的(double)就是一个类型转换运算符，这种运算符由一个类型名加()括号组成，后面的3是这个运算符的操作数。

## 2 运算符详解

### 2.1 逗号运算符

逗号运算符（Comma Operator）也是一种双目运算符，它的形式可以写出表达式1, 表达式2, 表达式3, ..., 表达式n这种形式，表达式1, 表达式2可以看作一个子表达式，先求表达式1的值，然后求表达式2的值作为这个子表达式的值，然后这个值再和表达式3组成一个更大的表达式，求表达式3的值作为这个更大的表达式的值，依此类推，整个计算过程就是从左到右依次求值，最后一个表达式的值成为整个表达式的值。
###2.2 sizeof运算符与typedef类型声明
sizeof是一个很特殊的运算符，它有两种形式：sizeof 表达式和sizeof(类型名)。它的特殊之处在于，sizeof 表达式中的表达式并不求值，只是根据类型转换规则求得该表达式的类型，然后把这种类型所占的字节数作为sizeof 表达式这整个表达式的值。另外一种形式sizeof(类型名)的括号则是必须写的，整个表达式的值也是这种类型所占的字节数。

    typedef unsigned long size_t;

typedef这个关键字用于给一个类型起个新的名字，上面的声明可以这么看：去掉typedef就成了一个变量声明unsigned long size_t;，size_t是一个变量名，类型是unsigned long，那么加上typedef之后，size_t就是一个类型名，就代表unsigned long类型。

## 3 计算机体系结构基础

### 3.1 内存管理单元

如果处理器启用了MMU，CPU执行单元发出的内存地址将被MMU截获，从CPU到MMU的地址称为虚拟地址（Virtual Address，以下简称VA），而MMU将这个地址翻译成另一个地址发到CPU芯片的外部地址引脚上，也就是将虚拟地址映射成物理地址，如下图所示。

![](images/LC04.png)

在启用MMU的情况下虚拟地址空间和物理地址空间是完全独立的，物理地址空间既可以小于也可以大于虚拟地址空间，例如有些32位的服务器可以配置大于4GB的物理内存。

MMU将虚拟地址映射到物理地址是以页（Page）为单位的，对于32位CPU通常一页为4KB。例如，MMU可以通过一个映射项将虚拟地址的一页0xb700 1000~0xb700 1fff映射到物理地址的0x2000~0x2fff，物理内存中的页称为物理页面或页帧（Page Frame）。

### 3.2 内存分级(Memory Hierarchy)

| 存储器类型 | 位于哪里                                                     | 存储容量                                                     | 半导体工艺                                                   | 访问时间                                               | 如何访问                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------ |
| CPU寄存器  | 位于CPU执行单元中。                                          | CPU寄存器通常只有几个到几十个，每个寄存器的容量取决于CPU的字长，所以一共只有几十到几百字节。 | 由一组触发器（Flip-flop）组成，每个触发器保存一个Bit的数据，可以做存取和移位等操作。计算机掉电时寄存器中保存的数据会丢失。 | 寄存器是访问速度最快的存储器，典型的访问时间是几纳秒。 | 使用哪个寄存器，如何使用寄存器，这些都是由指令决定的。       |
| Cache      | 和MMU一样位于CPU核中。                                       | Cache通常分为几级，最典型的是如上图所示的两级Cache，一级Cache更靠近CPU执行单元，二级Cache更靠近物理内存，通常一级Cache有几十到几百KB，二级Cache有几百KB到几MB。 | Cache和内存都是由RAM（Random Access Memory）组成的，可以根据地址随机访问，计算机掉电时RAM中保存的数据会丢失。不同的是，Cache通常由SRAM（Static RAM，静态RAM）组成，而内存通常由DRAM（Dynamic RAM，动态RAM）组成。DRAM电路比SRAM简单，存储容量可以做得更大，但DRAM的访问速度比SRAM慢。 | 典型的访问时间是几十纳秒。                             | Cache缓存最近访问过的内存数据，由于Cache的访问速度是内存的几十倍，所以有效地利用Cache可以大大提高计算机的整体性能。一级Cache是这样工作的：CPU执行单元要访问内存时首先发出VA，Cache利用VA查找相应的数据有没有被缓存，如果Cache中有就不需要访问物理内存了，是读操作就直接将Cache中的数据传给CPU寄存器，是写操作就直接在Cache中改写数据；如果Cache中没有，就去物理内存中取数据，但并不是要哪个字节就取哪个字节，而是把相邻的几十个字节都取上来缓存着，以备下次用到，这称为一个Cache Line，典型的Cache Line大小是32~256字节。如果计算机还配置了二级缓存，则在访问物理内存之前先用PA去二级缓存中查找。一级缓存是用VA寻址的，二级缓存是用PA寻址的，这是它们的区别。Cache所做的工作是由硬件自动完成的，而不是像寄存器一样由指令决定先做什么后做什么。 |
| 内存       | 位于CPU外的芯片，与CPU通过地址和数据总线相连。               | 典型的存储容量是几百MB到几GB。                               | 由DRAM组成，详见上面关于Cache的说明。                        | 典型的访问时间是几百纳秒。                             | 内存是通过地址来访问的，但是在启用MMU的情况下，程序指令中的地址是VA，而访问内存用的是PA，并无直接关系，这种情况下内存的分配和使用由操作系统通过修改MMU的映射项来协调。 |
| 硬盘       | 位于设备总线上，并不直接和CPU相连，CPU通过设备总线的控制器访问硬盘。 | 典型的存储容量是几百GB。                                     | 硬盘由磁性介质和磁头组成，访问硬盘时存在机械运动，磁头要移动，磁性介质要旋转，机械运动的速度很难提高到电子的速度，所以访问速度很受限制。但是保存在硬盘上的数据掉电后不会丢失。 | 典型的访问时间是几毫秒，是寄存器的10^6倍。             | 由驱动程序操作设备总线控制器去访问。由于硬盘的访问速度较慢，操作系统通常在一次从硬盘上读几个页面（典型值是4KB）到内存中缓存起来，如果这些数据后来都被程序访问到了，那么这一次硬盘访问的时间就可以分摊（Amortize）给多次数据访问了。 |

## 4、x86汇编程序基础

汇编程序中以`.`开头的名称并不是指令的助记符，不会被翻译成机器指令，而是给汇编器一些特殊的指示，称为汇编指示（Assembler Directive）或伪操作（Pseudo-operation），由于它不是真正的指令，所以加个“伪”字。

.section指示把代码划分成若干个段（Section），程序被操作系统加载执行时，每个段被加载到不同的地址，具有不同的读、写、执行权限。`.data`段保存程序的数据，是可读可写的，C程序的全局变量也属于`.data`段。本程序中没有定义数据，所以`.data`段是空的。`.text`段保存代码，是只读和可执行的，后面那些指令都属于这个`.text`段。



```sql
where id in ( #{ids} ) and ( name like concat(‘%’, #{name} ,’%’) or des like concat(‘%’, #{des} ,’%’) ) and status = #{status} 
```

