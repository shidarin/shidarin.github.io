---
layout: post
title: "Nuke Python Module Tips"
date: 2014-04-09 11:54
comments: true
external-url: 
categories: nuke dev python 
---

Using the Nuke 8 python module allows us to use Nuke in a lot of great ways, but there are some tricks to using it correctly. I'll cover adding the Nuke library to the python path, using Nuke in a tool with command line arguments, and some common errors.

<!-- more -->

The big issue that I can't seem to get around is that `import nuke` will exit python if there aren't any available Nuke render licenses. This exit doesn't call a `SystemExit` exception, and isn't any of the `__builtin__` exit calls or the `os._exit` and `thread.exit` calls. Depending on the license server configuration, one should be able to query the server to see if there are licenses, and only continue if one is available.

##Adding the Nuke Library to Python

We can add Nuke's library to the python path in one of two ways- either changing the system's python path, or appending Nuke's library directly in the python script. We'll cover the second method here.

It's really as simple as importing `sys` and using `sys.path.append()`.

####Hardcoding the Absolute Location

``` python Version Locked Import
# Standard Imports
import sys

sys.path.append('/usr/apps/nuke-8.0v3/lib/python2.7/site-packages')

# Nuke Imports
import nuke
```

The downside of this is that it's locked to a specific Nuke version and location. It'd be much easier to change the python path once for the entire system, than to change the Nuke version in a bunch of different files. The middle ground is to create an environment variable each script references, and then change that when the version or location changes.

####Using an Env Variable for the Version

``` python Version Variable mark:2,5-9 
# Standard Imports
import os
import sys

sys.path.append(
    '/usr/apps/nuke-{ver}/lib/python2.7/site-packages'.format(
        ver=os.getEnv(varname='nukeVersion', value='8.0v1'),
    )
)

# Nuke Imports
import nuke
```

The benefit with this method is that if the environment variable isn't found, `os.getenv` will default to the provided `value` argument- which we can set to a known safe value- if not the latest.

####Using an Env Variable for the Location

Using an environment variable for the nuke directory location is safer if there's a possibility the application location will change.

``` python Nuke Directory Variable mark:6-7
# Standard Imports
import os
import sys

sys.path.append(
    '{nukeDir}lib/python2.7/site-packages'.format(
        nukeDir=os.getEnv(varname='nukeDir', value='/usr/apps/nuke-8.0v1/'),
    )
)

# Nuke Imports
import nuke
```

Same method only the default value is a little messier. Note that many times calling environment variables with the dictionary-like `os.environ[varname]` is preferable, as you can do `Try/Except` looking for a `KeyError` and then do more advanced behavior in the event the variable isn't set. We don't need that in this simple example.

##Using Nuke in a Command Line Interface

Having all of Nuke at our fingertips when making a CLI tool for image manipulation can be pretty powerful, but `import nuke` currently has a little problem with usage in CLIs- it eats up the command line arguments!

``` python Sample CLI Arg Script
# Standard Imports
import sys

print 'Printing command line arguments before "import nuke":'
print sys.argv

# Nuke Imports
import nuke

print 'Printing command line arguments after "import nuke":'
print sys.argv
```

Save this to a script and call it and we get:

``` linenos:false
>>> python test.py hello world
Printing command line arguments before "import nuke":
['test.py', 'hello', 'world']
Printing command line arguments after "import nuke":
['']
```

####Protecting Args

Protecting the arguments provided to the tool is pretty easy- just assign them
to another variable and re-assign them after we import nuke.

``` python Fixed CLI Arg Script mark:7-8,13-14
# Standard Imports
import sys

print 'Printing command line arguments before "import nuke":'
print sys.argv

held = sys.argv
sys.argv = ['']

# Nuke Imports
import nuke

sys.argv = held
del held

print 'Printing command line arguments after "import nuke":'
print sys.argv
```

Now we get:

``` linenos:false
>>> python test.py hello world
Printing command line arguments before "import nuke":
['test.py', 'hello', 'world']
Printing command line arguments after "import nuke":
['test.py', 'hello', 'world']
```

Now we can go on to parse those args with ArgumentParser or anything else.

##System Library Conflicts

One thing to be aware of is that there is plenty of opportunity for library conflicts between what is normally compiled into Nuke (available inside of Nuke), and what you normally have in the system (available from within a python shell).

If something doesn't work in a python shell (even with the compiled python that ships with Nuke), but works inside of Nuke, it's a good chance you're looking at a library conflict. 

If you run into these, check with your System's people and see if they can track it down. You might have to run a preload before launching the script, which is very undesirable but sometimes the only solution.

If your systems people can replicate the conflict even with a bare install of your OS, make sure to report the conflict to Foundry as they are tracking these down.