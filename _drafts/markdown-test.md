#markdown语法用法实例
[官方语法说明](https://daringfireball.net/projects/markdown/syntax) |
[中文语法说明](http://www.appinn.com/markdown/)
###引用
*yinyong*

###水平线
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

##html
<p>cn blog <a href="http://donald-draper.iteye.com/blog">Iteye</a>.</p>


<del>html删除标签<span>行内元素，这些都Html便签都可以在markdown文件中使用</span></del>



###转义
&lt; and &amp;， &copy;

###粗体字
**bold words**

4<5

4&5



不过需要注意的是，code 范围内，不论是行内还是区块， < 和 & 两个符号都一定会被转换成 HTML 实体，这项特性让你可以很容易地用 Markdown 写 HTML code （和 HTML 相对而言， HTML 语法中，你要把所有的 < 和 & 都转换为 HTML 实体，才能在 HTML 文件里面写出 HTML code。）




