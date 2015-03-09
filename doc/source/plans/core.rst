HashDist core components
========================

Source store
------------

The idea of the source store is to help download source code from the net,
or "upload" files from the local disk. Items in the store are identified
with a cryptographic hash.

Items in the source store are only needed for a few hours while the
build takes place, and that may be the default configuration for
desktop users.  However, an advantage of keeping things around forever
is to always be able to redo an old build without relying on third
parties. Also, if many users share a source store on the local network
one can reduce the average time spent waiting for downloads. This is
an aspect that will be very different for different userbases.

Here's how to fetch some sources; the last line output (only one to ``stdout``)
is the resulting key::
    
    $ hit fetch http://python.org/ftp/python/2.7.3/Python-2.7.3.tar.bz2
    Downloading... progress indicator ...
    Done
    targz:mheIiqyFVX61qnGc53ZhR-uqVsY

    $ hit fetch git://github.com/numpy/numpy.git master
    Fetching ...
    Done
    git:c5ccca92c5f136833ad85614feb2aa4f5bd8b7c3

One can then unpack results later only by using the key::

    $ hit unpack targz:mheIiqyFVX61qnGc53ZhR-uqVsY src/python
    $ hit unpack git:c5ccca92c5f136833ad85614feb2aa4f5bd8b7c3 numpy

While ``targz:`` and ``git:`` is part of the key, this is simply to
indicate (mostly to humans) where the sources originally came from,
and not a requirement of the underlying source store implementation.

Note that git source trees are simply identified by their git commits,
not by an additional "repository name" or similar (the simplest
implementation of this is to pull all git objects into the same
local repository).

Re-fetching sources that are in cache already are not downloaded and
results in the same hash::

    $ hit fetch http://python.org/ftp/python/2.7.3/Python-2.7.3.tar.bz2
    targz:mheIiqyFVX61qnGc53ZhR-uqVsY

It's assumed that the content under the URL will not change (at least by
default). Downloading the same content from a different URL leads to
de-duplication and the registration of an additional URL for that
source.

Finally it's possible to store files from the local filesystem::

    $ hit fetch /home/dagss/code/fooproject
    dir:lkQYr_eQ13Sra5EYoXTg3c8msXs
    $ hit fetch -ttransient /home/dagss/code/buildfoo.sh
    file:tMwPj0cxhQVsA1pncZKcwCMgVbU

This simply copies files from local drive (mainly to make sure a copy
is preserved in pristine condition for inspection if a build fails).

**Tags:** The ``-ttransient`` option was used above to tag the
``buildfoo.sh`` script, meaning it's not likely to be important (or is
archived by other systems on local disk) so just keep it for a few
days. In general we have a system of arbitrary tags which one can then
make use of when configuring the GC.


Artifact builder
----------------
Assume in ``~/.hashdistrc``::

    [builder]
        store = ~/.hashdist/artifacts

The builder is responsible for executing builds, under the following
conditions:

 * The builder will *not* recurse to build a dependency. All
   software dependencies are assumed to have been built already
   (or be present on the host system).

 * All sources, scripts etc. has been downloaded/uploaded to the
   source store

Invoking a build::

    $ hit build < buildspec.json
    Not present in store, need to build. Follow log with
    
        tail -f /home/dagss/.hashdist/artifacts/numpy/2.6/_build0/build.log
    
    Done building, artifact name:
    numpy-2.6-Ymm0C_HRoH0HxNM9snC3lvcIkMo

    $ hit resolve numpy-2.6-Ymm0C_HRoH0HxNM9snC3lvcIkMo
    /home/dagss/.hashdist/artifacts/numpy/2.6/Ymm0C_HRoH0HxNM9snC3lvcIkMo

