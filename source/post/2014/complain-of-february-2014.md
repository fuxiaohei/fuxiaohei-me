```toml
title = "2月的一点抱怨"
slug = "complain-of-february-2014"
date = "2014-02-28 23:36:30"
author = "fuxiaohei"
tags = ["工作","开源"]

```

2月打春，可惜，西安还是在冬天的雾霾和淅淅沥沥里度过。自己还年轻，24不到，可总觉得有些工作的累心。怎么说呢？一个破烂的php项目折腾了半个月。而工作之余也在为开源的东西做些工作。可能是事情多了，累了吧。

#### 一个糟糕的php项目

公司承接一个海外的php项目，我刚开始满心欢喜。看了看他的代码，自己实现MVC模式，很容易理解。不过当我真的按照要求动手的时候，杯具开始。首先，是**到处的超全局变量**。

```php
$eventID = isset($_POST['event']) ? trim($_POST['event']) : NULL;
$title = isset($_POST['title']) ? trim($_POST['title']) : "";
$video_status = isset($_POST['video_status']) ? trim($_POST['video_status']) : "";
$video_type = isset($_POST['video_type']) ? trim($_POST['video_type']) : "";

if (isset($_FILES['file']) && !empty($_FILES['file']['tmp_name'])) {
//......
}
```

你再怎么样也别这样啊，不嫌繁琐啊。然后是，**奇特的类属性定义**。<!--more-->

```php
protected $_allow_add_new;
protected $_allow_edit;
protected $_allow_game_summaries;
protected $_allow_video_summaries;
protected $_allow_video_tags;
protected $_allow_video_markers;
protected $_page;
protected $_allowed_videos_to_view;
protected $_allowed_edit_evaluators;
protected $_allowed_edit_video_settings;
protected $_pageNo;
protected $_currentTab;
protected $_conf;
protected $_userSettings;
protected $_activeF;
protected $_perPage;
protected $_globalHelper;
protected $_userID;
protected $_titleID;   
```

你这里面有用户信息，有权限控制，还有分页。尼玛搞成几个class不就的啦，塞在一起做啥啊。最后是**蛋疼数据库查询**。

```php
$where_array = array();
$settingsHelper = new settings();

$sql = "SELECT events.*,
  users.name as owner_name,
  sports.name as sport_name,
  ( SELECT if(ue.id IS NOT NULL, 1, 0) 
  FROM user_events ue
  WHERE ue.user_id = ? AND ue.event_id = events.event_id 
  GROUP BY ue.event_id,ue.user_id ) AS selected";
array_push($where_array, $settingsArray[0]['user_id']);

$sql .= " FROM events
  LEFT JOIN sports ON events.sport_id = sports.sport_id
  LEFT JOIN videos ON videos.event_id = events.event_id";

if (count($titles) && $settingsArray['setTitle'] != $settingsHelper->get('USER_LEVEL_PARTICIPANT')) {
  $sql .= " LEFT JOIN users on users.user_id = videos.user_id";
  $titles_list = implode(",", $titles);
  $sql2 = " WHERE users.title_id IN ( %s )";
  $sql .= sprintf($sql2, $titles_list);
}
else {
  $sql .= " LEFT JOIN users on users.user_id = events.user_id";
  $sql .= " WHERE events.event_id IS NOT NULL";
}
      
if ($settingsArray['setEvents'] != null && count($settingsArray['setEvents']) > 0) {
  $events_list = implode(",", $settingsArray['setEvents']);
  $sql2 = " AND events.event_id IN ( %s )";
  if ($settingsArray[0]["group_id"] == 0) {
    $sql .= sprintf($sql2, "SELECT event_id FROM user_events WHERE user_id = " . (int)$settingsArray[0]["user_id"]);
  } else {
    $sql .= sprintf($sql2, $events_list);
  }
}
else if ($settingsArray['setEvents'] === false) {
  $sql .= " AND events.event_id = 0";
}

if ($selectedSport != null && $selectedSport != -1) {
  $sql .= " AND sports.sport_id = ?";
  array_push($where_array, $selectedSport);
}

$sql .= " GROUP BY events.event_id";

if ($countOnly == false) {

$sql .= " ORDER BY events.event_time DESC";

if ($offset != false && $perPage != false) {
  if ($offset === null) {
    $offset = 0;
  }
  if ($perPage === null) {
    $perPage = $this->getPerPage();
  }
  $sql.= " LIMIT $offset, $perPage";
  }
}

$this->_setSql($sql);
$result = $this->getAll($where_array);
```

把各种权限，用户状态和过滤信息杂糅一起，拼接如此复杂的一次查询。你让我怎么猜你到底执行的是啥sql啊。

为了这个破项目，折腾了一个多星期。终于是在各种烂代码的补充下，完成了客户的要求。TM太坑了。

#### 开源事业

最近一直在研究go，比如这个博客程序。搭建了新的一页面 [开源](/open-source.html) 简单介绍和记录这个程序。在开源中国，我也添加了这个项目 [Fxh.Go](http://www.oschina.net/p/fxhgo)。总之，算是步入正轨的。最简单的想法，让这个程序一直活下去吧。

然后是[gobuild.io](http://gobuild.io)。其实我没写什么核心功能，只是做的全新的界面。写了点前端js，css和html而已。看起来有点高大上的感觉，比较满意。原作者[@skyblue](https://github.com/shxsun/)最近也在忙着换工作，很多进度慢下来，大家也就休息了。不过目前还是有些细节优化在做，慢慢来吧。能在年内把这个做的像模像样，就很不错啦。

最后，是在研究php的异步扩展[swoole](http://www.swoole.com)。这货不支持windows，我也就只能在ubuntu下工作研究，甚是不上。不过感觉这个东西非常炫酷屌，很有意思。回头看看写个简单的东西，试试效果哈哈。

#### 一句话结尾

其实我是来抱怨工作中的破项目的，就这么简单。
