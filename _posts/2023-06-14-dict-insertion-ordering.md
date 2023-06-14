---
title:  "Dict Insertion Ordering in Python 3.7>"
date:   2023-05-05 12:34:48 +0300
categories:
    - python
author: "Can"
excerpt: "Be careful when relying on dict insertion ordering"
header:
    overlay_image: /assets/images/chatgpt.jpeg
    overlay_filter: linear-gradient(rgba(2, 0, 36, 0.5), rgba(0, 146, 202, 0.5))
slug: pr-review-with-chatgpt
sidebar:
    nav: docs
layout: custom
---

Dict Insertion Ordering before Python 3.7 is not guaranteed. It is an implementation detail. So, if you are relying on it, you should be careful.
Here is an example:

```python
>>> d = {}
>>> d['a'] = 1
>>> d['b'] = 2
>>> d['c'] = 3
>>> d
{'a': 1, 'c': 3, 'b': 2}
```

As you can see, the order of the keys is not the same as the order of insertion. 
This is because of the implementation details of the dict. In Python 3.7, insertion ordering is guaranteed. So, the above example will be like this:

```python
>>> d = {}
>>> d['a'] = 1
>>> d['b'] = 2
>>> d['c'] = 3
>>> d
{'a': 1, 'b': 2, 'c': 3}
```

If you want to use insertion ordering in Python 3.6 or earlier, you can use OrderedDict from collections module.

```python
>>> from collections import OrderedDict
>>> d = OrderedDict()
>>> d['a'] = 1
>>> d['b'] = 2
>>> d['c'] = 3
>>> d
OrderedDict([('a', 1), ('b', 2), ('c', 3)])
```

## References

- [https://docs.python.org/3/library/stdtypes.html#dict](https://docs.python.org/3/library/stdtypes.html#dict)
- [https://docs.python.org/3/library/collections.html#collections.OrderedDict](https://docs.python.org/3/library/collections.html#collections.OrderedDict)
- Effective Python, 2nd Edition, Brett Slatkin, Item 15: Be Caution When Relying on dict Insertion Ordering
```