---
layout: post
title: "SMPTE Timecode Segment"
date: 2014-04-19 15:50
comments: true
external-url: 
categories: python editing vfx
---

While developing a test suite for another project, I started writing a bit of python to generate SMPTE timecode- start, end, and duration. I needed to have the ability for it to generate random timecode as well.

<!-- more -->

In the end, it became so large it needed its own tests (Timecode can be pretty complex) and I ended up throwing it on a gist. It's of limited use now, my end goal would be to develop a datetime style TimeCode class which could be added,
subtracted, delta'd etc.

## Other SMPTE Timecode projects

I might end up forking the abandoned [pytimecode](https://pypi.python.org/pypi/pytimecode.py/0.1.0) (which is thankfully under an MIT License) and combining it with something like [smptecalc](https://www.gitorious.org/smptecalq), which is unfortunately without any attached license (so forking it and developing it on my own is legally iffy).

## SMPTE TimeCode Segment

### Usage

`TimeCodeSegment` takes `hour`, `minute`, `second`, and `frame` args for initializing the starting timecode, and `duration` (in frames) and `fps` for calculating an ending timecode and duration. If any of these args (except fps) are not provided during class initializing, they will be randomized. You can actually just call TimeCodeSegment() to get a completely random, 24 frames per second timecode segment.

``` python linenos:false
>>> tc = TimeCodeSegment(hour=12, minute=45, second=10, frame=20, duration=300)
>>> tc.start
'12:45:10:20'
>>> tc.end
'12:45:23:08'
>>> tc.dur
'00:00:12:12'
```

### Testing

The gist comes with a 100% coverage test suite, so if you just call the python file you'll run the unittests.

``` linenos:false
python smpte_timecode_segment.py 
..........
----------------------------------------------------------------------
Ran 10 tests in 0.001s

OK
```

## Code

{% gist 11091783 %}