The build specification may look like this for a build::

    {
        "name" : "numpy",
        "version" : "1.6",
        "build-dependencies" : {
             "blas" : "ATLAS-3.10.0-gijMQibuq39SCBQfy5XoBeMSQKw",
             "gcc" : "gcc-4.6.3-A8x1ZV5ryXvVGLUwoeP2C01LtsY",
             "python" : "python-2.7-io-lizHjC4h8z5e2Q00Ag9xUvus",
             "bash" : "python-4.2.24.1-Z8GcCVzYOOH97n-ZC6qhfQhciCI"
         },
         "sources" : {
             "numpy" : "git:c5ccca92c5f136833ad85614feb2aa4f5bd8b7c3",
             "build.sh" : "file:gijMQibuq39SCBQfy5XoBeMSQKw",
         },
         "command" : ["$bash/bin/bash", "build.sh"],
         "env" : {
             "NUMPYLAPACKTYPE" : "ATLAS"
         },
         "env_nohash" : {
             "MAKEFLAGS" : "-j4",
         },
         "parameters" : [
             "this is free-form json", "build script can parse this information",
             "and use it as it wants"
         ],
         "parameters_nohash" : {
             "again-we-have" : ["custom", "json"]
         }
    }

What happens:

 1. A hash is computed of the contents of the build
    specification. This is simple since all dependencies are given in
    terms of their hash. Then, look up in the store; if found, we
    are done. (Dictionaries are supposed to be unordered and sorted
    prior to hashing.)

 #. Let's assume the artifact doesn't exist. A temporary directory is
    created for the build using ``mkdtemp`` (this is important so that
    there's no races if two people share the store and attempt the same build
    at the same time; the directory is moved atomically to its final location
    after the build).

 #. ``chdir`` to that directory, redirect all output to ``build.log``,
    and store the build spec as ``build.json``.  Unpack the sources
    listed using the equivalent of ``hit unpack``. The result in
    this case is a ``numpy`` subdirectory with the git checkout, and a
    ``build.sh`` script.

 #. Set environment variables (as documented elsewhere, TBD).  The
    keys in the `build-dependencies` section maps to environment variable names,
    so that ``$blas`` will contain ``ATLAS-3.10.0-gijMQibuq39SCBQfy5XoBeMSQKw``
    and ``$blaspath`` will contain
    ``../../ATLAS/3.10.0/gijMQibuq39SCBQfy5XoBeMSQKw``.
    This is the sole purpose of the keys in the `build-dependencies`
    section.  (Build scripts may also choose to parse ``build.json``
    too instead of relying on the environment.).

 #. Set up a sandbox environment. The sandboxing should be the topic
    of another section.

 #. Execute the given command. The command **must** start with a
    variable substitution of one of the dependencies listed, unless it
    is ``hit``.  (The bootstrapping problem this creates should be
    treated in another section.)


Build policy
''''''''''''

It's impossible to control everything, and one needs to trust the builds
that are being run that they will produce the same output given the same
input. The ``hit build`` tool is supposed to be a useful part in bigger
stack, and that bigger stack is what needs to make the tradeoffs between
fidelity and practicality.

One example of this is the ``X_nohash`` options, which provide for
passing options that only controls *how* things are built, not *what*,
so that two builds with different such entries will have the same
artifact hash in the end. The builder neither encourages nor discourages
the use of these options; that decision can only be made by the larger
system considering a specific userbase.


Build environment and helpers
-----------------------------

A set of conventions and utilities are present to help build scripts.

Dependency injection
''''''''''''''''''''

Builds should never reach out and detect settings or software (except
for very special bootstrapping packages), they should always get the
artifacts of all dependencies injected through ``build.json``.

This is largely something we cannot enforce and where one relies on sane
use of the system.

(Nix builds its own toolchain in order to strictly enforce this, we
consider that a too high cost to pay.)

Temporary build profiles
''''''''''''''''''''''''

If one calls ``hit makebuildprofile build.json``, then
``build.json`` is parsed and a profile environment created containing
all build dependencies, whose path is then printed to standard
output. Thus one can hand a single set of paths to ones scripts
rather than one path per dependency.

