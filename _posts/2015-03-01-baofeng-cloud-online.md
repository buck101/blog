---
layout:       post
title:        暴风云视频横空出世
author:     "Me"
tags:
    - 项目
---
<div id='wx_logo' style='margin:0 auto;display:none;'>
<img src='/img/2015-03-01-baofeng-cloud-online/bfc.jpg'/>
</div>

强力团队历时半年打造，基于多年在线视频领域的技术积累和运营经验，为广大用户提供点播、直播等业务的一站式解决方案。
[官方网站](http://www.baofengcloud.com)
[官方微博](http://weibo.com/u/5274297492)
[官方论坛](http://bbs.baofengcloud.com/forum.php)

<video width="320" height="240" controls="controls" id="video" autoplay="autoplay" type="video/mp4">
    <script type="text/javascript">
        jQuery(document).ready(function(){
            $.ajax({
                type: "get",
                async: false,
                url: "http://cdnquery.baofengcloud.com/c2VydmljZXR5cGU9MCZ1aWQ9NTI1MzE1OSZmaWQ9MjE0MDc3RjIyNzY4N0RBRDlFNUQ1MDZFQUNDRTkxMzk=",
                dataType: "json",
                success: function(json) {
                    document.getElementById('video').src = json.urllist[0];
                },
            });
        });
    </script>
</video>

<img src="/img/2015-03-01-baofeng-cloud-online/logo.jpg" alt="BaofengCloud" title="暴风云视频logo"/>
