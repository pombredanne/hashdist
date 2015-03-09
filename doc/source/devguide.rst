Developers's guide
==================



Terminology
-----------

**Distribution**:
    An end-user software distribution that makes use of HashDist
    under the hood, e.g., python-hpcmp

**Artifact**:
    The result of a build process, identified by a hash of the
    inputs to the build

**Profile**:
    A "prefix" directory structure ready for use through
    ``$PATH``, containing subdirectories ``bin``, ``lib``, and so on
    with all/some of the software one wants to use.

**Package**:
    Used in the loose sense; a program/library, e.g., NumPy, Python etc.; 
    what is **not** meant is a specific package format like ``.spkg``, ``.egg``
    and so on (which is left undefined in the bottom two HashDist layers)

Design principles
-----------------

 * Many small components with one-way dependencies, accessible through
   command line and as libraries (initially for Python, but in principle
   for more languages too).

 * Protocols between components are designed so that one can imagine
   turning individual components into server applications dealing with
   high loads.

 * However, implementations of those protocols are currently kept as
   simple as possible.

 * HashDist is a language-neutral solution; Python is the
   implementation language chosen but the core tools can (in theory)
   be rewritten in C or Ruby without the users noticing any difference

 * The components are accessible through a common ``hit`` command-line tool.
   This accesses both power-user low-level features and the higher-level
   "varnish", without implying any deep coupling between them (just like `git`).



Powerusers' guide, layer by layer
---------------------------------

HashDist consists of two (eventually perhaps three) layers. The idea
is to provide something useful for as many as possible. If a
distribution only uses the the *core layer* (or even only some of the
components within it) it can keep on mostly as before, but get a
performance boost from the caching aspect.  If the distribution wants
to buy into the greater HashDist vision, it can use the *profile
specification layer*.  Finally, for end-users, a final user-interface
layer is needed to make things friendly.  Here HashDist
will probably remain silent for some time, but some standards,
best practices and utilities may emerge. For now, a user interface ideas section is
included below.
