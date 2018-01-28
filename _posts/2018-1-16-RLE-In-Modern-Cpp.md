---
layout: post
date: 2018-01-22 12:55:00 +0300
title: "Run-Length Encoding in Modern C++"
img: spider_meadows.jpg # Add image post (optional)
tags: [C++, Modern, Modern C++, Utilities, Code, Standalone]
---

This is a test blogpost, yay! 

Test code block fragment:
{% highlight cpp %}
template<typename T>
constexpr inline size_t rle_compressor_t<T>::countUniques(const_iterator iter, 
    const_iterator end) const noexcept {
    const auto first = iter;
    while ( (iter + 1) != end && *(iter + 1) != *iter) {
        ++iter;
    }
    return std::distance(first, iter);
}
{% endhighlight %}