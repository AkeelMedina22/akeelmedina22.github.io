---
title: "C++ Design Patterns"
date: 2025-06-01T19:11:23
description: "Design patterns and their benefits (Meyers Singleton)"
tags: ["HPC", "C++", "Design Patterns", "Matrix Multiplication", "Singletons"]
draft: false
---

I was recently browing [this](https://github.com/amdazm05/libtransform.cc) repository on Github, and was inspired to do something similar. I hadn't written C++ in over 2 years, since taking a course in GPU-Accelerated Computing. But, with my Masters starting in a few months, I decided to start looking in to Modern C++. I noticed this library had some interesting usage of `static`, and it was the first thing I wanted to investigate. I went down a rabbit hole of the static-initialization order, [this](https://www.youtube.com/watch?v=IMZMLvIwa-k&list=LL&index=19) video by The Cherno (I got through my course on OOP 5 years ago because of him!), and a bunch of random sources on the Meyers Singleton pattern and Registry's. 