#### 1. 获取git上测试源码  

#### 2. go 嵌入代替继承的概念  
go使用type，type不与class绑定  
java或者python与class绑定  

#### 3. go安装与设置
src 源码文件目录  
pkg 标准库归档文件  
设置环境变量 GOROOT与PATH  
工作区使用的GOPATH目录  

#### 4. 工程结构
目录与package相绑定，直接src目录下则为main的package  
pkg目录 代码包归档文件 .a结尾 属于go install 后的不可执行文件  
bin目录 可执行文件 属于go install后的可执行文件  
命令源码文件：属于main代码包且包含无参数声明和结果声明的main函数的源码文件  

#### 5. 源码文件
命令源码文件：main代码包，在go build下生成同目录可执行文件；go install下生成bin下可执行文件  
库源码文件：不包含main，包名与目录名一致。库文件的路径与src中的相对路径对标。  
测试源码文件：  
* 文件名以_test.go结尾
* 文件中需要至少包含一个名称为Test开头或Benchmark开头，且拥有一个类型为*testing.T或者*testing.B的参数函数。testing.T用于功能测试的type，testing.B用于基准测试的type
go test处理  

#### 6. 字符集
go也是UTF-8的编码  

#### 7. 代码包规则
包声明：package关键字，包名为代码包路径的最后一个元素。（不同路径，同名的情况?同名在导入中会引起错误，需要使用别名）  
包导入：import关键字，src目录下相对路径；同一个源码文件中导入的多个代码包最后一个元素不可以重复。（公开的程序实体导入使用？）  
* "."符号的使用  
* 标识符的使用：不能以数字或者下划线开头  
包初始化：init()函数，init()函数在main()函数执行前执行。全局变量在初始化函数执行前会初始化完成。初始化函数中若涉及全局变量，则可直接使用。（重复命名则会失败？）  
* 被导入的代码包初始化函数先执行  
	
#### 8. 标识符（变量或者对象）的公开与否
首字符大写则可公开；首字符小写则本包内访问  
完全公开访问需再加两个条件：标识符不是被定义在函数或者结构体内部；代码包所属目录在GOPATH中。  

#### 9. 只初始化代码包
只初始化代码包不需要当前源码文件中此代码包的程序实体，则可以用"_"代替。  

#### 10. go的相关命令
build/clean/doc/env/fix/fmt/generate/get/install/list/run/test/tool/vet/version  
go tool pprof/trace  
