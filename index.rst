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

OS X 10.11 (El Capitan) is not compatible with any version of the LSST
software stack, including Summer 2015. This is due to the
new `System Integrity Protection
<https://developer.apple.com/library/prerelease/ios/documentation/Security/Conceptual/System_Integrity_Protection_Guide/System_Integrity_Protection_Guide.pdf>`_ (SIP)
feature. As well as preventing anyone from modifying system files,
protected binaries no longer inherit linker environment variables. In
particular :envvar:`DYLD_LIBRARY_PATH` is ignored. The stack and EUPS
completely rely on this environment variable and without it any
packages using C++ will not import.

A normal stack build of Summer 2015 fails almost immediately in the
``base`` package as that is the first that attempts to import C++
library code into python.

:abbr:`SIP (System Integrity Protection)` only affects Apple-supplied binaries. For the stack the issue is
that Python scripts and :command:`scons` are always run with a shebang (``#!``)
line of ``#! /usr/bin/env python``. Since :command:`env` is in :file:`/usr/bin` it is
covered by :abbr:`SIP (System Integrity Protection)` protections such that the library load path environment
variable is stripped before being executed. :command:`scons` is executed via
:command:`env` and runs the tests in a subprocess which will inherit a
stripped environment and will therefore fail. Furthermore, executable
scripts in the :file:`bin` directory will also have the environment
stripped if those scripts are executed via :command:`env`, and will therefore
fail to load C++ python modules.

Changes to the Stack
====================

The following changes were required to enable the LSST software to
build on El Capitan:

1. Modify EUPS table files to introduce a new environment variable,
   :envvar:`LSST_LIBRARY_PATH`, that looks identical to :envvar:`DYLD_LIBRARY_PATH`
   but which is not intercepted by :abbr:`SIP (System Integrity Protection)`. This variable is used inside
   :command:`scons` to ensure that all tests are executed with the correct
   environment enabled (:command:`scons` launches tests as sub processes).

2. :command:`/usr/bin/env` can no longer be used to run scripts from the
   command line. The shebang line must point to an explicit executable
   and that executable can not be in ``/usr/bin`` or ``/bin``. For
   Python scripts the shebang must point to a user-installed Python
   binary. To allow the rewriting of the shebang to occur a new
   :command:`scons` build target has been created, ``shebang``, that will
   copy files from a :file:`bin.src` directory to a :file:`bin` directory,
   modifying them during the copy. The rewriting does not happen on
   all platforms (although that is not guaranteed behavior for the
   future) and only files requiring rewrites should be placed in that
   directory.

The reason for the new environment variable specifically for running
tests is that it is difficult to ensure that the build is being
triggered with every parent process being correctly configured to pass
through the library path. At the very least we would have to fix
:command:`eups`, :command:`scons` and :command:`lsstsw` and even so
any shell scripts that people may use to trigger builds will also have
their environment stripped.

One additional complication on El Capitan is that Apple no longer
distributes the OpenSSL include files. Apple deprecated the use of
OpenSSL in OS X 10.7 (Lion) and removed the include files in El Capitan
(the libraries remain for binary compatibility). The
``activemqcpp`` and ``libevent`` packages were modified to
disable the use of SSL on OS X. [#f1]_

At the time of writing ``lsst_distrib`` builds correctly on El Capitan.

One other approach was considered and that was to copy
:command:`/usr/bin/env` to a new location and change every script to
use the new :command:`env`. This would have worked because the copied
:command:`env` would no longer be susceptible to :abbr:`SIP (System Integrity Protection)` restrictions. The
consensus was that this solution of a new :command:`env` did not feel
acceptable and would require too many edge cases in the documentation.


Porting to El Capitan
=====================

For developers the following must be remembered when modifying packages:

1. Ensure that :envvar:`LSST_LIBRARY_PATH` appears wherever
   :envvar:`DYLD_LIBRARY_PATH` appears in a table file.

2. Python scripts should be placed in the :file:`bin.src` directory and not
   the :file:`bin` directory. A suitable :file:`SConscript` file is shown at the end
   of this document and can also be found in the `package template repository <https://github.com/lsst/templates>`_.

3. People can no longer build or use the stack with the system Python.

4. Executable shell scripts should ensure they run :command:`setup` rather than
   relying on the setup of the parent shell. This is because
   :envvar:`DYLD_LIBRARY_PATH` will no longer be guaranteed to be set in the
   subshell. For an explicit discussion of this see :ref:`sip-examples`.

5. If a package requires OpenSSL, consider supporting both OpenSSL and
   Apple CommonCrypto. Otherwise OpenSSL may have to be made an explicit
   prerequisite on OS X.


Remaining Issues
================

The changes to allow tests to correctly inherit the environment only
affect packages built using ``sconsUtils``. Two packages are known not
to work on El Capitan:

1. ``partition`` uses ``sconsUtils`` in a non-standard way such that
   most of the targets are hand-crafted. The test target does not use
   the ``sconsUtils`` test framework so all the tests fail.

2. ``qserv`` uses a bespoke :command:`scons` configuration system that may
   need to be taught how to inherit :envvar:`LSST_LIBRARY_PATH` for the test
   environment. Additionally ``qserv`` uses OpenSSL when calculating
   digests and these will have to be ported to CommonCrypto.


Relevant JIRA Tickets
=====================

* `DM-3200 <http://jira.lsstcorp.org/browse/DM-3200>`_ : Primary ticket for port to El Capitan.
* `DM-4327 <http://jira.lsstcorp.org/browse/DM-4327>`_ : Disable SSL on ``activemqcpp``.
* `DM-4334 <http://jira.lsstcorp.org/browse/DM-4334>`_ : Disable SSL on ``libevent``.
* `DM-3803 <http://jira.lsstcorp.org/browse/DM-3803>`_ : Discussion of deprecated SSL on OS X as used by Qserv.

.. _sip-examples:

Example SIP Behavior
=====================

The following code

.. code-block:: python

   #! /usr/bin/env python
   import os
   print(os.environ["DYLD_LIBRARY_PATH"])

generates a ``KeyError`` on El Capitan. Running it as
:samp:`python test.py` correctly prints the value of the environment variable.

Similarly shell scripts, which always tend to use shells from
:file:`/bin` or :file:`/usr/bin`, will therefore also lose
:envvar:`DYLD_LIBRARY_PATH`. This script:

.. code-block:: shell

   #!/bin/bash

   echo DYLD: $DYLD_LIBRARY_PATH
   echo LSST: $LSST_LIBRARY_PATH

will only result in values appearing from the second line.
One solution is to explicitly set the path at the start of the script:

.. code-block:: shell

   #!/bin/bash

   # On OS X El Capitan we need to pass through the library load path
   if [[ $(uname -s) = Darwin* ]]; then
       if [[ -z "$DYLD_LIBRARY_PATH" ]]; then
           export DYLD_LIBRARY_PATH=$LSST_LIBRARY_PATH
       fi
   fi

This approach is used in the `LSST stack demo
<https://github.com/lsst/lsst_dm_stack_demo/blob/master/bin/demo.sh>`_. [#f2]_
The alternative is to explicitly call :command:`setup` in the script to
ensure that the variables are set.

SConscript
==========

The following code can be used in the :file:`bin.src` directory to configure :command:`scons`:

.. code-block:: python

   from lsst.sconsUtils import scripts
   scripts.BasicSConscript.shebang()

.. rubric:: Footnotes

.. [#f1] The LSST stack does not use SSL capabilities in
         ``activemqcpp`` or ``libevent`` so there is no impact in
         removing SSL support in these packages.

.. [#f2] Interestingly, if the shebang is removed and replaced with a
         blank line, the environment is inherited without being
         filtered by the default POSIX shell.

.. envvar:: DYLD_LIBRARY_PATH

  OS X equivalent of ``LD_LIBRARY_PATH``. Specifies the search path
  for loading shared libraries.

.. envvar:: LSST_LIBRARY_PATH

 Equivalent to :envvar:`DYLD_LIBRARY_PATH` but set by EUPS and
 guaranteed to not be stripped by :abbr:`SIP (System Integrity Protection)` when sub-processes
 are launched.
