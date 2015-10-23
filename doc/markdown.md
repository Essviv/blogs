#Markdown

##段落和换行
+ 段落前后要有一个以上的空行
+ 换行可以通过在行尾添加两个以上的空格来实现
 
这是一个新的段落

这个地方不会
出现断行

但这个地方  
会出现断行
  
##标题
+ Setext: ===和---
+ atx: #号

##引用
引用可以通过“>”来实现

> 这是个引用的段落  
> 这还是个引用的段落

##列表
列表可以通过加号、星号和减号来实现

+ 加号选项1
+ 加号选项2
 * 星号选项1
 * 星号选项2
 * 星号选项3
+ 加号选项3
  - 减号选项1
  - 减号选项2
  - 减号选项3

##代码
代码可以通过反单引号(`)来实现

	public static void main(String[] args) throws IOException {
        final String markdownFilename = "/markdown.txt";
        final String htmlFilename = "C:\\Users\\Lenovo\\Desktop\\markdown.html";

        File file = new File(Markdown.class.getResource(markdownFilename).getFile());
        String html = new PegDownProcessor(Extensions.ALL).markdownToHtml(FileUtils.readFileToString(file));
        FileUtils.writeStringToFile(new File(htmlFilename), html);

        System.out.println("OK");
    }

##分隔线
分隔线可以通过在一行中超过三个以上的星号，减号或底线来完成

 下面会有一条横线， 是用星号画出来的
 ***

 下面也会有一条横线，是用底线画出来的
 ___

 下面还会有一条横线，是用减号画出来的
 ---

##链接
+ \[an example\]\(hyperlink\)  
  [百度](http://www.baidu.com/) [谷歌](http://www.google.com/)  

+ \[an example\]\[id\]  
  \[id\]: hyperlink  
  [百度][1] [谷歌][2]
  [1]: http://www.baidu.com/  
  [2]: http://www.google.com/

##图片
图片引用的格式： !\[alt text\]\(path\)  
![百度首页LOGO](https://ss0.bdstatic.com/5aV1bjqh_Q23odCf/static/superman/img/logo/bd_logo1_31bdc765.png)

##强调
强调可以通过在文字前后加星号或者下划线来完成

*某某很帅*  
_某某真的很帅_



