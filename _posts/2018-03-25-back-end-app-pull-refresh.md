---
layout:     post
title:      实现类似新闻类app下拉刷新功能的思路（后端）
subtitle:   从后端角度分析并实现一个下拉刷新的功能需求。
date:       2018-03-25             
author:     jessehzx                
header-img: img/post-bg-2015.jpg
catalog: 	  true
tags:
    - Java
        
---

>版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！


### 开篇

首先明确一点，此篇文章不是教你怎么在Android/iOS客户端实现下拉刷新，而是从后端角度去实现类似今日头条这类新闻客户端的下拉换一批的效果。这里暂且不论数据统计、精准推荐那些，只谈我个人的实现思路。

### 需求

产品经理给到我这边的需求大概是这样：有一批短视频，数据量1w多条，共有10个分类，每个分类下平均1千条左右数据。每次下拉第一页的数据不重复（前20次），上拉加载在当前页随机打乱也不允许重复。前端不做逻辑处理，只负责调接口拿数据。后端做实现。

### 分析

那这里就有几个问题了： 

1. 每次前端下拉的时候，请求接口的参数pageNum都是1，按常理都返回第一页的数据，那怎么保证每次给的数据不一样呢？ 
2. 上拉加载，传的pageNum是2，3，4…，按正常的分页去数据库拿不就行了。但需求明确指出要打乱要随机，如果每次用Collections.shuffle(list);去打乱然后分页去取，那又可能出现和其他页重复的数据。

> 有没有一种方案既能保证下拉20次不重复，又能兼顾上拉时每一页数据不重复？

### 思路

想象一下：如果这下拉的20次，每次都给一个虚拟分页页号（1-20），这个页号在每次请求第一页的时候（即pageNum==1）都作累加并放入缓存，当大于20的时候重置为1，那这样就形成了一个闭环，每次下拉其实都是在第1页到第20页之间来回切换。（PS：放缓存的时候别忘了把用户id和短视频类型也作为该缓存key的一部分)

由于每个短视频bean的字段还不少，我们不能把对应type下1千条左右的数据都放到一个key的memcached里，需要将每个短视频都对应一个key放入缓存。然后每一个类型都去生成20个数据池，每个池子的编号就是用户下拉的虚拟分页页号，比如第1个池子，第一页就是下拉的第一页，第二页开始就要临时创建一个tempList，把第一页的数据addAll并在原list上做removeAll，然后写个循环，用new Random或Math.random取随机数作为下标，拿到对应的元素一个个add进tempList，同时remove掉原list里的这个元素，直到原list.size()为0了就跳出循环，把这个临时tempList放入缓存就是一个池子。其他虚拟分页的思路以此类推。

弄好20个池子的数据顺序后，根据前端请求的pageNum和用户对应的虚拟分页，就能判断到底是取哪个池子的数据了，只需要根据分页参数做截取。

### 实现

实现步骤如下： 

1. 遍历每个类型下的短视频，每个都放一份缓存，用下标作为key，便于后续使用下标取到缓存中的短视频； 
2. 将每个类型下的短视频放20份池子，每个池子里面除了第一页数据不一样，其余都是打乱的；

代码如下：

```
public List<Integer> getIdlist(List<Integer> idList, int index, int pageSize) {
        if (CollectionUtils.isEmpty(idList)) {
            return null;
        }
        List<Integer> idListTmp = idList;
        List<Integer> tmp = new ArrayList<Integer>();
        if (index * pageSize < idListTmp.size()) {
            for (int i = (index - 1) * pageSize; i <  index * pageSize; i++) {
                tmp.add(idListTmp.get(i));              
            }
            idListTmp.removeAll(tmp);               
        }

        boolean hasNext = true;
        Random random = new Random(System.currentTimeMillis());
        while (hasNext) {
            if (idListTmp.size() < 1) {
                hasNext = false;
                break;
            } else {
                int indexTmp = random.nextInt(idListTmp.size());
                int id = idListTmp.get(indexTmp);
                tmp.add(id);
                idListTmp.remove(indexTmp);
            }
        }

        return tmp;
    }
```

3.根据用户虚拟分页对上池子编号，拿到那个池子的缓存数据，再根据前端请求的分页参数返回数据即可。

```
    List<Video> videoList = new ArrayList<Video>();
	// 按分页返回
    PageUtils<Integer> pages = new PageUtils<Integer>(idList, request.getPageNum(), request.getPageSize());
    if (CollectionUtils.isNotEmpty(pages.getResult())) {
        for (int index : pages.getResult()) {
            Object obj = Cache.get(type + index);
            if (null != obj) {
                videoList.add((VideoBean) obj);
            }
        }
    }
```