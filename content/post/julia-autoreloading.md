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
One of the major components of being productive with my Python workflows was the IPython magic command `%autoreload 2`. This magic command throws a hook on the start of running an IPython cell, and reloads recompiles Python modules where source code changes are detected. This is probably the single most useful component of my workflow since I modify and work on packages in Sublime and test and develop the code in an interactive Jupyter notebook. When I first started with PyCall in Julia, I sorely missed this feature. Julia itself offers a sort-of-autoreload for Julia modules using the package `ClobberingReload.jl`. Although this works for the Julia packages, any changed Python source code is omitted. To add this functionality to `PyCall.jl`, I did some digging through IPython to figure out how Python does it and how it can be replicated for PyCall loaded modules in Julia.

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

and loaded in Julia with:

```julia
using IJulia: push_preexecute_hook
using PyCall
@pyimport rmutils.jlutils.pyreimport as prl
pyreloader = prl.PyReloader()
push_preexecute_hook(() -> pyreloader[:reload]())
```
