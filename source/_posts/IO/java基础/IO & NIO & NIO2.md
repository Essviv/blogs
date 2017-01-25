# IO & NIO & NIO2

## IO
* Inputstream/OutputStream: 流对象，用于字节读写
* Reader/Writer: 用于字符读写
* DataInput/DataOutput: 用于二进制读写
* ObjectInput/ObjectOutput: 用于对象读写
* Scanner/Formatter: 用于读取格式化的字符

## NIO
* Path: 从**语法**上对系统中的文件和文件夹的抽象
* Files: 封装对文件的操作，比如对文件的增删改查，拷贝，重命名，读取文件属性等操作
* Paths: 获取path对象的工具类
* FileAttribute: 文件属性
* WatchService: 监听文件/文件夹增删改查
* FileSystem/FileStore: 代表文件系统和磁盘

## NIO2
* ByteBuffer: 字节缓冲区
* Channel: 通道，用于从缓冲区读取数据或者向缓冲区写入数据
* Selector: 多路复用选择器

![IO对比](http://p.blog.csdn.net/images/p_blog_csdn_net/haoel/15190/o_java.nio.00.jpg)

## 练习代码
1. [git@github.com:Essviv/nio.git](git@github.com:Essviv/nio.git)