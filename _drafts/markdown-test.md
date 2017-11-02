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
