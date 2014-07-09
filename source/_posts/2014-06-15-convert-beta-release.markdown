---
layout: post
title: "cdl_convert Beta Release"
date: 2014-06-15 11:52
comments: true
external-url: 
categories: cdl_convert color
---

The last few months I've been working on a python module that works as both a simple script to convert between [ASC ColorDecisionList](http://en.wikipedia.org/wiki/ASC_CDL) formats, and a fully fledged Python API.

<!-- more -->

The module can be installed with pip via the [pypi repo](http://pypi.python.org/pypi/cdl_convert/):

``` linenos:false
pip install cdl_convert    
```

Or you can grab the source on [github](https://github.com/shidarin/cdl_convert).

It's got [full documentation at ReadTheDocs](http://cdl-convert.readthedocs.org/), and we use [issue tracking on github](https://github.com/shidarin/cdl_convert/issues?state=open).

It's mostly feature complete now- the only features it lacks are:

* Parse CMX EDL
* Parse OCIOCDLs in .nk scripts
* Write OCIOCDLs into .nk scripts
* Generate a Nuke node (if called from within a Nuke session)

The rest are bug fixes.

## Sample Script Usage

This will take a `FLEx` file, named `big_edl_with_cdls.flex`, and convert
it to individual `cc` files.

``` linenos:false
cdl_convert big_edl_with_cdls.flex -o .cc
```

## Sample API usage

Parsing a file from disk:

``` python linenos:false
>>> cc = cdl_convert.parse_file('./xf/015.cc')
>>> cc.slope
(Decimal('1.02401'), Decimal('1.00804'), Decimal('0.89562'))
>>> cc.offset
(Decimal('-0.00864'), Decimal('-0.00261'), Decimal('0.03612'))
>>> cc.power
(Decimal('1.0'), Decimal('1.0'), Decimal('1.0'))
>>> cc.sat
Decimal('1.2')
>>> cc.id
'015_xf_seqGrade_v01'
>>> cc.file_in
'/Users/niven/cdls/xf/015.cc'
```

Or create one from scratch and attach it to a collection:

``` python linenos:false
>>> from decimal import Decimal
>>> coll = cdl_convert.ColorCollection()
>>> cc = cdl_convert.ColorCorrection('myUniqueId')
>>> cc.slope = [1, 2, 3]
>>> cc.offset = (
    Decimal('2334.3789378921378901921738979'),
    Decimal('3789.32789137891278947891478912738972198731289'),
    Decimal('0.000000000000000000000000000000000001')
)                                                                     
>>> cc.power = (1.243, 4.53, 9.23572)
>>> cc.sat = '6.7'
>>> coll.append_child(cc)
True
>>> print coll.xml
<ColorCorrectionCollection xmlns="urn:ASC:CDL:v1.01">
    <ColorCorrection id="myUniqueId">
        <SOPNode>
            <Slope>1.0 2.0 3.0</Slope>
            <Offset>2334.3789378921378901921738979 3789.32789137891278947891478912738972198731289 0.000000000000000000000000000000000001</Offset>
            <Power>1.243 4.53 9.23572</Power>
        </SOPNode>
        <SATNode>
            <Saturation>6.7</Saturation>
        </SATNode>
    </ColorCorrection>
</ColorCorrectionCollection>

>>> coll.set_to_cdl()
>>> print coll.xml
<ColorDecisionList xmlns="urn:ASC:CDL:v1.01">
    <ColorDecision>
        <ColorCorrection id="myUniqueId">
            <SOPNode>
                <Slope>1.0 2.0 3.0</Slope>
                <Offset>2334.3789378921378901921738979 3789.32789137891278947891478912738972198731289 0.000000000000000000000000000000000001</Offset>
                <Power>1.243 4.53 9.23572</Power>
            </SOPNode>
            <SATNode>
                <Saturation>6.7</Saturation>
            </SATNode>
        </ColorCorrection>
    </ColorDecision>
</ColorDecisionList>
```

Share, enjoy and give feedback.

Did I mention the test suite with ***glorious*** 100% coverage?
