# 爬虫实战 - 利用BeautifulSoup抓取豆瓣电影top250

已经有不少博主写过抓豆瓣top250的文章，作为一个想要入门爬虫的人而言（我），这真的是一个basic的练手方式。
我也是结合了几篇文章之后总结出来的一些要点。之前因为已经写过了re，这次是主要想要学学bs，但是发现还是技术不到家，有些文本信息还是需要re进行处理。
如果你对
1 beautifulsoup使用方法
2 如何用re提取中文和英文名字
3 数据输出到csv
感兴趣的话，那你可以选择看一下这篇文章-。-

## 老生常谈-爬虫必备的一些包

~~~python
from bs4 import BeautifulSoup
import requests
import urllib
import re # 正则
import csv
~~~

## 与网页交互

~~~
def get_html(offset):
    ua_headers={'User-Agent':'Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_6_8; en-us) AppleWebKit/534.50 (KHTML, like Gecko) Version/5.1 Safari/534.50'} # 请求头，这样网站不会把你当成虫虫
    url='https://movie.douban.com/top250?start='+str(offset)+'&filter=' #网页格式
    request = urllib.request.Request(url,headers=ua_headers) # 请求访问
    content = requests.get(url, headers = ua_headers).text # 读取网页内容
    return content
~~~
这里要说的就是豆瓣250的网页都是一个格式
比如
https://movie.douban.com/top250?start=25&filter=
https://movie.douban.com/top250?start=50&filter=
所以我们把不同的部分取出来用一个参数传进去，并且用str()转换成string
到这里就是把网页当中所有的东西爬了下来

## 解析网址

这也是最关键的，我大概看了两晚上，才大概知道怎么找到我想要的信息
先说一下我自己是想要把电影的名字，评分，爬下来然后和imdb做个对比，所以我需要外文名，这就是我捣鼓了最久的，如何提取英文名，中文名等。效果也不是特别好，请见谅。

###  beautifulsoup的使用方法

我个人认为它就像一个树形结构一样，你需要的东西就在一个node上，你要做的事情就是找到通往它的一条路径。
首先我们先看一下，爬下来的东西
![image.png](https://upload-images.jianshu.io/upload_images/19398691-2791495368f3168d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 网页结构怎么看
比如<div> ......... </div>中间代表一级，就像是左括号和右括号一样。我要看的电影信息就在<div class='item'>里面；
再举个例子，我要找到电影的title，路径是这样的：
1 <div class='item'> 
2 <div class="hd">
3  <a href="https://movie.douban.com/subject/1292052/" class="">
4 <span class="title">

**敲重点了，如果你要读的是第二个span里的值怎么办？**
读到第一个span然后 `.next_sibling`

### 规范信息
**1 replace**
`a.span.next_sibling.next_sibling.get_text().strip().replace('/\xa0',"")`

这么长一串其实我主要是想说replace可以把趴下来的数据当中那些占位符，空格或者奇奇怪怪的东西去掉，简单说就是把他们替换成“ ”

**2 如何处理外国人的中文译名（当中有·）和英文名（当中有空格）**

原始信息如下：

```
导演: 弗兰克·德拉邦特 Frank Darabont   主演: 蒂姆·罗宾斯 Tim Robbins /...1994 / 美国 / 犯罪 剧情
```

`re.findall(r'.*?(\b[a-z A-Z]+)\s+',info)`

`re.findall`是一个全匹配的函数，有一个找一个

`\b[a-z A-Z]+`只要英文，+表示一个或多个

`\s+`表示允许空格

`r'.*?([\u4e00-\u9fa5·+]+)\s+'` 

`\u4e00-\u9fa5`匹配中文

`\u4e00-\u9fa5·+`匹配中文和·

最后效果

![image-20190901154538509](/Users/liyiting/Library/Application Support/typora-user-images/image-20190901154538509.png)




### 具体代码如下
```python
def parse_page(offset,movie_list):
    content=get_html(offset)
    soup=BeautifulSoup(content,'lxml')
    items=soup.find_all('div',{'class':'item'}) #想像成一棵树，往下找叶子节点
    for item in items:
        a=item.find('div',{'class':'hd'}).a
        s=item.find('div',{'class':'star'})
        bd=item.find('div',{'class':'bd'})
        title=a.span.get_text() #get_text 得到string
        eng_title=a.span.next_sibling.next_sibling.get_text().strip().replace('/\xa0',"")
        # eng_title=re.findall(r'.*?(\b[a-z A-Z])\s+',eng_title)
        rating=s.find('span',{'class':'rating_num'}).get_text()
        info=bd.p.get_text().strip() #strip 去掉空格
        cast=re.findall(r'.*?(\b[a-z A-Z]+)\s+',info)
        other=re.findall(r'.*?([\u4e00-\u9fa5·+]+)\s+',info)
        movie={'title':title,'eng_title':eng_title,'rating':rating,'cast':cast,'other':other}
        movie_list.append(movie)
    return movie_list
```
输出效果差不多这样，个人还算满意：
~~~python
{'title': '指环王3：王者无敌',
  'eng_title': 'The Lord of the Rings: The Return of the King',
  'rating': '9.2',
  'cast': [' Peter Jackson', ' Viggo Mortensen'],
  'other': ['彼得·杰克逊', '维果·莫腾森', '美国', '新西兰', '剧情', '动作', '奇幻']},
{'title': '天空之城',
  'eng_title': '天空の城ラピュタ',
  'rating': '9.1',
  'cast': [' Hayao Miyazaki', ' Mayumi Tanaka'],
  'other': ['宫崎骏', '田中真弓', '横泽启子', '日本', '动画', '奇幻']}
~~~

## 输出到csv

1 直接用**pandas**

~~~python
import pandas as pd
test=pd.DataFrame(data=df)
test.to_csv('douban_top250.csv',encoding='utf-8')
~~~

2 利用csv，可以逐行写入

~~~python
with open('douban_top250.csv','a',encoding='utf-8',newline='')as f:
csvFile=csv.writer(f)
csvFile.writerows(movie_list)
~~~



## 参考

https://blog.csdn.net/weixin_36605200/article/details/82291845

https://blog.csdn.net/qq_38268886/article/details/80744721

https://blog.csdn.net/xing851483876/article/details/80578998