This isn't necesarrily a recommended mode of working, but "practicality
beats purity". If it's equally easy to pass in all dependencies explicitly
to the configuration phase, then please do that.

Sandboxing
''''''''''

By setting ``LD_PRELOAD`` it is possible to override ``libc.so`` and
filter all filesystem calls in order to create a sandbox and make sure
that the build does not read from ``/usr`` (or anywhere outside the
HashDist store, except through symlinks), which would indicate that
the build reaches out to pull stuff it shouldn't. The Grand Unified
Builder (GUB) takes this approach.

We may provide a ``hit sandbox`` command to do this.  One may either
want to turn that one for debugging, or all the time. One may have to
create wrappers scripts around ``gcc`` etc. to set up sandboxing since
some build tools like waf and scons like to control all the
environment variables during the build.


Scheduler
---------

To do many build artifacts in the right order (and in parallel in the
right way), we should include a basic job scheduler for doing downloading
and building. Say, ``hit runtasks tasks.json``, with ``tasks.json`` thus::

    {
        "numpysources" : {
            "type" : "fetch",
            "url" : "git://github.com/numpy/numpy.git"
            "hash" : "git:9c5a9226e7d742e3549d4e53d07d53517096f123"
        },
        "numpy" : {
            "type" : "build",
            "ncores" : 1,
            "body" : {
                "name" : "numpy",
                "dependencies" : {
                    "blas" : "$atlas"
                    ...
                },
                "sources" : {
                    "numpy" : "git:9c5a9226e7d742e3549d4e53d07d53517096f123"
                }
            }
        }
        "atlas" : {
            "type" : "build",
            "exclusive" : true,
            ...
        }

Open question: Support the ``"$atlas"`` notation used above, or require
that the hash for the atlas build section is computed and use that? Probably
the latter?


Profile tools
-------------

A (software) "profile" is a directory structure ready for use through
``$PATH``, containing subdirectories ``bin``, ``lib``, and so on which
links *all* the software in a given software stack/profile.

Creating a profile is done by::
        
    hit makeprofile /home/dagss/profiles/myprofile numpy-2.6-Ymm0C_HRoH0HxNM9snC3lvcIkMo ...

The command takes a list of artifacts, and reads ``install.json`` in
each one and use the information to generate the profile.  While the
``install.json`` file is generated during the build process, the
Builder component has no direct knowledge of it, and we document it
below.

Profiles are used as follows::

    # Simply print environment changes needed for my current shell
    $ hit env /home/dagss/profiles/myprofile
    export PATH="/home/dagss/profiles/myprofile/bin:$PATH"

    # Start new shell of the default type with profile
    $ hit shell /home/dagss/profiles/myprofile

    # Import settings to my current shell
    $ source <(hit env /home/dagss/profiles/myprofile)

Of course, power users will put commands using these in their
``~/.bashrc`` or similar.

``install.json``
''''''''''''''''

The ``install.json`` file is located at the root of the build artifact
path, and should be generated (by packages meant to be used by the profile
component) as part of the build.

Packages have two main strategies for installing themselves into a
profile:

 * **Strongly recommended:** Do an in-place install during the build, and let the
   installation phase consist of setting up symlinks in the profile

 * Alternatively: Leave the build as a build-phase, and run the install at profile
   creation time

The reason for the strong recommendation is that as part of the build,
a lot of temporary build profiles may be created (``hit
makebuildprofile``).  Also, there's the question of disk
usage. However, distributions that are careful about constructing
builds with full dependency injection may more easily go for the
second option, in particular if the system is intended to create
non-artifact profiles (see below).

