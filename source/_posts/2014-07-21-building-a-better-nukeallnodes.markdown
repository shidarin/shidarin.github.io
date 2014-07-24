---
layout: post
title: "Building a Better `nuke.allNodes`"
date: 2014-07-21 11:04
comments: true
external-url: 
categories: nuke python
---

Nuke's built in `nuke.allNodes()` function offers two very useful features- the ability to filter for a single node class, and the ability to recurse deep into groups. What it doesn't offer is the ability to do both at once. If you set a filter class and `recurseGroups=True`, the `recurseGroups` argument will be ignored and only the first level results will be found. Also of importance is that Nuke attempts to derive what `Group` you want to search within if you don't provide a `group` argument, but it usually guesses poorly. Let's fix those bugs as easily as we can.

<!-- more -->

## The Basics

Since we need to be a drop in replacement for `nuke.allNodes()`, we need to start with the same arguments, titled the same way, in the same order. This way you can import our `allNodes()` and have it override the `nuke.allNodes()`.

``` python
def allNodes(filter=None, group=None, recurseGroups=False):
```

Now let's try for an exact passthrough of our given arguments. Nuke will trip us up here, as the default values for `filter` and `group` are not None, so we can't just pass our exact received arguments in. Instead, we'll construct a keyword argument dictionary, and pass that in.

``` python
def allNodes(filter=None, group=None, recurseGroups=False):
    kwargs = {
        'filter': filter,
        'group': group,
        'recurseGroups': recurseGroups
    }
    
    # Remove empty arguments
    if not filter:
        del kwargs['filter']
    if not group:
        del kwargs['group']

    return nuke.allNodes(**kwargs)
```

We now have a simple wrapper. Our arguments are passed through correctly when given, and `None` is never passed to `nuke.allNodes()`.

## Fixing the Group Bug

If we want Nuke to stop attempting to (badly) guess what Group we want to search within, we'll set the default `group` value to be `nuke.root()`:

``` python mark:1,11
def allNodes(filter=None, group=nuke.root(), recurseGroups=False):
    kwargs = {
        'filter': filter,
        'group': group,
        'recurseGroups': recurseGroups
    }
    
    # Remove empty arguments
    if not filter:
        del kwargs['filter']

    return nuke.allNodes(**kwargs)
```

## Fixing the Filter Bug

The bug occurs when both `recurseGroups=True` and a `filter` argument are passed to `nuke.allNodes()`. Anything else can be given and returned straight from the native `allNodes`. It then makes sense to start our function off with an `if` clause, placing the parts of the function we've already created behind the `else`:

``` python
def allNodes(filter=None, group=nuke.root(), recurseGroups=False):
    if filter and recurseGroups:
        pass
    else:
        kwargs = {
```

Then we need to begin constructing the meat of the fix. We have two choices:

1. Search for all group nodes with `nuke.allNodes()`, then execute `nuke.allNodes()` within those group nodes with the given `filter` argument.
2. Search for all nodes period, then construct & return a list containing only nodes which match our filter.

Let's use `timeit` to figure out which runs faster.

### Timing the Fixes

I have a script with a bunch of group nodes, and some other nodes hidden inside. A few are ColorWheels, and we'll search for those.

``` python linenos:false mark:23,26,29,36,39,42
import timeit

# Search for Groups, then search for wanted node
def bygroup(filter):
    all_groups = nuke.allNodes('Group', recurseGroups=True)
    all_nodes = []
    for sub_group in all_groups:
        all_nodes.extend(nuke.allNodes(filter, sub_group))

    return all_nodes

# Search for all nodes, then use list comprehension to filter
def listcomp(filter):
    all_nodes = nuke.allNodes(recurseGroups=True)

    return [node for node in all_nodes if node.Class() == filter]
    
# Run each a million times

# By Group ====================================================================
timeit.timeit('bygroup("ColorWheel")', "from __main__ import bygroup", number=1000000)
# Result:
0.78045105934143066
timeit.timeit('bygroup("ColorWheel")', "from __main__ import bygroup", number=1000000)
# Result:
0.79747390747070312
timeit.timeit('bygroup("ColorWheel")', "from __main__ import bygroup", number=1000000)
# Result:
0.79068994522094727
# =============================================================================


# List Comprehension ==========================================================
timeit.timeit('listcomp("ColorWheel")', 'from __main__ import listcomp', number=1000000)
# Result:
0.75563311576843262
timeit.timeit('listcomp("ColorWheel")', 'from __main__ import listcomp', number=1000000)
# Result:
0.74588203430175781
timeit.timeit('listcomp("ColorWheel")', 'from __main__ import listcomp', number=1000000)
# Result:
0.74688887596130371
# =============================================================================

```

The results are close enough to not really matter, but we'll still use the list comprehension method because it's cleaner ***and*** faster.

## Final `allNodes()`

Our final replacement for `nuke.allNodes()` looks like this:

``` python
def allNodes(filter=None, group=nuke.root(), recurseGroups=False):
    """ Wraps nuke.allNodes to allow filtering and recursing
    
    Args:
        filter=None : (str)
            A Nuke node name to filter for. Must be exact match.
        
        group=nuke.root() : (<nuke.nodes.Group>)
            A Nuke node of type `Group` to search within. If not provided, will
            begin the search at the root level.
        
        recurseGroups=False : (bool)
            If we should continue our search into any encountered group nodes.
    
    Returns:
        [<nuke.Node>]
            A list of any Nuke nodes found whose Class matches the given filter.
    
    Raises:
        N/A
    
    """
    # First we'll check if we need to execute our custom allNodes function.
    # If we don't have a filter AND recurseGroups=True, `nuke.allNodes` will
    # do the job fine.
    if filter and recurseGroups:
        # Search for every node, then filter using a list comprehension.
        # Faster than searching for all groups, then searching again
        # for the filter.
        all_nodes = nuke.allNodes(group, recurseGroups=True)

        return [node for node in all_nodes if node.Class() == filter]

    else:
        # We just need to execute Nuke's `nuke.allNodes` function.
        # But we need to modify our list of keyword arguments and remove
        # the filter argument if it wasn't passed, otherwise Nuke chokes.
        kwargs = {
            'filter': filter,
            'group': group,
            'recurseGroups': recurseGroups
        }

        # Remove empty arguments
        if not filter:
            del kwargs['filter']

        return nuke.allNodes(**kwargs)
```