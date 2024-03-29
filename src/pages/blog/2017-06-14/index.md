---
title: 乐驾 - 语义理解算法
date: 2017-06-14 12:00:00
categories: iOS
background: https://i.imgur.com/4mAnp0g.jpg
tags:
    - iOS
    - Algorithm
    - Swift
---

## 前言

介绍乐驾在做语义理解与语音控制的过程和思路. 算法是在讯飞提供分词的基础上来实现的.

<!--more-->

![Imgur](https://i.imgur.com/Y1bu1kH.jpg)

关键词:

* 基于 API, 有的放矢, 效率高
* 无序理解, 倒序寻找关键词
* 副词互斥, 避免只找关键词的二义性
* 元组多维结果, 增加理解深度, 理解上下文

## 算法梗概

乐驾的愿景是让用户在脱离双手的情况下控制整个应用, 包括地图导航, 油价查询, 音乐电台, 汽车状态播报, 违章查询等功能. 所以, 在乐驾中, 对语义理解的要求是单方面精准理解, 而不求大而泛. 因为整个应用就做了这么多功能, 不管我们怎么理解其他语句, 我们本身并不能提供相关的 API 来支持. 

### 基于 API, 有的放矢, 效率高

前期, 我尝试过两种思路来理解一句话. 第一种是从理解句子的角度来想, 太麻烦了, 句子千变万化.

第二种是从我能提供的 API 有哪些的角度来出发, 为什么这样考虑, **如果不是基于一种操作来写语义理解的过程, 就算你理解了这句话, 然而你并无相关的操作来满足这句话的要求.**

所以, 从 API 的角度来说, 能极大的提高算法的效率.

举个例子:

1. 播放英文歌曲
2. 我想播放英文歌曲
3. 我要播放英文歌曲
4. 乐驾, 给我播放英文歌曲
5. 今天心情比较开心, 给我播放英文歌曲

如果按**第一种思路**来的话:

对于第二和第三种情况, 我想 / 我要 这种代表多级操作的词, 没有任何意义. 所以需要去掉这种无用的词, 然后再判断交给哪个二级操作或者一级操作.

对于第四和第五种情况, 有些某些无关的状语从句, 还是要在分词的基础上判断有无关键词.

如果按**第二种思路**来的话: (在分词的基础上)

先找 `播放` 再找 `英文` 或者 `中文` 关键字, 就可以完成操作了. 

### 无序理解, 倒序寻找关键词

> “研表究明，汉字的序顺并不定一能影阅响读，比如当你看完这句话后，才发这现里的字全是乱的。”
> 人眼在阅读, 大脑在处理信息的时候, 是不分顺序的, 所以无顺序的信息不影响理解.

关键字更是一种具有操作意义的词.

不是完全倒序, 是按照重要程度的倒序, 由里到外的顺序, 操作API的顺序.

为什么要简化呢, 举个例子, 因为我不管多么的理解 `播放英文男歌手的歌曲` 这句话, 我没有相关的操作啊, 我就算知道了用户干嘛, 底层没有给我相关的API, 我也束手无策.

中文的习惯是把重要的放在后面

**倒序, 倒着才符合中文逻辑**

### 数据格式

数据格式的选取, 考虑到横向和纵向都有无限的层级, 我没有选取树, 而是选用了 JSON / XML 这种可以无限嵌套的数据格式.

目前是**人为训练模型**

### 副词互斥, 避免只找关键词的二义性

只找关键词是远远不够的, 还要找一些副词. 一来判断用户的情感, 当然了, 只有 "是" 与 "不是" 两个等级. 二来进一步判断用户的意图, 例如用户关于某个问题的意图是 "是什么" 还是 "为什么".

### 增加理解深度, 理解上下文

关于理解深度的问题, 有些问题具有上下文语境, 例如:

- 我的车辆状态怎么样
	- 您的哪一辆车辆? (奥迪A4, 大众宝来, 沃尔沃S90)
		- 奥迪A4
			- 您的奥迪A4, 已行驶45466公里, 发动机和变速器性能正常, 车灯性能异常.

这是一个具有上下文的对话. 在"人工训练模型"中, 深度值已标明, 在返回结果的时候, 元组中有一位是代表深度值, 如果不是 0, 则需要自动监听并继续识别和理解.

## 实践

目前算法经历了两次迭代, 因为测试日志过于冗长, 只贴部分:

### 第一次迭代

* 无上下文
* 语音指令结构简单
* 递归理解

第一版算法对于测试用例, 经过调试之后, 全部通过, 然而对于实际语音的输入产生了很多问题, 只能说想法在落地的时候, 摔的比较惨.

#### 问题有以下几个

1. 语义理解的分词不够细致, 出现问题比如: "限行", 然而讯飞给的是 "限", "行", 导致无法理解
2. 没有对空白输入进行过滤, 在对讯飞关闭标点符号之后, 原本属于输入标点符号的参数, 变成了空, 应该对其进行过滤
3. 没有对输入进行一定程度的合成.
4. 提取参数除了问题, 按照之前的设想, "导航去中心校区", 分为["导航","去","中心校区"], 没有任何问题, 现在讯飞给的是["导航","去","中心","校区"], 只提取了 "校区", 而没有"中心", 所以, 需要进行合成并提取参数.

#### 改进思路:

1. 将结构化语义关键词进行细化.
2. 需要添加粗处理阶段, 由1.问题导致出现许多无关词汇, 比如 "的", "我"
3. 粗处理阶段过后, 且能理解的情况下, 要进行参数合成
4. 粗处理阶段过后, 如果不能理解, 则需要进行关键词合成

#### 调试日志:

##### 讯飞返回数据

```json
{
 "sn":1,
 "ls":false,
 "bg":0,
 "ed":0,
 "ws":[
   {
     "bg":0,
     "cw":[
       {
         "sc":0.00,
         "w":"搜索"
       }
     ]
   },
   {
     "bg":0,
     "cw":[
       {
         "sc":0.00,
         "w":"附近"
       }
     ]
   },
   {
     "bg":0,
     "cw":[
       {
         "sc":0.00,
         "w":"的"
       }
     ]
   },
   {
     "bg":0,
     "cw":[
       {
         "sc":0.00,
         "w":"加油站"
       }
     ]
   }
 ]
}
```

##### 日志(部分)

```

["搜索", "中心校区"],❓
---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** ["搜索", "中", "心", "校", "区"] ** ----
---- ** 搜索中心校区 ** ----

********************************
* 理解成功 😄
* 指令:     ["搜索", "中", "心", "校", "区"]
* 深度:     0
* 方法:     searchLocal
* 参数:     ["区"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** ["搜索", "中心", "校区"] ** ----
---- ** 搜索中心校区 ** ----

********************************
* 理解成功 😄
* 指令:     ["搜索", "中心", "校区"]
* 深度:     0
* 方法:     searchLocal
* 参数:     ["校区"]
* 描述:     
********************************


["导航", "洪家楼"],["导航", "洪", "家", "楼"]❓
---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** ["导航", "洪", "家", "楼"] ** ----
---- ** 导航洪家楼 ** ----

********************************
* 理解成功 😄
* 指令:     ["导航", "洪", "家", "楼"]
* 深度:     0
* 方法:     naviLocal
* 参数:     ["楼"]
* 描述:     
********************************

["下一首"],❓
---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** ["下", "一", "首"] ** ----
---- ** 下一首 ** ----

********************************
* 理解失败 😡
* 指令:     ["下", "一", "首"]
* 描述:     
********************************

["播放","英文","歌曲"],
---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** ["播放", "英文", "歌曲"] ** ----
---- ** 播放英文歌曲 ** ----

********************************
* 理解成功 😄
* 指令:     ["播放", "英文", "歌曲"]
* 深度:     0
* 方法:     muiscPlayEnglish
* 参数:     []
* 描述:     
********************************
```

### 第二次迭代

针对上一版算法出现的各种问题, 进行改进和得到的结果.

#### 上一版问题

1. 语义理解的分词不够细致, 出现问题比如: "限行", 然而讯飞给的是 "限", "行", 导致无法理解
2. 没有对空白输入进行过滤, 在对讯飞关闭标点符号之后, 原本属于输入标点符号的参数, 变成了空, 应该对其进行过滤
3. 没有对输入进行一定程度的合成.
4. 提取参数除了问题, 按照之前的设想, "导航去中心校区", 分为["导航","去","中心校区"], 没有任何问题, 现在讯飞给的是["导航","去","中心","校区"], 只提取了 "校区", 而没有"中心", 所以, 需要进行合成并提取参数.


#### 第二版改进思路

针对以上4个问题


1. 对于讯分分词过于细致的问题, 做了两方面的改进. 第一方面, 如果是具有操作意义的命令关键词, 则改进原来的结构化语义关键词, 使其能理解更加细致的话
2. 过滤了空白以及标点符号.
3. 此项并没有改进, 而是选择了另一种解决方式, "粗处理". 不对输入进行合成, 而是先粗处理把无关的次去掉, 交给语义理解部分, 如果理解了, 则对剩余部分进行合成, 而不是一开始对输入进行粗合成.
4. 调整了匹配关键词与子层操作顺序调整, 当该层且该区的关键词全部匹配且删除之后, 交给子层操作. 加上第三步, 变能得到细致的输出.

#### 调试日志:

##### 详细理解过程(语义理解Debug输出)

```
---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 导航去附近的加油站 ** ----
---- ** 分词指令: ["导航", "去", "附近", "的", "加油站"] ** ----
---- ** 粗处理指令: ["导航", "去", "附近", "加油站"] ** ----


- 当前层级为: 1 层 -
 - 当前区块为: 1 区 -
 1. 关键词为: 搜索
   正在查找是否包含关键词: 搜索...
 2. 关键词为: 查找
   正在查找是否包含关键词: 查找...
 3. 关键词为: 寻找
   正在查找是否包含关键词: 寻找...
 4. 关键词为: 在哪
   正在查找是否包含关键词: 寻找...
 4. 关键词为: 在哪
   正在查找是否包含关键词: 在哪...
 - 1 区结束 -
没有找到匹配的关键词!!
-- 1 层结束 --

- 当前层级为: 1 层 -
 - 当前区块为: 2 区 -
 1. 关键词为: 地图
   正在查找是否包含关键词: 地图...
 - 2 区结束 -
没有找到匹配的关键词!!
-- 1 层结束 --

- 当前层级为: 1 层 -
 - 当前区块为: 3 区 -
 1. 关键词为: 导航
   正在查找是否包含关键词: 导航...
   匹配到关键词: 导航!!!
 2. 关键词为: 去
   正在查找是否包含关键词: 去...
   匹配到关键词: 去!!!
 3. 关键词为: 怎么走
   正在查找是否包含关键词: 怎么走...
 4. 关键词为: 走
   正在查找是否包含关键词: 走...
 5. 关键词为: 驾车
   正在查找是否包含关键词: 驾车...
 6. 关键词为: 驾车去
   正在查找是否包含关键词: 驾车去...
 7. 关键词为: 导航去
   正在查找是否包含关键词: 导航去...
有子层

- 当前层级为: 2 层 -
 - 当前区块为: 1 区 -
 1. 关键词为: 附近的
   正在查找是否包含关键词: 附近的...
 2. 关键词为: 周围的
   正在查找是否包含关键词: 周围的...
 3. 关键词为: 四周的
   正在查找是否包含关键词: 四周的...
 4. 关键词为: 前方的
   正在查找是否包含关键词: 前方的...
 5. 关键词为: 最近的
   正在查找是否包含关键词: 最近的...
 6. 关键词为: 近一点的
   正在查找是否包含关键词: 近一点的...
 7. 关键词为: 附近
   正在查找是否包含关键词: 附近...
   匹配到关键词: 附近!!!
 8. 关键词为: 周围
   正在查找是否包含关键词: 周围...
 9. 关键词为: 四周
   正在查找是否包含关键词: 四周...
 10. 关键词为: 前方
   正在查找是否包含关键词: 前方...
 11. 关键词为: 最近
   正在查找是否包含关键词: 最近...
 12. 关键词为: 近一点
   正在查找是否包含关键词: 近一点...
没有子层
********************************
* 该层可以执行操作
* 用户原始指令:  导航去附近的加油站
* 操作方法名称:  naviPOI
* 操作参数个数:  1
* 操作参数名称:  ["加油站"]
* 该层已经执行操作
********************************
子层发现并已经执行了操作

********************************
* 理解成功 😄
* 指令:     ["导航", "去", "附近", "的", "加油站"]
* 深度:     0
* 方法:     naviPOI
* 参数:     ["加油站"]
* 描述:     
********************************
```

##### 简洁结果日志(测试用例全部通过)

```
---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 搜索四周的医院 ** ----
---- ** 分词指令: ["搜索", "四周", "的", "医院"] ** ----
---- ** 粗处理指令: ["搜索", "四周", "医院"] ** ----

********************************
* 理解成功 😄
* 指令:     ["搜索", "四周", "的", "医院"]
* 深度:     0
* 方法:     searchPOI
* 参数:     ["医院"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 搜索中心校校\345\214区 ** ----
---- ** 分词指令: ["搜索", "中心", "校区"] ** ----
---- ** 粗处理处理\346\214指处理\346\214指\344\273令: ["搜索", "中心", "校区"] ** ----

********************************
* 理解成功 😄
* 指令:     ["搜索", "中心", "校区"]
* 深度:     0
* 方法:     searchLocal
* 参数:     ["中心校区"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 导航洪家楼 ** ----
---- ** 分词指令: ["导航", "洪", "家", "楼"] ** ----
---- ** 粗处理指令: ["导航", "洪", "家", "楼"] ** ----

********************************
* 理解成功 😄
* 指令:     ["导航", "洪", "家", "楼"]
* 深度:     0
* 方法:     naviLocal
* 参数:     ["洪家楼"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 去附近的酒店 ** ----
---- ** 分词指令: ["去", "附近", "的", "酒店"] ** ----
---- ** 粗处理指令: ["去", "附近", "酒店"] ** ----

********************************
* 理解成功 😄
* 指令:     ["去", "附近", "的", "酒店"]
* 深度:     0
* 方法:     naviPOI
* 参数:     ["酒店"]
* 描述:     
********************************


---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 这句话不可能理解 ** ----
---- ** 分词指令: ["这", "句", "话", "不", "可能", "理解"] ** ----
---- ** 粗处理指令: ["这", "句", "话", "不", "可能", "理解"] ** ----

********************************
* 理解失败 😡
* 指令:     ["这", "句", "话", "不", "可能", "理解"]
* 描述:     
********************************


---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 导航去附近的加油站 ** ----
---- ** 分词指令: ["导航", "去", "附近", "的", "加油站"] ** ----
---- ** 粗处理指令: ["导航", "去", "附近", "加油站"] ** ----

********************************
* 理解成功 😄
* 指令:     ["导航", "去", "附近", "的", "加油\347\253站"]
* 深度:     0
* 方法:     naviPOI
* 参数:     ["去加数:     ["去加\346数:     ["去加油站"]
* 描述:     
********************************


---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 去周围的酒店 ** ----
---- ** 分词指令: ["去", "周围", "的", "酒店"] ** ----
---- ** 粗处理指令: ["去", "周围", "酒店"] ** ----

********************************
* 理解成功 😄
* 指令:     ["去", "周围", "的", "酒店"]
* 深度:     0
* 方法:     naviPOI
* 参数:     ["酒店"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 去中心校区 ** ----
---- ** 分词指令: ["去", "中心", "校区"] ** ----
---- ** 粗处理指令: ["去", "中心", "校区"] ** ----

********************************
* 理解成功 😄
* 指令:     ["去", "中心", "校区"]
* 深度:     0
* 方法:     naviLocal
* 参数:     ["中心校区"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 驾车去趵突泉 ** ----
---- ** 分词指令: ["驾车", "去", "趵突泉"] ** ----
---- ** 粗处理指令: ["驾车", "去", "趵突泉"] ** ----

********************************
* 理解成功 😄
* 指令:     ["驾车", "去", "趵突泉"]
* 深度:     0
* 方法:     naviLocal
* 参数:     ["驾车趵突泉"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 我要去千佛山 ** ----
---- ** 分词指令: ["我", "要", "去", "千佛山"] ** ----
---- ** 粗处理指令: ["去", "千佛山"] ** ----

********************************
* 理解成功 😄
* 指令:     ["我", "要", "去", "千佛山"]
* 深度:     0
* 方法:     naviLocal
* 参数:     ["千佛山"]
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 打开音乐加油界面 ** ----
---- ** 分词指令: ["打开", "音乐", "加油", "界面"] ** ----
---- ** 粗处理指令: ["打开", "音乐", "加油", "界面"] ** ----

********************************
* 理解成功 😄
* 指令:     ["打开", "音乐", "加油", "界面"]
* 深度:     0
* 方法:     showRefuel
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 预约加油 ** ----
---- ** 分词指令: ["预约", "加油"] ** ----
---- ** 粗处理指令: ["预约", "加油"] ** ----

********************************
* 理解成功 😄
* 指令:     ["预约", "加油"]
* 深度:     0
* 方法:     showRefuel
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理始理解 ** ----
---- ** 原始始指令: 加油订单 ** ----
---- ** 分词指令: ["加油", "订单"] ** ----
---- ** 粗处理指令: ["加油", "订单"] ** ----

********************************
* 理解成功 😄
* 指令:     ["加油", "订单"]
* 深度:     0
* 方法:     speechOrder
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 我订原有哪些 ** ----
---- ** 分词指令: ["我", "的", "订单", "有", "哪些"] ** ----
---- ** 粗处理指令: ["订单", "有", "哪些"] ** ----

********************************
* 理解成功 😄
* 指令:     ["我", "的", "订单", "有", "哪些"]
* 深度:     0
* 方法:     showOrder
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 上一首 ** ----
---- ** 分词指令: ["上", "一", "首"] ** ----
---- ** 粗处理指令: ["上", "一", "首"] ** ----

********************************
* 理解成功 😄
* 指令:     ["上", "一", "首"]
* 深度:     0
* 方法:     muiscPre
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 下一首 ** ----
---- ** 分词指令: ["下", "一", "首"] ** ----
---- ** 粗处理指令: ["下", "一", "首"] ** ----

********************************
* 理解成功 😄
* 指令:     ["下", "一", "首"]
* 深度:     0
* 方法:     muiscNext
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 播放英文歌曲 ** ----
---- ** 分词指令: ["播放", "英文", "歌曲"] ** ----
---- ** 粗处理指令: ["播放", "英文", "歌曲"] ** ----

********************************
* 理解成功 😄
* 指令:     ["播放", "英文", "歌曲"]
* 深度:     0
* 方法:     muiscPlayEnglish
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 你的车辆状态怎么样 ** ----
---- ** 分词指令: ["你", "的", "车辆", "状态", "怎么样"] ** ----
---- ** 粗处理指令: ["你", "车辆", "状态", "怎么样"] ** ----

********************************
* 理解成功 😄
* 指令:     ["你", "的", "车辆", "状态", "怎么样"]
* 深度:     0
* 方法:     askWhichCar&&speechCarInfo
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 我有违章吗 ** ----
---- ** 分词指令: ["我", "有", "违章", "吗"] ** ----
---- ** 粗处理指令: ["有", "违章"] ** ----

********************************
* 理解成功 😄
* 指令:     ["我", "有", "违章", "吗"]
* 深度:     0
* 方法:     askWhichCar&&speech?Info
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 车辆有违章信息吗 ** ----
---- ** 分词指令: ["车辆", "有", "违章", "信息", "吗"] ** ----
---- ** 粗处理指令: ["车辆", "有", "违章", "信息"] ** ----

********************************
* 理解成功 😄
* 指令:     ["车辆", "有", "违章","信息", "吗"]
* 深度:     0
* 方法:     askWhichCar&&speech?Info
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 帮我查一下违章信息 ** ----
---- ** 分词指令: ["帮我", "查", "一下", "违章", "信息"] ** ----
---- ** 粗处理指令: ["查", "一下", "违章", "信息"] ** ----

********************************
* 理解成功 😄
* 指令:     ["帮我", "查", "一下", "违章", "信息"]
* 深度:     0
* 方法:     showLifestyle
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 明天天气怎么样 ** ----
---- ** 分词指令: ["明天", "天气", "怎么样"] ** ----
---- ** 粗处理指令: ["明天", "天气", "怎么样"] ** ----

********************************
* 理解成功 😄
* 指令:     ["明天", "天气", "怎么样"]
* 深度:     0
* 方法:     speechWeatherTomorrow
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 后天适合洗车吗 ** ----
---- ** 分词指令: ["后天", "适合", "洗车", "吗"] ** ----
---- ** 粗处理指令: ["后天", "适合", "洗车"] ** ----

********************************
* 理解成功 😄
* 指令:     ["后天", "适合", "洗车", "吗"]
* 深度:     0
* 方法:     speechWashCarAfterTomorrow
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 限行规则是什么 ** ----
---- ** 分词指令: ["限", "行", "规则", "是", "什么"] ** ----
---- ** 粗处理指令: ["限", "行", "规则", "是", "什么"] ** ----

********************************
* 理解成功 😄
* 指令:     ["限", "行", "规则", "是", "什么"]
* 深度:     0
* 方法:     限行规则
* 参数:     []
* 描述:     
********************************

---- ** SpeechManager ** ----
---- ** 开始理解 ** ----
---- ** 原始指令: 前方有加油站吗 ** ----
---- ** 分词指令: ["前方", "有", "加油站", "吗"] ** ----
---- ** 粗处理指令: ["前方", "有", "加油站"] ** ----

********************************
* 理解成功 😄
* 指令:     ["前方", "有", "加油站", "吗"]
* 深度:     0
* 方法:     searchPOI
* 参数:     ["加油站"]
* 描述:     
********************************

```


