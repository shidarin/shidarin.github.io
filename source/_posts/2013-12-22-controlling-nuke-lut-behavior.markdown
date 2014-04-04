---
layout: post
title: "Controlling Nuke LUT behavior"
date: 2013-12-22 14:33
comments: true
external-url: 
categories: nuke dev python luts
---

The LUT implementation steps that Foundry advises for both 1D and 3D LUTs can use some work. By default, the 1D LUTs affect every channel (even alphas). For 3D LUTs it depends on the node used, but problems can vary from only affecting the default `rgb` to touching every layer- even data layers.

<!-- more -->

For those not totally familiar with the concept of LUTs, a good overview is [here](http://www.lightillusion.com/luts.html), while the wikipedia page for [3D LUTs](http://en.wikipedia.org/wiki/3D_lookup_table) is actually pretty decent.

All code presented here is covered under a standard [MIT license](/licensing), and owned by Rhythm & Hues (posted with permission). A [gist](http://gist.github.com/shidarin/9925799) of this code is also published.

##1D LUTs

The workflow the foundry advises for a 1D LUT is as follows:

>To register one of the LUTs in the Project Settings as a Viewer Process, use, for example, the following function in your init.py:

``` python Nuke Online Help 8v1 or User Guide 7v8 (page 611) http://help.thefoundry.co.uk/nuke/8.0/#user_guide/configuring_nuke/using_gizmo_viewer_process.html mark:5
nuke.ViewerProcess.register(
    "Cineon",
    nuke.createNode,
    (
        "ViewerProcess_1DLUT",
        "current Cineon"
    )
)
```

>This registers a built-in gizmo called `ViewerProcess_1DLUT` as a Viewer Process and sets it to use the Cineon LUT. The registered Viewer Process appears in the Viewer Process dropdown menu as Cineon.

####Protecting The Alpha Channel

The built-in gizmo unfortunately lacks the one option we need, limiting the application of the LUT to `rgb` channels only. However, the gizmo only includes a single node, `ViewerLUT`, that does include that option. Let's create a basic group that includes the `ViewerLUT` node, but gives us more freedom.

``` python mark:10
def lutGroup1D(lutName):
    """Builds a proper 1D viewer lut that only affects the rgb channels"""
    # Start our group
    group = nuke.nodes.Group()
    group.begin()

    inputNode = nuke.nodes.Input()  # Group input

    conversion = nuke.createNode('ViewerLUT')
    conversion['rgb_only'].setValue(True)
    conversion['current'].setValue(lutName)
    conversion.setInput(0, inputNode)

    nuke.nodes.Output(
        inputs=[conversion],
    )

    group.end()

    return group
```

The key is line 10, we need to toggle the `rgb_only` bool checkbox on the `ViewerLUT` node so that it doesn't affect the alpha channels anymore.

Despite recommending the usage of the `ViewerProcess_1DLUT`, the Foundry does point out that parameter of the `ViewerLUT` on the very next page.

Now our 1D LUT is no longer affecting alpha channels, but unfortunately several layers representing data channels. Motion vectors, zdepth, etc, are still being hit with our LUT, as those channels are being shuffled into `rgb` for display purposes.

####Protecting Data Channels

We need to add a `Remove` node to strip these layers before the `ViewerLUT` node, and then a `Copy` node afterwards to add them back in. If those layers aren't present in the incoming stream, Nuke doesn't complain.

``` python start:8
    # Remove special layers if present
    
    REMOVE_KNOBS = {
        'operation': 'remove',
        'channels': 'depth',
        'channels2': 'motion',
        'channels3': 'forward',
        'channels4': 'backward'
    }

    remove = nuke.nodes.Remove(
        inputs=[inputNode],
    )
    for knob in REMOVE_KNOBS:
        remove[knob].setValue(REMOVE_KNOBS[knob]) 
```

When changing many knob values at once, it's often helpful to define a dictionary whose keys are the knob name, and values are the desired knob value. Then we can iterate through the dictionary, grabbing the knob object, and setting it to the retrieved value from the dictionary.

Here's the copy node we need that adds the removed layers back in from the original input image stream:

``` python start:28 mark:33-35
    # Add our special channels back in
    copy = nuke.nodes.Copy(
        inputs=[inputNode, conversion],
        channels='all'
    )
    # We need to eliminate any attempts at auto copy.
    for i in range(4):
        copy['from' + str(i)].setValue('none')
```

With the copy, we set it to `all` channels and remove the automatically filled in values in the `from0` and on knobs. We'll change the input to the `Output` node below to be linked to the `copy` node, and then we'll add an expanded docstring.

####Final 1D LUT Function

``` python lutGroup1D (with changes marked) mark:3-15,23-36,40,43-50,53
def lutGroup1D(lutName):
    """Builds a proper 1D viewer lut that only affects the rgb channels

    Args:
        lutName : (str)
            The 1d lut name. Must be already registered in Nuke's settings.

    Raises:
        N/A

    Returns:
        (<nuke.node.Group>)
            A nuke group node that will handle the 1d LUT correctly.

    """
    # Start our group
    group = nuke.nodes.Group()

    group.begin()

    inputNode = nuke.nodes.Input()  # Group input

    REMOVE_KNOBS = {
        'operation': 'remove',
        'channels': 'depth',
        'channels2': 'motion',
        'channels3': 'forward',
        'channels4': 'backward'
    }

    # Remove special layers if present
    remove = nuke.nodes.Remove(
        inputs=[inputNode],
    )
    for knob in REMOVE_KNOBS:
        remove[knob].setValue(REMOVE_KNOBS[knob])

    conversion = nuke.createNode('ViewerLUT')
    conversion['rgb_only'].setValue(True)
    conversion['current'].setValue(lutName)
    conversion.setInput(0, remove)

    # Add our special channels back in
    copy = nuke.nodes.Copy(
        inputs=[inputNode, conversion],
        channels='all'
    )
    # We need to eliminate any attempts at auto copy.
    for i in range(4):
        copy['from' + str(i)].setValue('none')

    nuke.nodes.Output(
        inputs=[copy],
    )

    group.end()

    return group
```

It's important to return the `group` object, because that returned object will be used as the Viewer Process itself!

####Adding A LUT With lutGroup1D()

Now that we have our function, we can use it in place of the `ViewerProcess_1DLUT` node:

``` python mark:3,5
nuke.ViewerProcess.register(
    "Cineon",
    lutGroup1D,
    (
        "Cineon",
    )
)
```

##3D LUTs

We have 3 goals for 3D LUTs:

1. Make sure every color layer is being affected by the LUT transform.
2. Protect data layers from the transform.
3. Ensure the transform happens in the correct colorspace.

We don't need to worry about 3D LUTs affecting the alpha channel, because while a 1D LUT affects every channel the same, a 3D LUT has instructions for each color channel specifically- meaning it won't have any instructions for alpha channels and will leave them alone.

Foundry gives some example nodes to use for 3D LUTs, such as the `Vectorfield` node, but doesn't mention the best node to use, the [OpenColorIO](http://opencolorio.org/) node, `OCIOFileTransform`.

#####Why Use OpenColorIO?

[OpenColorIO](https://github.com/imageworks/OpenColorIO) is an open source project sponsored by Sony Imageworks. It has a lot of features, but right now we're primarily interested in it because it allows us to select which layers to apply the LUT to- including all of them.

In comparison, the `Vectorfield` node only affects the default `rgb` layer, which is a pretty large handicap. The `Vectorfield` node does have an option for colorspace conversion of input images, which is a plus. We'll have to do our own colorspace conversion before applying the LUT with `OCIOFileTransform`, as almost all LUTs expect the inputted channels to be in a Log colorspace.

####Building A Basic Viewer Process With OCIOFileTransform

We'll use the basic framework of our `lutGroup1D()` function to create a `lutGroup3D()`, including protection of data layers.

``` python mark:1,24-29,33
def lutGroup3D(lutFile):
    """Builds a proper 3D viewer lut that only affects the rgb channels"""
    # Start our group
    group = nuke.nodes.Group()
    group.begin()

    inputNode = nuke.nodes.Input()  # Group input

    REMOVE_KNOBS = {
        'operation': 'remove',
        'channels': 'depth',
        'channels2': 'motion',
        'channels3': 'forward',
        'channels4': 'backward'
    }

    # Remove special layers if present
    remove = nuke.nodes.Remove(
        inputs=[inputNode],
    )
    for knob in REMOVE_KNOBS:
        remove[knob].setValue(REMOVE_KNOBS[knob])

    # Use our 3d LUT file
    mainLut = nuke.nodes.OCIOFileTransform(
        inputs=[remove],
        channels='all',
        file=lutFile
    )

    # Add our special channels back in
    copy = nuke.nodes.Copy(
        inputs=[inputNode, mainLut],
        channels='all'
    )
    # We need to eliminate any attempts at auto copy.
    for i in range(4):
        copy['from' + str(i)].setValue('none')

    nuke.nodes.Output(
        inputs=[copy],
    )

    group.end()

    return group
```

We've only had to change a few lines here. Instead of a string representing an already registered LUT name, the `OCIOFileTransform` requires a full filepath, which we need to provide in the `lutFile` arg.

In addition, we've set `channels` to `all`, which (unlike the `Vectorfield` node) will ensure that every layer's `rgb` will get affected. We're already protecting the data channels by removing them, just like in `lutGroup1D()`.

####Setting Input Colorspace

Most LUTs require an input colorspace that's different from Linear space that Nuke works in. `Vectorfield` has this conversion as a built in option, but `OCIOFileTransform` does not, so we'll have to add it before getting the correct result.

Traditionally the expected input colorspace is Cineon Log, but on an Arri Alexa show it would be LogC. We'll specify which in an arg of the function, and use use a `ViewerLUT` node to convert from to that.

Why use `ViewerLUT` as opposed to a `Colorspace`? Because `ViewerLUT` lets us specify any of the 1D LUTs currently within the Nuke script, not just the built in ones. This means your pipeline can specify a different version of Cineon Log or an older revision of Alexa LogC.

We'll make this input colorspace optional, so if the 3D LUT is built for linear values you can omit the arg.

####Final 3D LUT Function

``` python lutGroup3D (with changes marked) mark:1,4-17,39-46,50
def lutGroup3D(lutFile, inputSpace=None):
    """Builds a proper 3D viewer lut that only affects the rgb channels

    Args:
        lutFile : (str)
            The filepath to a 3d lut
        inputSpace=None : (str)
            The colorspace the 3d lut file expects.

    Raises:
        N/A

    Returns:
        (<nuke.node.Group>)
            A nuke node group that will handle the 3d LUT correctly.

    """
    # Start our group
    group = nuke.nodes.Group()
    group.begin()

    inputNode = nuke.nodes.Input()  # Group input

    REMOVE_KNOBS = {
        'operation': 'remove',
        'channels': 'depth',
        'channels2': 'motion',
        'channels3': 'forward',
        'channels4': 'backward'
    }
    
    # Remove special layers if present
    remove = nuke.nodes.Remove(
        inputs=[inputNode],
    )
    for knob in REMOVE_KNOBS:
        remove[knob].setValue(REMOVE_KNOBS[knob])

    if inputSpace:
        # Convert to Cineon or LogC colorspace
        conversion = nuke.createNode('ViewerLUT')
        conversion['rgb_only'].setValue(True)
        conversion['current'].setValue(inputSpace)
        conversion.setInput(0, remove)
    else:
        conversion = remove

    # Use our 3d LUT file
    mainLut = nuke.nodes.OCIOFileTransform(
        inputs=[conversion],
        channels='all',
        file=lutFile
    )

    # Add our special channels back in
    copy = nuke.nodes.Copy(
        inputs=[inputNode, mainLut],
        channels='all'
    )
    # We need to eliminate any attempts at auto copy.
    for i in range(4):
        copy['from' + str(i)].setValue('none')

    nuke.nodes.Output(
        inputs=[copy],
    )

    group.end()

    return group
```

####Adding A LUT With lutGroup3D()

Now that we have our function, we can use it in place of the `Vectorfield` node:

``` python
nuke.ViewerProcess.register(
    "Rambo XXIV Dailies LUT",
    lutGroup3D,
    (
        "/proj/ramboXXIV/share/luts/iridas/dailes.cube",
        "AlexaV3LogC",
    )
)
```

####Final Thoughts

That should be it. You've still got some odds and ends left over- if you want to add a 3D LUT for every *.cube file found in a folder, you need some way of deriving the LUT's desired input colorspace.

Also, a slicker means of doing this would involve creating a base LUT object class, with methods for doing things like deriving input colorspace, building the lut group, and registering it. A class level variable could then keep track of all the created LUTs for science.