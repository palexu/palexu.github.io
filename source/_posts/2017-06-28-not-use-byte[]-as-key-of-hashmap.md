---
title: 别拿byte[]数组作为key
date: 2017-06-28 12:00
---
byte[]数组作为key值，只是数组的地址的引用的hashcode，不能够根据byte[]数组的内容来，创建相应的hashcode，也就是所谓的索引key。所以，如果想用byte[]数组来作为map的key值的话，有三种方法：

1. 将byte[]，先转化为string
2. 将采用list<byte>
3. 将byte[]自己包装，使用byte[]数组的内容来重写hashcode和equals方法
