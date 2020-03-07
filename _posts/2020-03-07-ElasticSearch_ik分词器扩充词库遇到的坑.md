---
layout: post
title: ElasticSearch ik分词器扩充词库遇到的坑
date: 2019-09-05
categories: blog
tags: [ElasticSearch,ik分词器]
description: ik扩展词库后需要更新历史数据哦~
---

# ElasticSearch ik分词器扩充词库遇到的坑

博主使用了ik分词器作为搜索服务中文分词器。ik分词器支持扩充词库，具体方法这里不提，大家自行google。

这里记录在扩充词库后大家容易遇到的坑，因为我就遇到了。

在扩充完词库后，搜索词的分词会使用到扩展的词库，一般我们使用ik_smart以匹配最长的词，让结果更精准。这里举个例子。扩充词库里加了阿莫西林胶囊。扩充词库之前，阿莫西林胶囊使用ik_smart会被分词为[阿莫西林,胶囊]，扩充完词库后，会变成分词为[阿莫西林胶囊]。

一般我们用ik_max_word来作为索引文档的分词器，以能匹配到更多的搜索关键字。我们用ik_max_word 来分词药品名称阿莫西林胶囊，在扩展词库之前它会被分成[阿莫西林、莫西、西林、胶囊]，扩展词库之后它会被分为[阿莫西林、莫西、西林、胶囊、阿莫西林胶囊]。所以以下命令搜索是能够匹配到product_name为阿莫西林胶囊的文档的。

```json
GET product/_search
{
  "query": {
    "match": {
      "product_name": "阿莫西林胶囊"
    }
  }
}
```

然而，扩充完词库后，无法匹配了。后来想到是搜索词使用了扩充词库被分词成了 [阿莫西林胶囊]，但是索引的文档没有更新。所以需要更新索引文档。

可以使用_update_by_query API完成，这个命令不会修改 _source而更新文档。

```
POST product/_update_by_query?conflicts=proceed
```

它还支持同时更新多个索引和类型

```
POST index1,index2,index3/type1,type2,type3/_update_by_query?conflicts=proceed
```

_update_by_query 在开始时获取一个索引的快照。当快照的版本和索引版本一致时则进行更新，并且递增文档版本号。

如果在获取快照之后，该文档被修改了就会遇到冲突。冲突默认情况会导致更新过程终止，之前的更新不会回滚。如果不想因为冲突导致整个更新过程终止，可以在url中添加参数conflicts=proceed。

以上，大家在扩展词库之后不要忘记更新索引数据哦~