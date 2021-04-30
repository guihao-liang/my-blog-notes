---
title: "wheel not installed by pip"
subtitle: "remember to check PYTHONPATH if package is already satisfied"
date: 2020-03-26 20:10:49
author: Guihao Liang
tags: ['python']
categories: ['python']
---

Recently, I'm assigned to redesign the packaging pipeline for turicreate [python wheel](https://pythonwheels.com/), which is a ZIP-formatted archive.

After finishing packaging the wheel package, I want to install it into the site-package of my virtual environment. But it refuses to install it there by saying,

```shell
$ pip install ~/turicreate-6.1-cp36-cp36m-macosx_10_14_x86_64.whl
Requirement already satisfied: turicreate==6.1 from file:///Users/guihaoliang/turicreate-6.1-cp36-cp36m-macosx_10_14_x86_64.whl in /Users/guihaoliang/Work/guicreate-1/debug/src/python (6.1)
```

I have never installed it with pip in my virtual env before. `env/libs/python3.6/site-packages` has no package called `turicreate`. But why it says the package is already installed? And where it's installed?

Then I realized the `PYTHONPATH`, which points to the turicreate source repository in the workspace, might be related.

> PYTHONPATH is an environment variable which you can set to add additional directories where python will look for modules and packages.

That's why my virtual env refuses to install the wheel into its `site-packages` directory because it can find turicreate package through `PYTHONPATH`. That's the trick to let python uses the packages that live outside of its default package search path. After I unset the `PYTHONPATH`, pip installs the wheel into its `site-packages` directory.

The lesson here is to check `PYTHONPATH` when wheel is not installed into the `site-packages`.
