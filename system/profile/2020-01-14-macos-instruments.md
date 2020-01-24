---
title: Profiling with Macos Instruments
style: post
date: 2020-01-14 22:31:40
---

recently, I work on profiling the performance regression of the stype transfer toolkit of turicreate. And I need to know which part really slows the performance down from 250ms to 800ms.

I utilize the tool called `instrument` to inspect the performance and reported the [investigation](https://github.com/apple/turicreate/issues/2798#issuecomment-574429475).

## Resources

- [ ] [WWDC instruments demo](https://developer.apple.com/videos/play/wwdc2019/411/)