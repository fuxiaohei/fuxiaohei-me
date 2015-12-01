```ini
[meta]
title = FuXiaohei.Me
subtitle = 傅小黑的自留地
; print in html <meta>
keyword = 傅小黑,fuxiaohei,blog,go,golang
; print in html <meta>
desc = FuXiaohei.Me - 傅小黑的自留地
; build links for feed, sitemap
domain = fuxiaohei.me
root = http://fuxiaohei.me

[nav]
; reference to [home] block, same below.
-:home
-:archive
-:about
-:friends
; -:source

[nav.home]
link = /
title = Home
i18n = 文章
hover = home

[nav.archive]
link = /archive
title = 归档
i18n = archive
hover = archive

[nav.about]
link = /about-me
title = 关于
i18n = about
blank = true
hover = about

[nav.friends]
link = /friends
title = 好友
i18n = friends
blank = true
hover = friends


[comment.disqus]
site = fuxiaohei

```