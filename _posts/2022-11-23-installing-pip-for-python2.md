---
layout: post
title:  "Installing pip for Python2"
categories: 
excerpt: "While Python2 is end of life and should not be used, there are some tools and scripts out there that are not compatible with Python3. This guide shows how to get pip installed for Python2."
image: 
permalink: 
---

Python 2 has been End of Life since January, 2020. As of this writing that was over 2 years ago, but still today there are some tools that have not been rewritten to be compatible with Python3. 

While many scripts/tools can be ran through `2to3` to convert them, some don't quite work and you just need to run Python2. In those instances `pip` is often needed but it's not easy to install `pip` for Python2 like you can for Python3. 

Thankfully, you can still install it you just have to explicitly download the `get-pip.py` and run it manually. 

To install `pip` on `python2` run these commands:

```
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
sudo python2 get-pip.py
```