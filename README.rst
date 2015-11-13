#############################################
DMTN-001 Porting the stack to OS X El Capitan
#############################################

Dealing with System Integrity Protection on OS X El Capitan

View this report at http://dmtn-001.lsst.codes



..
  Uncomment this section and modify the DOI strings to include a Zenodo DOI badge in the README
  .. image:: https://zenodo.org/badge/doi/10.5281/zenodo.#####.svg
     :target: http://dx.doi.org/10.5281/zenodo.#####

Build this Report
=================

You can clone this repository and build the report locally with `Sphinx`_

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-001
   cd dmtn-001
   pip install -r requirements.txt
   make html

The built report is located at ``_build/html/index.html``.

Editing this Report
===================

You can edit the ``index.rst`` file, which is a reStructuredText document.
A good primer on reStructuredText is available at http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html

Remember that images and other types of assets should be stored in the ``_static/`` directory of this repository.
See ``_static/README.rst`` for more information.

The published report at http://dmtn-001.lsst.codes will be automatically rebuilt whenever you push your changes to the ``master`` branch on `GitHub <https://github.com/lsst-dm/dmtn-001`_.

Updating Metadata
=================

Report metadata is maintained in ``metadata.yaml``.
In this metadata you can edit the report's title, authors, publication date, etc..
``metadata.yaml`` is self-documenting with inline comments.

****

Copyright 2015 AURA/LSST

This work is licensed under the Creative Commons Attribution 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by/4.0/.

.. _Sphinx: http://sphinx-doc.org