The recommended use of ``install.json`` is::

    {
        "runtime-dependencies" : {
            "python" : "python-2.7-io-lizHjC4h8z5e2Q00Ag9xUvus",
            "numpy" : "numpy-2.6-Ymm0C_HRoH0HxNM9snC3lvcIkMo"
        },
        "command" : ["hit", "install-artifact"],
        "profile-env-vars" : {
            "FOO_SOFT_TYPE" : "FROBNIFICATOR",
        },
        "parameters" : {
            "rules" : [
                ["symlink", "**"], # ant-style globs
                ["copy", "/bin/i-will-look-at-my-realpath-and-act-on-it"],
                # "/build.json", "/install.json" excluded by default
            ]
        }
    }

(In fact, ``python`` is one such binary that benefits from being
copied rather than symlinked.)  However, one may also do the
discouraged version::
    
    {
        "runtime-dependencies" : {
            "python" : "python-2.7-io-lizHjC4h8z5e2Q00Ag9xUvus",
            "numpy" : "numpy-2.6-Ymm0C_HRoH0HxNM9snC3lvcIkMo"
        },
        "command" : ["$python/bin/python", "setup.py", "install", "--prefix=$profiletarget"],
        "parameters" : {
            "free-form" : ["json", "again", "; format is determined by command in question"]
        }
    }

More points:

 * The `runtime-dependencies` are used during the ``hit makeprofile`` process
   to recursively include dependencies in the profile.

 * The `profile-env-vars` are exported in the ``hit env``. This
   happens through a ``profile.json`` that is written to the profile
   directory by ``hit makeprofile``. This can be used to, e.g., set up
   ``PYTHONPATH`` to point directly to artifacts rather than
   symlinking them into the final profile.

 * ``install.json`` does not need to be hashed at any point.


Artifact profiles vs. non-artifact profiles
'''''''''''''''''''''''''''''''''''''''''''

Usually, one will want to run ``hashdist makeprofile`` as part of a build, so that
the profile itself is cached::
    
    {
        "name" : "profile",
        "build-dependencies" : {
             "numpy" : "numpy-2.6-Ymm0C_HRoH0HxNM9snC3lvcIkMo",
         },
         "command" : ["hit", "makeprofile", "$numpy"],
    }

Then, one force-symlinks to the resulting profile::

    $ hit build < profile.json
    profile-Z8GcCVzYOOH97n-ZC6qhfQhciCI
    $ ln -sf $(hit resolve profile-Z8GcCVzYOOH97n-ZC6qhfQhciCI) /home/dagss/profiles/default

This allows atomic upgrades of a user's profile, and leaves the possibility of
instant rollback to the old profile.

However, it is possible to just create a profile directly.
This works especially well together with the ``--no-artifact-symlinks`` flag::
    
    $ hit makeprofile --no-artifact-symlinks /path/to/profile artifact-ids...

Then one gets a more traditional fully editable profile, at the cost
of some extra disk usage. One particular usage simply clones a profile
that has been built as an artifact::

    $ hit makeprofile --no-artifact-symlinks /path/to/profile profile-Z8GcCVzYOOH97n-ZC6qhfQhciCI

This works because ``hit makeprofile`` emits an ``install.json``
that repeats the process of creating itself (profile creation is
idempotent, sort of).


The shared profile manager
''''''''''''''''''''''''''

To use a profile located in the current directory,
``./myprofile`` must be used. Calling ``hit env myprofile`` instead
looks up a central list of profile nicknames. In ``~/.hitconfig``::

    [hashdist]
        profiles = ~/.hashdist/profiles # this is the default

...and following that, we find::

    $ ls -la ~/.hashdist/profiles
    myprofile -> /home/dagss/profiles/myprofile
    qsnake -> /home/dagss/opt/qsnake
    qsnake_previous -> ./artifacts/qsnakeprofile/0.2/io-lizHjC4h8z5e2Q00Ag9xUvus

(The paths in ``/home/dagss`` are likely further symlinks into
``~/.hashdist/artifacts`` too, but which artifact gets changed by the
distribution). Distributions are encouraged to make use of this
feature so that one can do::

    $ hit shell sage
    $ hit shell qsnake

...and so on. The intention is to slightly blur the line between different
distributions; software distributions simply become mechanisms to build profiles.

