---
date: 2019-11-25 14:49:08
comments: false
thumbnail: https://cdn.jsdelivr.net/gh/removeif/blog_image/img/2020/20201030170800.png
---
<div class = "text-center"><h1>给我留个言吧</h1></div><div class = "text-tips">

<!-- <span id="busuanzi_container_page_pv"><span id="busuanzi_value_page_pv"></span></span></div> -->
<div id="comment-container1"><div class="text-tips">留言板加载中，请稍等...</div></div>
<!-- <link rel="stylesheet" href="https://cdnjs.loli.net/ajax/libs/gitalk/1.6.0/gitalk.css"/> -->
  <link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">

<script>
    $.getScript("https://unpkg.com/gitalk/dist/gitalk.min.js", function () {
        var gitalk = new Gitalk({
            clientID: 'a1ad219ff90da8100007',
            clientSecret: '97b9266d67d87e37cdcb328e10460af8295a4e7d',
            id: '2423',
            repo: 'hxq94.github.io',
            owner: 'hxq94',
            admin: "hxq94",
            createIssueManually: true,
            distractionFreeMode: false
        });
        gitalk.render('comment-container1');
    });
</script>