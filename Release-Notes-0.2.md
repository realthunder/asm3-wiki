New Feature
* Exposed all supported constraints to toolbar.
* Support sketching using Draft wire, circle/arc.
* Add auto element visible command.
* Support `Trace` of any element in the assembly container.

Bug Fix
* Partial fix of SymPy backend
* Fix `Trace` command
* Fix and improve auto recompute 
* Fix move command of fixed parts

FreeCAD LinkStage3
* Support editing external linked objects
([02a6c640](/realthunder/FreeCAD/commit/02a6c64087758e806de9b098ebb0647b2ba61cb3),
[ea882f5d](/realthunder/FreeCAD/commit/ea882f5d9c427f409859a90a627c0cabb297fa91),
[bff894ce](/realthunder/FreeCAD/commit/bff894ce1d259449ab3f4bce12c06c024aa38fb1),
[0b10c1a8](/realthunder/FreeCAD/commit/0b10c1a82859b8c1c436cf106364eb8bdfb5e368),
[eb5fad1f](/realthunder/FreeCAD/commit/eb5fad1fefc34e284e663dbe22b835c67777f68d),
[83ecc946](/realthunder/FreeCAD/commit/83ecc94604b8ca183f44407188a01a03311a95b0),
[39b0e1ce](/realthunder/FreeCAD/commit/39b0e1ce3323d6c5807fcfa89bab39856edc4149),
[64604d37](/realthunder/FreeCAD/commit/64604d37c3d50a3d99121e532aed0a49c7a5a5a6),
[1f621a17](/realthunder/FreeCAD/commit/1f621a179fd1e2724feb1a81c87c944880802327),
[759c8660](/realthunder/FreeCAD/commit/759c8660f9cd9fd12bb03bdee59ad37c7736f7ad)).
* Introduce auto transaction to improve cross document undo/redo
([089f3e1c](/realthunder/FreeCAD/commit/089f3e1c2071a2cf1b3cf65fa1b39051f5936706),
[6738a16e](/realthunder/FreeCAD/commit/6738a16e1fe4edf4c52f6f7f70ee39cba21a2a9c)).
* Handling of 3D element picking inside container, in order to support external object editing
([8fa95443](/realthunder/FreeCAD/commit/8fa9544308c8631060416255297f95133e2fd392),
[47154326](/realthunder/FreeCAD/commit/47154326325ba3d8f11523858c4e91b401d14d3e)).
* Error/Warning message display enhancement in main window status bar
([d82e26bd](/realthunder/FreeCAD/commit/d82e26bdcad7c7065eb911dae2c1f2b523a994be),
[f22ff4ec](/realthunder/FreeCAD/commit/f22ff4ec8787e0564bc329f5d019cb88a45b71d1)).
* Trigger action update on signal instead of periodic timer
([38f51ee0](/realthunder/FreeCAD/commit/38f51ee0e2e9454341837afe1f90e237048faff6)).
* Save property status bits
([a5ce0b7d](/realthunder/FreeCAD/commit/a5ce0b7d71ee2d48e7c1e71b021d0cd4c3ccceda)).
* New API Selection.updateSelection()
([4334b74f](/realthunder/FreeCAD/commit/4334b74f6c544e542d90aaa5818320dc2be5abca)).
* Add parameter `resolve` to `SelectionObserver` for backward compatibility (
([9a49a993](/realthunder/FreeCAD/commit/9a49a993fdc4012db55ef71bf1fab41ef81b3130)).


