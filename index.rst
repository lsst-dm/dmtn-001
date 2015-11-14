..
  Content of technical report.

  See http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-report-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.


The Summer 2015 version of the LSST software stack does not work with
OS X El Capitan due to the new `System Integrity Protection
<https://developer.apple.com/library/prerelease/ios/documentation/Security/Conceptual/System_Integrity_Protection_Guide/System_Integrity_Protection_Guide.pdf>`_ (SIP)
feature. As well as preventing anyone from modifying system files,
protected binaries no longer inherit linker environment variables. In
particular ``DYLD_LIBRARY_PATH`` is ignored. The stack and EUPS
completely rely on this environment variable and without it any
packages using C++ will not import.

A normal stack build of Summer 2015 fails almost immediately in the
``base`` package as that is the first that attempts to import C++
library code into python.

SIP only affects Apple-supplied binaries. For the stack the issue is
that Python scripts and ``scons`` are always run with a shebang (``#!``)
line of ``#! /usr/bin/env python``. Since ``env`` is in ``/usr/bin`` it is
covered by SIP protections such that the library load path environment
variable is stripped before being executed. ``scons`` runs the tests and
so the tests do not have the correct environment; applications such as
``processCcd.py`` will also not function properly.

Changes to the Stack
====================

The following changes were required to enable the LSST software to
build on El Capitan:

1. Modify EUPS table files to introduce a new environment variable,
   ``LSST_LIBRARY_PATH``, that looks identical to ``DYLD_LIBRARY_PATH``
   but which is not intercepted by SIP. This variable is used inside
   `scons` to ensure that all tests are executed with the correct
   environment enabled (``scons`` launches tests as sub processes).

2. ``/usr/bin/env`` can no longer be used to run python scripts from the
   command line. The shebang line must point to an explicit python
   executable and that executable can not be the system python. To
   allow the rewriting of the shebang to occur a new ``scons`` build
   target has been that will copy files from a ``bin.src`` directory to
   a ``bin`` directory, modifying them during the copy. The rewriting
   does not happen on all platforms (although that is not guaranteed
   behavior for the future) and only files requiring rewrites should
   be placed in that directory.

The reason for the new environment variable specifically for running
tests is that it is difficult to ensure that the build is being
triggered with every parent process being correctly configured to pass
through the library path. At the very least we would have to fix
``eups``, ``scons`` and ``lsstsw`` and even so any shell scripts that
people may use to trigger builds will also have their environment
stripped.

One additional complication on El Capitan is that Apple no longer
distribute the OpenSSL include files. Apple have deprecated the use of
their OpenSSL since OS X 10.7 (Lion) and in El Capitan they finally
removed it (the libraries remain for binary compatibility). The
``activemqcpp`` and ``libevent`` packages had to be modified to
disable the use of SSL on OS X.

At the time of writing ``lsst_distrib`` builds correctly on El Capitan.

One other approach was considered and that was to copy
``/usr/bin/env`` to a new location and change every script to use the
new env. This would have worked because the copied ``env`` would no
longer be susceptible to SIP restrictions. The consensus was that this
solution of a new ``env`` did not feel acceptable and would require
too many edge cases in the documentation.


Porting to El Capitan
=====================

For developers the following must be remembered when modifying packages:

1. Ensure that ``LSST_LIBRARY_PATH`` appears wherever
   ``DYLD_LIBRARY_PATH`` appears in a table file.

2. Python scripts should be placed in the ``bin.src`` directory and not
   the ``bin`` directory. A suitable ``SConscript`` file is shown at the end
   of this document and can also be found in the `package template repository <https://github.com/lsst/templates>`_.

3. People can no longer build or use the stack with the system Python.

4. Executable shell scripts should ensure they run ``setup`` rather than
   relying on the setup of the parent shell.

5. If a package requires OpenSSL consider optionally using Apple
   CommonCrypto. Otherwise OpenSSL may have to be made an explicit
   pre-requisite on OS X.


Remaining Issues
================

The changes to allow tests to correctly inherit the environment only
affect packages built using ``sconsUtils``. Two packages are known not
to work on El Capitan:

1. ``partition`` uses ``sconsUtils`` in a non-standard way such that
   most of the targets are hand-crafted. The test target does not use
   the ``sconsUtils`` test framework so all the tests fail.

2. ``qserv`` uses a bespoke ``scons`` configuration system that may
   need to be taught how to inherit ``LSST_LIBRARY_PATH`` for the test
   environment. Additionally ``qserv`` uses OpenSSL when calculating
   digests and these will have to be ported to CommonCrypto.


Relevant JIRA Tickets
=====================

* `DM-3200 <http://jira.lsstcorp.org/browse/DM-3200>`_ : Primary ticket for port to El Capitan.
* `DM-4327 <http://jira.lsstcorp.org/browse/DM-4327>`_ : Disable SSL on ``activemqcpp``.
* `DM-4334 <http://jira.lsstcorp.org/browse/DM-4334>`_ : Disable SSL on ``libevent``.
* `DM-3803 <http://jira.lsstcorp.org/browse/DM-3803>`_ : Discussion of deprecated SSL on OS X as used by Qserv.

Example SIP Behavior
=====================

The following code

.. code-block:: python

   #! /usr/bin/env python
   import os
   print(os.environ["DYLD_LIBRARY_PATH"])

generates a ``KeyError`` on El Capitan. Running it as ``python
test.py`` correctly prints the value of the environment variable.

SConscript
==========

The following code can be used in the ``bin.src`` directory to configure ``scons``:

.. code-block:: python

   from lsst.sconsUtils import scripts
   scripts.BasicSConscript.shebang()

