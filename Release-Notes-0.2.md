# New Feature
    * Exposed all supported constraints to toolbar.
    * Support sketching using Draft wire, circle/arc.
    * Add auto element visible command.
    * Support `Trace` any element.

# Bug Fix
    * Partial fix of SymPy backend
    * Fix `Trace` command
    * Fix and improve auto recompute 
    * Fix move command of fixed parts

# FreeCAD LinkStage3
    * Support editing external linked objects
        * realthunder/FreeCAD@02a6c64087758e806de9b098ebb0647b2ba61cb3
        * realthunder/FreeCAD@ea882f5d9c427f409859a90a627c0cabb297fa91
        * realthunder/FreeCAD@bff894ce1d259449ab3f4bce12c06c024aa38fb1
        * realthunder/FreeCAD@0b10c1a82859b8c1c436cf106364eb8bdfb5e368
        * realthunder/FreeCAD@eb5fad1fefc34e284e663dbe22b835c67777f68d
        * realthunder/FreeCAD@83ecc94604b8ca183f44407188a01a03311a95b0
        * realthunder/FreeCAD@39b0e1ce3323d6c5807fcfa89bab39856edc4149
        * realthunder/FreeCAD@64604d37c3d50a3d99121e532aed0a49c7a5a5a6
        * realthunder/FreeCAD@1f621a179fd1e2724feb1a81c87c944880802327
        * realthunder/FreeCAD@759c8660f9cd9fd12bb03bdee59ad37c7736f7ad
    * Handling of 3D element picking inside container, in order to support external object editing
        * realthunder/FreeCAD@8fa9544308c8631060416255297f95133e2fd392
        * realthunder/FreeCAD@47154326325ba3d8f11523858c4e91b401d14d3e
    * Introduce auto transaction to improve cross document undo/redo 
        * realthunder/FreeCAD@089f3e1c2071a2cf1b3cf65fa1b39051f5936706
        * realthunder/FreeCAD@6738a16e1fe4edf4c52f6f7f70ee39cba21a2a9c
    * Error/Warning message display enhancement in main window status bar
        * realthunder/FreeCAD@d82e26bdcad7c7065eb911dae2c1f2b523a994be
        * realthunder/FreeCAD@f22ff4ec8787e0564bc329f5d019cb88a45b71d1
    * Trigger action update on signal instead of periodic timer
        * realthunder/FreeCAD@38f51ee0e2e9454341837afe1f90e237048faff6
    * Save property status bits
        * realthunder/FreeCAD@a5ce0b7d71ee2d48e7c1e71b021d0cd4c3ccceda
    * New API Selection.updateSelection()
        * realthunder/FreeCAD@4334b74f6c544e542d90aaa5818320dc2be5abca
    * Add parameter `resolve` to `SelectionObserver` for backward compatibility
        * realthunder/FreeCAD@9a49a993fdc4012db55ef71bf1fab41ef81b3130


