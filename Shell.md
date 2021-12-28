# 一、cd命令

​	cd ../目录
​	pwd 显示当前所在路径

# 二、ls命令

​	ls -l 显示当前目录下文件的附加信息
​	ls -a 将当前目录下隐藏文件和普通文件全部显示
​	ls -F 将当前目录下的目录和文件区别显示，目录后带有‘/’
​	ls -R 列出当前目录下所有的文件，包含子目录中的文件
​	ls -FR等价于ls -F -R
​	ls -l filename 显示出指定文件名或目录的附加信息
​	# filename中可以带有通配符*(0或多个字符)或?(单个字符)
​	# filename中可以用元通配符，例如[a-i]表示a到i的都显示出来
​	# [!a]表示带有a的不显示

# 三、touch命令

​	touch filename 创建以filename为文件名的空白文件
​	# 若filename文件已经存在，则改变该文件的修改时间
​	touch -a filename 改变该文件的访问时间
​	# ls -l --time=atime filename 查看该文件的最近的访问时间

# 四、cp命令

​	cp -i source destination 当source和destination都是文件名时，则将source复制成以destination为名的文件
​	# destination为路径时，则复制source到该路径下
​	cp -R Scripts/ Mod_Scripts 将Scripts/下的文件全部复制到Mod_Scripts目录下
​	# cp命令中同样可以使用通配符

# 五、ln命令

​	ln -s data_file sl_data_file 表示sl_data_file指向data_file

# 六、mv命令

​	mv fall fzll 更改fall文件名为fzll，inode编号和时间戳保持不变
​	mv fzll Pictures/ 将fzll移动到Pictures/目录下

# 七、rm命令

​	rm -i f?ll
​	rm -rf filename 没有任何提示，直接删除，慎用

# 八、mkdir命令

​	mkdir 目录名
​	mkdir -p 目录名  表示创建多个目录和子目录

# 九、rmdir命令

​	rmdir 目录名 只能删除空目录
​	# rm -r 目录名  使用-r选项可以向下进入目录，删除其中
​					的文件，然后再删除目录本身

# 十、file命令

​	file filename 查看文件类型

# 十一、cat命令

​	cat filename  显示文本文件中的内容
​	cat -n filename  -n参数会给所有的行加上行号
​	cat -b filename  -b参数只给有文本的行加上行号
​	cat -T filename  -T隐藏制表符

# 十二、more命令

​	more filename  在显示每页数据之后停下来

# 十三、less命令（可以使用上下页键）

# 十四、tail命令

​	tail filename 默认浏览文件最后10行
​	tail -n number filename  浏览文件最后number行
​	等价于tail -n filename 

# 十五、head命令

​	head filename 默认浏览文件前10行