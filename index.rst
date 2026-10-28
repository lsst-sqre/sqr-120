################################################
GitHub Actions OIDC token exchange in Gafaelfawr
################################################

.. abstract::

   This technote proposes a design for authenticating GitHub Actions workflows to Phalanx-deployed services without long-lived secrets. A workflow presents the OpenID Connect (OIDC) ID token that GitHub issues to its job, and Gafaelfawr exchanges it for a short-lived Gafaelfawr token whose scopes and lifetime are set by a registry of trusted GitHub projects.

Add content here
================

See the `Documenteer documentation <https://documenteer.lsst.io/technotes/index.html>`_ for tips on how to write and configure your new technote.
