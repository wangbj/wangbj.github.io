---
layout: post
title:  "Pythagorean?"
date:   2018-12-31
categories: programming
---

came across this interesting read from HN: 

[Modern-C-Lamentations](http://aras-p.info/blog/2018/12/28/Modern-C-Lamentations/). It really refreshed my mind with modern c++.

{% highlight haskell %}
triples :: Int -> [ (Int, Int, Int) ]
triples = flip take [ (x, y, z) | z <- [1..], x <- [1..z], y <- [x..z], x*x+y*y==z*z]
{% endhighlight %}

Now I know one more reason why I love FP better and more productive :)
