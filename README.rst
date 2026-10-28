.. image:: https://img.shields.io/badge/sqr--120-lsst.io-brightgreen.svg
   :target: https://sqr-120.lsst.io
.. image:: https://github.com/lsst-sqre/sqr-120/workflows/CI/badge.svg
   :target: https://github.com/lsst-sqre/sqr-120/actions/

################################################
GitHub Actions OIDC token exchange in Gafaelfawr
################################################

SQR-120
=======

This technote proposes a design for authenticating GitHub Actions workflows to Phalanx-deployed services without long-lived secrets. A workflow presents the OpenID Connect (OIDC) ID token that GitHub issues to its job, and Gafaelfawr exchanges it for a short-lived Gafaelfawr token whose scopes and lifetime are set by a registry of trusted GitHub projects.

**Links:**

- Publication URL: https://sqr-120.lsst.io
- Alternative editions: https://sqr-120.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-120
- Build system: https://github.com/lsst-sqre/sqr-120/actions/


Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

   git clone https://github.com/lsst-sqre/sqr-120
   cd sqr-120
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://sqr-120.lsst.io whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sqr-120.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
