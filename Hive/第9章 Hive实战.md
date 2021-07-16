# 1.需求描述

统计硅谷影音视频网站的常规指标，各种TopN指标：

- 统计视频观看数Top10
- 统计视频类别热度Top10
- 统计视频观看数Top20所属类别
- 统计视频观看数Top50所关联视频的所属类别Rank
- 统计每个类别中的视频热度Top10
- 统计每个类别视频观看数Top10

# 2.数据结构

![image-20210716175017717](https://gitee.com/peng-bo19951013/Picture/raw/master/20210716175020.png)

# 3.准备工作

（1）创建外部表：gulivideo_ori，

```mysql
create external table video_ori(
  videoId string, 
  uploader string, 
  age int, 
  category array<string>, 
  length int, 
  views int, 
  rate float, 
  ratings int, 
  comments int,
  relatedId array<string>)
row format delimited fields terminated by "\t"
collection items terminated by "&"
location '/gulivideo/video';
```

（2）然后建立内部表

```mysql
create table video_orc(
  videoId string, 
  uploader string, 
  age int, 
  category array<string>, 
  length int, 
  views int, 
  rate float, 
  ratings int, 
  comments int,
  relatedId array<string>)
stored as orc
tblproperties("orc.compress"="SNAPPY");
```

（3）向 ORC 表插入数据

```mysql
insert into table video_orc select * from video_ori;
```

# 4.业务分析

## 4.1 统计视频观看数Top10

思路：使用order by按照views字段做一个全局排序即可，同时我们设置只显示前10条。

最终代码：

```mysql
SELECT
  videoid,
  views
FROM
  video_orc
ORDER BY
  views DESC
LIMIT 10;
```

## 4.2 统计视频类别热度Top10 

思路：

（1）即统计每个类别有多少个视频，显示出包含视频最多的前10个类别。

（2）我们需要按照类别group by聚合，然后count组内的videoId个数即可。

（3）因为当前表结构为：一个视频对应一个或多个类别。所以如果要group by类别，需要先将类别进行列转行(展开)，然后再进行count即可。

（4）最后按照热度排序，显示前10条。

最终代码：

```mysql
SELECT
  cate,
  COUNT(videoid) n
FROM
  (
  SELECT
    videoid,
    cate
FROM
    video_orc LATERAL VIEW explode(category) tbl as cate) t1
GROUP BY
  cate
ORDER BY
  n desc
limit 10;
```

## 4.3 统计出视频观看数最高的20个视频的所属类别以及类别包含Top20视频的个数 

（1）先找到观看数最高的20个视频所属条目的所有信息，降序排列

（2）把这20条信息中的category分裂出来(列转行)

（3）最后查询视频分类名称和该分类下有多少个Top20的视频

最终代码：

```mysql
SELECT
  cate,
  COUNT(videoid) n
FROM
  (
  SELECT
    videoid,
    cate
  FROM
    (
    SELECT
      videoid,
      views,
      category
    FROM
     video_orc
    ORDER BY
      views DESC
    LIMIT 20 ) t1 LATERAL VIEW explode(category) tbl as cate ) t2
GROUP BY
  cate
ORDER BY
  n DESC;
```

## 4.4 统计视频观看数Top50所关联视频的所属类别排序

代码：

```mysql
SELECT
  DISTINCT t4.cate,
  t5.n
FROM
  (
  SELECT
   explode(category) cate
  FROM
    (
    SELECT
      DISTINCT t2.videoid,
      v.category
    FROM
      (
      SELECT
        explode(relatedid) videoid
      FROM
        (
        SELECT
          videoid,
          views,
          relatedid
        FROM
          video_orc
        ORDER BY
          views DESC
        LIMIT 50 ) t1 ) t2
    JOIN video_orc v on
      t2.videoid = v.videoid ) t3 ) t4
JOIN (
  SELECT
    cate,
    COUNT(videoid) n
  FROM
    (
    SELECT
      videoid,
      cate
    FROM
      video_orc LATERAL VIEW explode(category) tbl as cate) g1
  GROUP BY
    cate ) t5 ON
  t4.cate = t5.cate
ORDER BY
  t5.n DESC;
```

## 4.5 统计每个类别中的视频热度Top10，以Music为例

思路：

（1）要想统计Music类别中的视频热度Top10，需要先找到Music类别，那么就需要将category展开，所以可以创建一张表用于存放categoryId展开的数据。

（2）向category展开的表中插入数据。

（3）统计对应类别（Music）中的视频热度。

最终代码：

创建表类别表：

```mysql
create table gulivideo_category(
    videoId string, 
    uploader string, 
    age int, 
    categoryId string, 
    length int, 
    views int, 
    rate float, 
    ratings int, 
    comments int, 
    relatedId array<string>)
row format delimited 
fields terminated by "\t" 
collection items terminated by "&" 
stored as orc;
```

向类别表中插入数据：

```mysql
insert into table gulivideo_category  
    select 
        videoId,
        uploader,
        age,
        categoryId,
        length,
        views,
        rate,
        ratings,
        comments,
        relatedId 
    from 
        gulivideo_orc lateral view explode(category) catetory as categoryId;
```

统计Music类别的Top10（也可以统计其他）

```mysql
select 
    videoId, 
    views
from 
    gulivideo_category 
where 
    categoryId = "Music" 
order by 
    views 
desc limit
    10;
```

## 4.6 统计每个类别视频观看数Top10

最终代码：

```mysql
SELECT
    cate,
    videoid,
    views
FROM
    (
    SELECT
        cate,
        videoid,
        views,
        RANK() OVER(PARTITION BY cate
    ORDER BY
        views DESC) hot
    FROM
        video_category ) t1
WHERE
    hot <= 10;
```

