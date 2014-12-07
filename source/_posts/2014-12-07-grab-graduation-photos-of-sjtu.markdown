---
layout: post
title: "交大毕业生证件照批量抓取"
date: 2014-12-07 18:24:23 +0800
comments: true
categories: python
---
交大的毕业照都是委托一家叫[潮源数码](http://spring-pic.com/)的小公司拍摄的，下载自己的照片也在这个网站。简单看了一下这个简陋的网站的下载照片处，虽然有填写身份证号的地方，但观察下传参就能发现仅对姓名和学号是否一致进行了判断。这样只需要有姓名和学号的名单，就可以写脚本抓取所有毕业生的照片。

首先想到的是交大研究生院网站，果然一找就能找到，比如 [11级](http://www.gs.sjtu.edu.cn/inform/attachment/2011/20110902_002310_077.xls) [12级](http://www.gs.sjtu.edu.cn/inform/attachment/2012/20120702_145958_344.xls) 下载后整理成学号+姓名的纯文本即可。

python脚本如下，注意编码问题，GBK与UTF-8的互相转换。

``` python
#coding=gbk
import os
import urllib
import urllib2
import re
from threading import Thread  
from Queue import Queue  

#q is the task queue 
#NUM is the sum of threads  
q = Queue()  
NUM = 10
data_file_name = 'list.txt'
login_url = 'http://spring-pic.com/info/Management.asp'
pic_pattern = 'a href="../Images/UpPhoto/(.*?)"'
pic_base_url = 'http://spring-pic.com/Images/UpPhoto/'
str_pattern = '学校名称：</td>.*?font_1">(.*?)<.*?：.*?font_1">(.*?)<.*?：.*?font_1">(.*?)<.*?：.*?font_1">(.*?)<'
save_dir = r'/Users/pierce/Downloads/temp/'

def save_image(dir,image_name,image):
    if not os.path.isdir(dir):
        os.makedirs(dir)
    try:
        image_file = open(dir + image_name,'wb')
    except IOError as (error, strerror):
        print "I/O error({0}):{1}".format(error,strerror)
    else:
        image_file.write(image)
        image_file.close()

def get_image(name,student_no,pattern):
    #Configure opener to handle cookies
    opener = urllib2.build_opener(urllib2.HTTPCookieProcessor())
    urllib2.install_opener(opener)
    #Build Parameters
    params = urllib.urlencode({'daimaxuehao':student_no,'realname':name,'sfzh':'','SchoolID':10248,'action':'sub','imageField.x':16,'imageField.y':16})
    #Open login html
    f = opener.open(login_url,params)
    login_html = f.read()
    f.close()
    #Search the image link
    m = re.search(pattern,login_html)
    #The student hasn't taken picture
    if m is None:
        return None
    else:
        n = re.search(str_pattern, login_html, re.DOTALL)
        if n is not None:
            school_name = n.group(1)
            department = n.group(2)
            major = n.group(3)
            class_no = n.group(4)
            #Get the image    
            match_part = m.group(1)
            f = opener.open(pic_base_url + match_part)
            image = f.read()
            return (school_name, department, major, class_no, image)

def add_tasks(queue, data_file_name):
    data_file = open(data_file_name, 'r')
    lines = data_file.readlines()
    for line in lines:
        items = line.split()
        q.put(items)

def grab(arguments):
    student_no = arguments[0]
    name = arguments[1]
    items = get_image(name,student_no,pic_pattern)
    name = name.decode("gbk").encode("utf-8")
    if items is not None:
        (school_name, department, major, class_no, image) = items
        school_name = school_name.decode("gbk").encode("utf-8")
        department = department.decode("gbk").encode("utf-8")
        major = major.decode("gbk").encode("utf-8")
        save_image(save_dir,school_name+'_'+department+'_'+class_no+'_'+student_no + '_' + name +'_'+major+ '.jpg',image)
        print student_no + '_' + name + '.jpg' + ' is saved'
    else:
        print student_no + '\t' + name + ' no photo'


add_tasks(q, data_file_name)

#working thread,deal with the data from queue 
def working():
    while True:
        arguments = q.get()
        grab(arguments)
        q.task_done()  
#fork NUM threads  
for i in range(NUM):
    t = Thread(target=working)
    t.setDaemon(True)
    t.start()
```

![Alt text](/images/custom/20141207.jpg)
