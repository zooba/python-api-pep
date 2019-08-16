CPython API Design Guidelines

# Abstract

This document is intended to be a set of guiding principles for development of the current CPython API. Future additions and enhancements to the CPython API should follow, or at least be influenced by, the principles described here. At a minimum, any new or modified APIs should be able to be categorised according to the terminology defined here, even if exceptions have to be made.


# This document is NOT ...

This document is NOT a design of a completely new API (though the design of a hypothetical new API should be guided by this document).

This document is NOT documentation of the current API (though the principles in this document should become more obvious in the current API over time).

This document is NOT a set of binding rules in the same sense as PEP 7 and PEP 8 (though designs should be tested against it and new exceptions should be rare).

This document is NOT permission to make backwards-incompatible modifications to the current API (though backwards-incompatible modifications should still be made where justified).


# This document IS ...

This document is language agnostic, and applies to both Python and C APIs (though language-specific references are used for the benefit of readers familiar with CPython's implementation).

This document is meant to be referenced in discussions (though preferably by concept rather than section number).

This document is supposed to cause arguments about how to classify existing API members (though with shared definitions it should be possible to reach agreement).

# Definitions

A common understanding of certain terms is necessary to talking about any API, and so this section defines the terms we use to discuss the CPython API. This section has two goals: to clarify existing common terminology, and to introduce new terminology. Terms are presented in a logical order, rather than alphabetically.


## Existing terms

**Application**: A program that can be launched directly. Compare and contrast with *extension*. CPython is normally considered an application when used via the `python` or `python3` command.

**Extension**: A program that cannot be launched directly but must be loaded by an application. Python modules, native or otherwise, are considered extenions. When embedded into another application, CPython is considered an extension.

**Target Runtime**: The combination of operating system and externally-provided code that an application is compiled to use. Referred to as "target runtime" to distinguish from "Python runtime". For CPython, the target runtime is usually the processor, operating system and C runtime implementation.

**Native extension**: A subset of all extensions that are compiled to the same target runtime as the application they integrate with. CPython native extensions are compiled to native code that uses the CPython ABI. For contrast, Python source and bytecode files are *not* considered native extensions.

**API**: Application Programming Interface. The set of interactions defined by an application to allow extensions to extend, control, and interact with the application. An API typically defines functions, classes and values as abstract interfaces. CPython has one API that applies for all scenarios in all contexts, though practical scenarios only use a subset of this API. (Referring to a subset of the API as "the <x> API" is generally acceptable.)

**ABI**: Application Binary Interface. The implementation of an API such that its interactions can be realized by a digital computer. Typically includes memory layouts and binary representations, and is a function of the target platform and the build tools used to compile CPython. A particular build of CPython may have a different ABI (but identical API) to another build, which will impact compatibility with native extension builds but not sources.

**Stdlib**: Standard library. The set of components developed by the core team that build upon the Python language to provide building blocks and pre-written functionality.


## New terms

These terms are introduced briefly here and described in much greater detail below.

**API rings**: A way of dividing an API into subsets to specify stability guarantees. CPython extensions will care about rings and will allow themselves access to a particular ring to choose between deeper integration or long-term stability. Allowing access to one ring implies access to all rings outside of that one, which are "more stable" or "more accessible" (imagine a castle containing the most private API subset with multiple ringed walls around it, each ring providing "more public" APIs). Rings are orthogonal to layers.

**API layers**: A way of dividing an API into subsets to specify interdependencies and coupling. Applications that embed CPython, and the CPython implementation itself, care about layers. Applications choose to adopt or implement a particular layer, which requires that all "lower" API layers must be made available, either by inclusion or implementation (imagine a building with foundations, where each layer of the building is only feasible because of the layers below it). Layers are orthogonal to rings.

**API member**: One element of an API, such as a function, a class, or an accesible data value. Every user of an API will use it by accessing individual members. API members should belong to exactly one API ring and exactly one API layer.

# Quick Overview

For context as you continue reading, these are the API **rings** provided by CPython:

* Python ring (equivalent of the Python language)
* CPython ring (CPython-specific APIs)
* Internal ring (intended for internal use only)

These are the API **layers** provided by CPython:

* Optional stdlib layer (dependencies that must be explicitly required)
* Required stdlib layer (dependencies that can be assumed)
* Platform interaction layer (ability to interact with the platform and user)
* Core layer ("pure" mode with no dependency on target runtime)
* Platform adaption layer (isolates core layer from the target runtime)

(Reminder that this document does not reflect the current state of CPython, but is both aspirational and defining terms for the purposes of discussion. This is not a proposal to remove anything from the standard distribution!)


## Stability Guarantees Quick Overview

For context as you continue reading, these are half-sentence summaries of the stability guarantees made by CPython for each ring:

* Python ring APIs change only at `x` version number changes
* CPython ring APIs change only at `x.y` version number changes
* Internal ring APIs may change at any release


## Availability Guarantees Quick Overview

For context as you continue reading, these are half-sentence summaries of the availability guarantees made by CPython for each layer:

* Platform adaptation layer will always be present, but cannot be introspected (it is a "black box")
* Core layer will always be available
* Platform interaction layer may be present, but can only be accessed using APIs in the core layer
* Required stdlib layer is available when any of the optional stdlib layer is present
* Optional stdlib layer is made up of multiple components, any of which being present implies that the entire required stdlib layer and core layer are present


# API Rings

Rings delineate a user-visible API into functional groups, becoming progressively more "private" as you approach the innermost ring. Users of the API will select a ring to target, and will then only access that part of the API available in that and outer rings. This enables the API developer to provide specific stability guarantees for each ring.

Each member (function, data, etc.) that makes up part of the overall API should be defined as belonging to exactly one ring.

CPython provides three API rings, listed here from outermost to innermost:

* Python ring
* CPython ring
* Internal ring

An extension that targets the Python ring does not have access to the CPython or Internal rings. Likewise, an extension that targets the CPython ring does not have access to the Internal ring, but may use the Python ring.

All Python implementations should provide an equivalent Python ring of API (though not necessarily a directly compatible ABI). CPython explicitly supports extensions that target the CPython ring, and the Internal ring is available but unsupported.

There are no dependency restrictions between rings. The implementation of an API in the Python ring may depend on members of the CPython and Internal rings (and probably should, as it isolates users from changes to those parts of the API).


## Python API ring

The Python ring provides functionality that should be equivalent across all Python implementations - in essence, the Python language itself defines this ring.

CPython's implementation of the Python API (in C) allows native code to interact with Python objects as if it were written in Python. The Python API supports duck-typing and should correctly handle the substitution of alternative types.

> CPython example: `PyObject_GetItem` is part of the Python ring while `PyDict_GetItem` is in the CPython ring.

Compatibility requirements for the Python API match the language. Specifically, code relying on the Python API should only break or change behaviour if the equivalent code written in Python would also break or change behaviour.

For CPython, `#include "Python.h"` should only provide access to the Python ring. Accessing any other rings should produce a compile error.


## CPython API ring

The CPython ring provides functionality that is specific to CPython. Extensions that opt in to the CPython ring are inherently tied to the version of CPython they now target, but also have access to API members that are specific to CPython.

Members in the CPython ring may require the caller to be using C or be able to provide C structures allocated in memory.

In general, most applications that embed CPython will use the CPython ring, as will most extensions that are included with CPython.

> CPython example: the `PyCapsule` type belongs in the CPython ring (that is, other implementations are not required to provide this particular way to smuggle C pointers through Python objects)

> CPython example: `PyType_FromSpec` belongs in the CPython ring. (The equivalent in the Python ring would be to call the `type` object, while the equivalent in the internal ring would be to define a static `PyTypeObject`.)

> CPython example: `sys.prefix`, `exec_prefix` and related members belong in the CPython ring. (There is nothing inherent about Python that requires anything be installed.)

> CPython example: `sys._getframe` and `sys._xoptions` belong in the CPython ring.

Compatibility requirements for the CPython API match the CPython `x.y` version. Specifically, code relying on the CPython API should only break or change behaviour when the first or second version fields change.

For CPython, `#include "cpython/<header>.h"` provides access to API members in the CPython ring.


## Internal API ring

The Internal ring provides functionality that is used to implement CPython. Extensions that opt in to the Internal ring may need to rebuild for every CPython build.

In general, most of the Required stdlib layer will use the Internal ring.

> CPython example: `_PyRuntime` belongs in the Internal ring.

For CPython, `#include "internal/<header>.h"` provides access to API members in the Internal ring.


# API Layers

Layers delineate coupling boundaries between subsets of an API. The presence or absence of layers determines which part of the API a user has available, which depends on the implementer being consistent in the use of layers. Each API member is defined as belonging to exactly one layer.

As a rule, when "higher" layers are available, all "lower" layers are also available. This implies that the implementation of an API member in a particular layer can only assume the presence of its own layer and those below it.

CPython provides five API layers, listed here from top to bottom:

* Optional stdlib layer
* Required stdlib layer
* Platform interaction layer
* Core layer
* Platform adaptation layer

An application embedding Python targets one layer and all those below it, which affects the functionality available in Python.

Higher layers may depend on the APIs provided by lower layers, but not the other way around. In general, layers should aim to maximise interaction with the next layer down and avoid skipping it, but this is not a strict requirement.

The platform interaction layer is slightly different: it only interacts with the core layer. API elements in the other layers should use the interfaces provided by the core layer to access functionality of the target runtime. For example, the core layer provides the `open()` builtin, but defers the actual work to the platform interaction layer, which provides the resultant stream object.

Elements belonging to the optional stdlib layer can take their own dependencies on the target runtime. For example, socket support can be provided by an optional module and does not need to exist in the platform interaction layer.

Lower layers are required to maintain backwards compatibility more strictly than the layers above them.

Except for the optional stdlib layer, layers are "all or nothing". If any part is required, then the entire layer must be present.

Standard Python distributions (that is, anything that may be launched with the `python` command) will depend upon most components in the Optional stdlib layer, and hence will require _everything_ from the Required stdlib layer and below. Only embedders and potentially deployment tools will use reduced layers.

(Reminder: this document is not trying to be a reference for the current state of CPython, but as aspiration for a more organized architecture.)


## Platform adaptation layer

This layer provides the CPython implementation of platform-specific adapters to support the core layer.

* Memory allocation/deallocation
* Thread local storage
* Cryptographic random number generation
* Debug output

Notably, the platform adaptation layer does not enable file-system access or any text encoding abilities. This is because they are not required for scenarios where an embedding application can satisfy all imports with its own builtin modules and provide all code as in-memory strings (see the section on the core layer).

The debug output is the equivalent of the temporary printer that CPython uses during initialization. Depending on the application, it may redirect it to another form of output besides a console.


## Core layer

This layer is the core language and evaluation engine. By adopting this layer and a suitable platform adaptation layer, an application can provide basic Python execution of code provided as in-memory strings.

Examples of current components that fit into the core layer:

* Code compilation and interpreter loop
* Abstract and concrete object interfaces (`PyObject_*()` methods, etc.)
* Most built-in types and functions (str, int, list, dict, etc.)
* read-only members of the sys module
* import of builtin modules

Important but potentially non-obvious implications of relying only on the core layer:

* File system and standard streams are part of the platform interaction layer, which leaves `open` and `sys.stdout` (among others) without a default implementation. An application that wants to support these without adding more layers needs to provide its own implementations
* The core layer only exposes UTF-8 APIs. Encoding and decoding for the current platform requires the platform interaction layer, while arbitrary encoding and decoding requires the optional stdlib layer.
* Imports in the core layer are satisfied by a "blind" callback. The required stdlib layer provides `importlib` and hence support for frozen, bytecode and natively-encoded source imports, while the optional stdlib layer is required for arbitrary encodings in source files
* The dependency of `str.encode` and `bytes.decode` on the codec registry (part of the required stdlib API) is an exception to normal layering. For a configuration that omits the required stdlib, these functions will fail.
* The `async` and `await` keywords belong to the core layer, while `asyncio` is in the optional stdlib layer.


## Platform interaction layer

This layer provides most of what is today known as CPython (at least on supported platforms). By adopting this layer, Python can be used in ways that interact with the platform, including its user-facing and filesystem-facing mechanisms.

* File system access
* Standard input/output streams
* `os` module implementations (essentially, `posixmodule.c`)
* `python` entry point with command-line options

Important but potentially non-obvious implications of the platform interaction layer:

* The definition of a platform can vary wildly, and may not resemble a traditional "keyboard and file system" platform. By defining this as its own layer, it becomes easier for a new platform to substitute different implementations without suffering unbounded compatibility requirements.
* The `open` builtin is in the core layer, but exposes a hook that must be provided by this layer in order to work
* File system access generally requires text encodings, but the full set of codecs are in the optional stdlib layer. To fully separate these layers, an implementation of the current file system encoding would be required in this layer. (But arbitrarily encoding/decoding the _contents_ of a file may require higher layers.)
* Importing from source code may also require arbitrary encodings, but imports that can be fully satisfied without this are provided here (e.g. native extension modules, precompiled bytecode, frozen modules, natively encoded source files)
* As is currently the case, the API of the `os` implementation modules are not consistent between platforms.


## Required stdlib layer

This layer provides common APIs for interactions between other modules. All components in the optional stdlib layer may assume that everything in this layer is present.

* standard ABCs
* `importlib` package
* `os` module
* codec registry (initially empty)
* compiler services (e.g. `copy`, `functools`, `traceback`)
* standard interop types (e.g. `pathlib`, `enum`, `dataclasses`)

Important but potentially non-obvious implications of the required stdlib layer:

* `os` module is primarily implemented by the builtin `nt` or `posix` module provided by the platform interaction layer
* the codec registry consists of the API elements for registrering and looking up codecs, but does not actually include any codecs. This allows dependents of the required stdlib layer to reliably attempt to retrieve codecs at runtime, without forcing all codec definitions into this layer.
* `importlib` implements a hook from the core layer (to provide the imported module), uses the platform interaction layer (via the `os` module), and may use text codecs provided by the optional stdlib layer (via the codec registry in this layer).


## Optional stdlib layer

This layer provides modules that fundamentally stand alone. None of the lower levels may depend on these components being present, and components in this layer should explicitly declare dependencies on others in the same layer.

This layer is valuable for embedders and distributors that want to omit certain functionality. For example, omitting `socket` should be possible when that functionality is not required, as it is in the Optional stdlib layer, and omitting it should only affect those components in the optional stdlib layer that have explicitly required it.

* platform-independent algorithms (e.g. `itertools`, `statistics`)
* application-specific functionality (e.g. `email`, `socket`, `ftplib`, `ssl`)
* additional compiler services (e.g. `ast`)
* text codecs (e.g. `base64`, `codecs`, `encodings`)
* Python-level FFI (e.g. `ctypes`)
* tools (e.g. ``idlelib``, ``pynche``, ``distutils``, ``msilib``)
* configuration/information (e.g. ``site``, ``sysconfig``, ``platform``)

Important but potentially non-obvious implications of the optional stdlib layer:

* Components in the optional stdlib layer could be independently versioned, packaged, and/or distributed
* Interactions with the target runtime are independent of the platform
