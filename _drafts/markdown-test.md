# markdown语法用法实例
[官方语法说明](https://daringfireball.net/projects/markdown/syntax) |
[中文语法说明](http://www.appinn.com/markdown/)  
markdown超链接语法如上。
### 强调
*yinyong*

### 水平线
***

markdown兼容Html标签，不过标签与markdown文本前后加上空行与其它内容区隔开，还要求它们的开始标签与结尾标签不能用制表符或空格来缩进。

<p> the the html maker tag </P>
<table>
	<thead>
		<th>userId</th>
		<th>name</th>
		<th>email</th>
	</thead>
	<tbody>
		<tr>
			<td>00&amp;00</td>
			<td>donald&lt;han</td>
			<td>shaoqinghan@aliyun.com</td>
		</tr>
	</tbody>
</table>

请注意，在 HTML 区块标签间的 Markdown 格式语法将不会被处理。比如，你在 HTML 区块内使用 Markdown 样式的*强调*会没有效果。

## html
<p>cn blog <a href="http://donald-draper.iteye.com/blog">Iteye</a>.</p>


<del>html删除标签<span>行内元素，这些都Html便签都可以在markdown文件中使用</span></del>  

<cite>这是什么，斜体强调</cite>



### 转义
&lt; and &amp;， &copy;

### 粗体字
**bold words**

4<5

4&5



不过需要注意的是，code 范围内，不论是行内还是区块， < 和 & 两个符号都一定会被转换成 HTML 实体，这项特性让你可以很容易地用 Markdown 写 HTML code （和 HTML 相对而言， HTML 语法中，你要把所有的 < 和 & 都转换为 HTML 实体，才能在 HTML 文件里面写出 HTML code。）

## 区块元素

### 段落、换行
markdown段落分行可以使用html标签</br>,来分行，也可以使用在每行结束时，**空两格，再回车**，可以实现换行。  
比如：  
one line；</br> two line; three line;  
free line;  

### 标题

Markdown 支持两种标题的语法，类 Setext 和类 atx 形式。  

***  

类 Setext 形式是用底线的形式，利用 =（最高阶标题）和 - （第二阶标题），例如：  

春天的故事  
=

第一章  
-


任何数量的 = 和 - 都可以有效果。

#### atx形式

# 春天的故事后续
## 第一章
### 第一节
#### 节续
那是一个万物复苏的季节...

你可以选择性地「闭合」类 atx 样式的标题，这纯粹只是美观用的，若是觉得这样看起来比较舒适，你就可以在行尾加上 #，而行尾的 # 数量也不用和开头一样（行首的井字符数量决定标题的阶数）：

# 秋天的故事#
## 第一章##
### 第一节########
#### 节续
那是一个红叶镶漫天的季节...  

**注意标题符和标题中间，最好添加一个字符**


## 区块引用 Blockquotes

Markdown 标记区块引用是使用类似 email 中用 > 的引用方式。如果你还熟悉在 email 信件中的引言部分，你就知道怎么在 Markdown 文件中建立一个区块引用，那会看起来像是你自己先断好行，然后在每行的最前面加上 > ：
>message from donald  
>date 2017-11-02 15:03

>reply to me.

hello jamel, how are you. as longer,wo paly together, miss you eveyday.

Markdown 也允许你偷懒只在整个段落的第一行最前面加上 > ：

>message from jamel
han.
>>date 2017-11-02 18:00
my dear !  
>
>doanld, i 'm fine. what about you?

引用的区块内也可以使用其他的 Markdown 语法，包括标题、列表、代码区块等：
>### email box

何像样的文本编辑器都能轻松地建立 email 型的引用。例如在 BBEdit 中，你可以选取文字后然后从选单中选择增加引用阶层

## 列表

Markdown 支持有序列表和无序列表。无序列表使用星号、加号或是减号作为列表标记：


* red  
* green  
* blue  

等同于：

+ red  
+ green  
+ blue

也等同于：

- red  
- green  
- blue

有序列表则使用数字接着一个英文句点：  
1. red  
2. green  
3. blue  

很重要的一点是，你在列表标记上使用的数字并不会影响输出的 HTML 结果，上面的列表所产生的 HTML 标记为：
<ol>
	<li>red</li>
	<li>greeb</li>
	<li>blue</li>
</ol>

如果你的列表标记写成：  
1. red  
2. green  
3. blue  

或甚至是：  
1. red  
3. green  
1. blue  


你都会得到完全相同的 HTML 输出。重点在于，你可以让 Markdown 文件的列表数字和输出的结果相同，或是你懒一点，你可以完全不用在意数字的正确性。

如果你使用懒惰的写法，建议第一个项目最好还是从 1. 开始，因为 Markdown 未来可能会支持有序列表的 start 属性。

列表项目标记通常是放在最左边，但是其实也可以缩进，最多 3 个空格，项目标记后面则一定要接着至少一个空格或制表符。

要让列表看起来更漂亮，你可以把内容用固定的缩进整理好：

* The spring is coming, where are you travel ?   
  chinese scenary is beatiful.  
* when summer, you can swimming to natrual rivel.
  but, you must go with your guys.  

但是如果你懒，那也行：  

* The spring is coming, where are you travel ?   
chinese scenary is beatiful.  
* when summer, you can swimming to natrual rivel.
but, you must go with your guys.  

如果列表项目间用空行分开，在输出 HTML 时 Markdown 就会将项目内容用 <p> 标签包起来，举例来说：

* bird
* pig


会被转换为：

<ul>
	<li>Bird</li>
	<li>Magic</li>
</ul>

但是这个：  

* bird

* pig

会被转换为：

<ul>
	<li><p>Bird</p></li>
	<li><p>Magic</p></li>
</ul>

列表项目可以包含多个段落，每个项目下的段落都必须缩进 4 个空格或是 1 个制表符：  

1.  啊，秋天到了，一群大雁往南飞；  
    一会儿排成个人字，一会儿排成的一字。  

    等到明年春天，燕子是否还会在回来，啊，大雁。  
2.  乌云满天电风扇；  
    雨呼呼，雨哗哗。  

    吓得小鸡，唧唧唧；  
    吓得小鸭，嘎嘎嘎。  

如果你每行都有缩进，看起来会看好很多，当然，再次地，如果你很懒惰，Markdown 也允许：  
*   This is a list item with two paragraphs.

    This is the second paragraph in the list item. You're
only required to indent the first line. Lorem ipsum dolor
sit amet, consectetuer adipiscing elit.

*   Another item in the same list.

如果要在列表项目内放进引用，那 > 就需要缩进：  
* 测试在列表中使用引用
> the first line  
  > second line  



如果要放代码区块的话，该区块就需要缩进两次，也就是 8 个空格或是 2 个制表符：  

当然，项目列表很可能会不小心产生，像是下面这样的写法：  
1889. 打到小日本  

换句话说，也就是在行首出现数字-句点-空白，要避免这样的状况，你可以在句点前面加上反斜杠。  

1889\.小日本已经灭亡  


## 代码区块

和程序相关的写作或是标签语言原始码通常会有已经排版好的代码区块，通常这些区块我们并不希望它以一般段落文件的方式去排版，而是照原来的样子显示，Markdown 会用 <pre> 和 <code> 标签来把代码区块包起来。

要在 Markdown 中建立代码区块很简单，只要简单地缩进 4个空格或是 1个制表符就可以，例如，下面的输入：

计算和函数  

    funtion(int org){
        int  sum = 0;  
        for(int i = 0 ; i< org ;i ++){
        sum += i;
        }
        return sum;
    }

Markdown 会转换成：     

<pre>
  <code>
  funtion(int org){
      int  sum = 0;  
      for(int i = 0 ; i< org ;i ++){
        sum += i;
      }
      return sum;
  }
  </code>
</pre>  

**注意一般情况下制表符为4个空格，但是有些软件的制表符默认为两个比如github atom，自己配置直接符的空格数。**  


    jack
        main
    rain
    markdown

一个代码区块会一直持续到没有缩进的那一行（或是文件结尾）。
在代码区块里面， & 、 < 和 > 会自动转成 HTML 实体，这样的方式让你非常容易使用 Markdown 插入范例用的 HTML 原始码，只需要复制贴上，再加上缩进就可以了，剩下的 Markdown 都会帮你处理，例如：  

<div class="footer">
    &copy; 2004 Foo Corporation
</div>

会被转换为：

<pre><code>&lt;div class="footer"&gt;
    &amp;copy; 2004 Foo Corporation
&lt;/div&gt;
</code></pre>

代码区块中，一般的 Markdown 语法不会被转换，像是星号便只是星号，这表示你可以很容易地以 Markdown 语法撰写 Markdown 语法相关的文件。



## 分隔线

你可以在一行中用三个以上的星号、减号、底线来建立一个分隔线，行内不能有其他东西。你也可以在星号或是减号中间插入空格。下面每种写法都可以建立分隔线：

***   

---   
___

*****  

---------

## 区段元素
### 链接

Markdown 支持两种形式的链接语法： 行内式和参考式两种形式。

不管是哪一种，链接文字都是用 [方括号] 来标记。

要建立一个行内式的链接，只要在方块括号后面紧接着圆括号并插入网址链接即可，如果你还想要加上链接的 title 文字，只要在网址后面，用双引号把 title 文字包起来即可，例如：


[creative book](http://www.jianshu.com/users/79667750f82e/timeline) doandhan  
cn blog [iteye](http://donald-draper.iteye.com/blog "Donald_Draper") .

**注意链接和连接提示中间要加空格**

如果你是要链接到同样主机的资源，你可以使用相对路径：

See my [About](/about/) page for details.   

参考式的链接是在链接文字的括号后面再接上另一个方括号，而在第二个方括号里面要填入用以辨识链接的标记：

This is [cn-blog][blog-iteye] reference-style link.

你也可以选择性地在两个方括号中间加上一个空格：

This is [an example] [id] reference-style link.

接着，在文件的任意处，你可以把这个标记的链接内容定义出来：

[blog-iteye]: http://donald-draper.iteye.com/blog  "Donald_Draper"
链接内容定义的形式为：

* 方括号（前面可以选择性地加上至多三个空格来缩进），里面输入链接文字
*  接着一个冒号
* 接着一个以上的空格或制表符
* 接着链接的网址
* 接着一个以上的空格或制表符
* 选择性地接着 title 内容，可以用单引号、双引号或是括弧包着

下面这三种链接的定义都是相同：

[blog-iteye]: http://donald-draper.iteye.com/blog  "Donald_Draper"  

[blog-iteye]: http://donald-draper.iteye.com/blog  'Donald_Draper'  

[blog-iteye]: http://donald-draper.iteye.com/blog  (Donald_Draper)


请注意：有一个已知的问题是 Markdown.pl 1.0.1 会忽略单引号包起来的链接 title。  

This is [cn-blog][blog-iteye] reference-style link.
