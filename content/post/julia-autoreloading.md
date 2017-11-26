+++
tags = ["Julia", "Python"]
highlight = true
math = true
image = ""
date = "2017-11-13"
title = "AutoReloading PyCall imported Python modules in Julia"
summary = """
`%autoreload 2` for PyCall imported python modules in the IJulia environment
"""
+++

## AutoReloading PyCall Imported Modules
One of the major components of being productive with my Python workflows was the IPython magic command `%autoreload 2`. This magic command throws a hook on the start of running an IPython cell, and recompiles Python modules where source code changes are detected. This is probably the single most useful component of my workflow since I modify and work on packages in Sublime and test and develop the code in an interactive Jupyter notebook. When I first started with PyCall in Julia, I sorely missed this feature. Julia itself offers a sort-of-autoreload for Julia modules using the package `ClobberingReload.jl`. Although this (mostly) works for the Julia packages, any changed Python source code is omitted. To add this functionality to `PyCall.jl`, I did some digging through IPython to figure out how Python does it and how it can be replicated for PyCall loaded modules in Julia.

The first component is to generate a new class that mimics what is being done by the IPython Magic reloader:

```python
from IPython.extensions.autoreload import ModuleReloader
import sys

class PyReloader:
    """
    Class pretty well taken directly from the IPython %autoreload magic function..
    very untested..
    """
    def __init__(self):
        self.mr = ModuleReloader()
        self.mr.enabled = True
        self.mr.check_all = True
        self.mr.check()
        self.loaded_modules = set(sys.modules)

    def reload(self):
        self.mr.check()
        self.reload_step2()

    def reload_step2(self):
        newly_loaded_modules = set(sys.modules) - self.loaded_modules
        for modname in newly_loaded_modules:
            _, pymtime = self.mr.filename_and_mtime(sys.modules[modname])
            if pymtime is not None:
                self.mr.modules_mtimes[modname] = pymtime
        self.loaded_modules.update(newly_loaded_modules)
```

Then, it should be loaded in Julia with PyCall, and made to run before each IJulia cell using:

```julia
using IJulia: push_preexecute_hook
using PyCall
@pyimport rmutils.jlutils.pyreimport as prl
pyreloader = prl.PyReloader()
push_preexecute_hook(() -> pyreloader[:reload]())
```

And now in an interactive Julia session, if Python code is loaded using PyCall, and modified, it will be automatically reloaded as if `%autoreload 2` was working in the background!


---
Update - 25-11-2017

Using the `ClobberingReload` and the above PyReloader would casue things to get very messy when reloading Julia modules that contained the above python-reload code... the following solves this issue:

```julia
# load the AutoReload hack for Python code in Julia
using IJulia: push_preexecute_hook, preexecute_hooks
using PyCall
@pyimport rmutils.jlutils.pyreimport as pyrl
pyreloader = pyrl.PyReloader()
pyloadfunc() = pyreloader[:reload]()
if !(pyloadfunc in preexecute_hooks)
    push_preexecute_hook(pyloadfunc)
    println("loading pyautoreload...")
end
```

The modified method first checks `preexecute_hooks` to make sure the reload function hasnt already been added to the list of functions to execute. This ensures that the interactive Julia and Python sessions don't duplicate the Python reload functions!
