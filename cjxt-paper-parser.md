# 试卷解析详解

* v0.1 by summer - 2013-05-01

## 解析的作用以及结果

试卷解析的作用，就是通过上传的一个试卷的word文档，解析出试卷的所有大题，小题，答案等保存到数据库中，比如以下的word试卷：

```html
一、选择题
1. 选择题描述1
A.1 B.2
【详解】选择题详解

2. 选择题2

二、填空题
1. 填空题描述1

2. 填空题描述2
```
  
解析的结果为：

* 一、选择题
   * 1选择题1
      * 选项
	     * A. 1
		 * B. 2
	  * 详解：选择题详解
   * 2选择题2
* 二、填空题
   * 1填空题1
   * 2填空题2
   
## 解析的总体思路

由于Word文件不好直接处理，但直接转换成纯文本的话又缺少对格式的支持，于是就采用了一个折中的方案，即先将Word文件转换成HTML网页(带格式和图片的)，然后通过解析这张网页将题目解析出来。将Word转换成HTML再解析的原因主要是是HTML很容易处理，且可以直接显示在网页上。

## 源代码的阅读

跟解析相关的代码主要在包cn.edu.zucc.cjxt.paper.paser下，主要有Word2HtmlUtil、QuestionParser这两个类。其中QuestionParser是最核心的部分，下面主要讲解QuestionParser的组成。

类QuestionParser对外的接口是`parseWordFile`方法，相当于C语言的main方法，所以阅读源代码可以从这个方法入手。

然后就可以以这个方法为主线阅读代码了，在eclipse中可以使用`CTRL + 鼠标点击`的方式直接定位到相应的方法的代码。这其中最重要的步骤是`解析文档`，也就是`parse(doc)`语句。

## Word文件转成HTML

直接使用网上现成的代码(使用POI库)，主要是`Word2HtmlUtil.toHtml`方法。

## 解析的主要过程
   
解析的主要过程是对一个word从上往下扫描，解析出这个结构。扫描过程中的四个重要的变量：

```java
/** 当前正在处理的Object，类型为大题、题目和属性(String) */
protected Object currentObject = null;
/** 当前的大题 */
protected PaperBigQuestion currentBigQuestion = null;
/** 当前的题目 */
protected PaperQuestion currentQuestion = null;
/** 当前属性的名称 */
protected String currentPropertyName = null;
```

例如，上面的word文档的解析的过程为：

1. 首先，所有currentXXX都设置为null，表示当前还没有扫描到任何的题目；
2. 判断第1段`一、选择题`是大题的开始，然后设置`currentBigQuestion = '一、选择题'`，设置`currentObject = '一、选择题'`；
3. 判断第2段，识别出`1. 选择题描述1`是题目的开始，那么设置`currentQuestion = '1选择题1'`，同样设置`currentObject`，根据当前的`currentBigQuestion`，我们能够轻易知道当前所在的大题为`一、选择题`；
4. 判断第3段，发现`A.1 B.2`既不是大题的开始(大题的开始形式为`一、`)，也不是题目的开始，于是根据`currentObject == '1选择题1'`知道该段为选择题1的内容(具体ABCD选项最后再解析)；
5. 判断第4段`【详解】选择题详解`为属性，根据当前的`currentQuestion == '1选择题1'`，知道这是选择题1的详解；
6. 判断第5段，为空，忽略；
7. 判断第6段，识别出`2. 选择题2`是题目的开始，跟步骤3一样处理；
8. 如此循环，知道扫描完整个word文档；

可以总结，currentXXX用来表示当前的节点，如currentBigQuestion表示当前扫描到的行所在的大题，这样，扫描到的题目就可以通过currentBigQuestion知道这道题目所在的大题。

## 用Jerry来解析HTML

之前的解析的过程都是基于从上到下扫描所有段落的，那么，如何扫描呢？

Jerry is a jQuery in Java. 相信用过jQuery的都知道用jQuery来遍历DOM树是多么的简单，Jerry的用法跟jQuery基本一致，比如这里用到用Jerry来遍历HTML中的每一个段落，同时提取出每一个段落的文本(QuestionPaser.java:331)：

```java
doc.$("body > *").each(new JerryFunction() {
	@Override
	public boolean onNode(Jerry self, int idx) {
		String text = self.text();
		// ...处理代码
		return true;
	}
});
```

其中doc表示的是整个HTML文档，这里用选择符`body > *`来选择`<body>`标签下的所有的儿子标签，在这里就是转换之前的Word的每一个段落。

## 段落的类别的判断

判断的段落类别的，即判断一个段落如`一、选择题`是表示大题的开始、题目的开始、属性的开始还是普通的文本。具体实现对应于源码中的`parseBigQuestionStart`，`parseQuestionStart`，`parsePropertyStart`函数。

### 大题开始的判断

大题一般以一个中文的数字和顿号开始，那么就可以用这个特征，用一个正则表达式来判断：

	String regex = "\\G\\s*[一|二|三|四|五|六|七|八|九|十]+[\\.|．|、]\\s*(?<name>\\S*)"; // 匹配大题开始的的正则表达式
	
### 题目开始的判断

题目一般以一个阿拉伯数字和点号开始，之后可以接上用【】符号括起来的题目类型，那么久可以用这个特征，用一个正则表达式来判断：

	String regex = "\\G\\d+[\\.|．|、](\\s*【(?<type>.*)】)?"; // 匹配题目开始的的正则表达式
	
### 属性开始的判断

属性一般以一个用【】括号括起来的属性名开始，那么可以用正则表达式来判断：

	String regex = "\\G【(?<name>.*)】"; // 匹配题目开始的的正则表达式
	



