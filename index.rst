################################################
GitHub Actions OIDC token exchange in Gafaelfawr
################################################

.. abstract::

   This technote proposes a design for authenticating GitHub Actions workflows to Phalanx-deployed services without long-lived secrets.
   A workflow presents the OpenID Connect (OIDC) ID token that GitHub issues to its job, and Gafaelfawr exchanges it for a short-lived Gafaelfawr token of a new ``github`` type whose scopes and lifetime are set by a registry of trusted GitHub projects.

Introduction
============

GitHub Actions workflows increasingly need to call services deployed on Phalanx.
Today the only way for a workflow to authenticate is a long-lived Gafaelfawr token created by an administrator and stored as a GitHub Actions secret.
Such secrets have well-known problems: they do not expire unless someone remembers to rotate them, they are not bound to the workflow that is supposed to use them, anyone with write access to the repository's secrets can exfiltrate them, and provisioning one for every new repository (or organization) is a manual chore for the SQuaRE team.

GitHub Actions offers an alternative.
Every job can request a signed OpenID Connect (OIDC) ID token from GitHub that attests which repository, workflow file, git ref, and GitHub environment the job is running in.
The token is short-lived and needs no secret to obtain.
A service that trusts GitHub's signing keys can verify the token and decide, based on its claims, whether to grant access.
This is the mechanism behind PyPI's trusted publishers, which lets a workflow publish a package without a stored API token.

This technote proposes a design in which `Gafaelfawr <https://gafaelfawr.lsst.io/>`__, the Rubin Science Platform's authentication and authorization service, accepts a GitHub Actions ID token and exchanges it for a short-lived Gafaelfawr token of a new ``github`` type, minted for a bot user.
Which repositories may do this, what scopes they receive, and how long the token lives are governed by a registry of GitHub projects stored in Gafaelfawr's database and managed through a new admin API.
Services behind Gafaelfawr's ingress need no changes: they see a bot user's token, which the ingress validates like any other Gafaelfawr token.

The motivating use cases are:

- Docverse projects uploading documentation builds from GitHub Actions.
- Documentation projects authenticating to Ook for its APIs, such as the link checker and the intersphinx cache.
- Front-end projects such as Squareone authenticating to the turborepo-cache-proxy to share Turborepo build caches.
- Notebook-based technote projects authenticating to Noteburst to execute their notebook in the Rubin Science Platform before publication.

SQuaRE already uses GitHub OIDC tokens in Ook to establish the provenance of documentation uploads, that is, to record which repository and workflow produced an upload.
This design goes further and uses the ID token for authentication and authorization.

The token exchange is implemented directly in Gafaelfawr rather than in a separate broker service.
Minting tokens for arbitrary bot users requires the ``admin:token`` scope, so a separate broker would itself have to hold a long-lived, highly privileged Gafaelfawr token, reproducing the problem this design is meant to remove.
Gafaelfawr also already owns the token store, the token change history used for auditing, the admin API conventions, and a deployment in every Phalanx environment, so the registry and the exchange endpoint fit naturally there.

The rest of this technote first explains how OIDC works in the GitHub Actions context, then describes the registry of GitHub projects in Gafaelfawr, the token exchange endpoint, how workflows use it, and the security considerations.
It closes with the decisions the team needs to make before implementation begins.

How OIDC works in GitHub Actions
================================

OpenID Connect (OIDC) is an identity layer on top of OAuth 2.0.
Its central artifact is the ID token: a JSON Web Token (JWT) signed by an identity provider that states who the bearer is and when the statement expires.
Anyone holding the provider's public signing keys can verify the token offline, without calling the provider and without sharing a secret with it.
The provider publishes those keys as a JSON Web Key Set (JWKS) at a URL advertised in its OpenID configuration document.

GitHub operates an OIDC identity provider for GitHub Actions at ``https://token.actions.githubusercontent.com``.
Its configuration document is at ``https://token.actions.githubusercontent.com/.well-known/openid-configuration``, which in turn points to the JWKS.
The identities it attests are not people but workflow jobs: an ID token says that a particular job of a particular workflow file in a particular repository, triggered by a particular event on a particular git ref, is running right now.

How a job obtains an ID token
-----------------------------

A job can request an ID token only if its workflow grants the ``id-token: write`` permission, which should be set at the job level rather than for the whole workflow:

.. code-block:: yaml

   permissions:
     id-token: write
     contents: read

When that permission is granted, the runner provides two environment variables, ``ACTIONS_ID_TOKEN_REQUEST_URL`` and ``ACTIONS_ID_TOKEN_REQUEST_TOKEN``.
The job makes an HTTP GET to the request URL, authenticated with the request token as a bearer token, and may append an ``audience`` query parameter naming the service the ID token is intended for.
GitHub responds with a JSON document whose ``value`` field is the signed JWT.
The ``actions/core`` toolkit wraps this in ``getIDToken(audience)`` for JavaScript actions, and the ``id`` package on PyPI does the same for Python.

No secret is configured anywhere in this process.
The workflow proves its identity with a token that GitHub mints on demand and that expires within minutes.
The only trust relationship is that the receiving service trusts GitHub's signing keys and GitHub's claims about which job is running.

The exchange sequence
---------------------

.. mermaid::

   sequenceDiagram
       participant Job as GitHub Actions job
       participant GitHub as GitHub OIDC provider
       participant Gafaelfawr
       participant Service as Phalanx service

       Job->>GitHub: 1. Request ID token (audience = environment base URL)
       GitHub-->>Job: 2. Signed JWT with claims
       Job->>Gafaelfawr: 3. POST JWT to the token exchange endpoint
       Gafaelfawr->>GitHub: 4. Fetch JWKS (cached between requests)
       GitHub-->>Gafaelfawr: JWKS
       Note over Gafaelfawr: 4. Verify signature, issuer, audience, validity window
       Note over Gafaelfawr: 5. Match claims against the GitHub project registry
       Gafaelfawr-->>Job: 6. Short-lived Gafaelfawr token of type github
       Job->>Service: 7. Call the service with the Gafaelfawr token
       Service-->>Job: Response (ingress validates the token as usual)

1. A job with ``id-token: write`` requests an ID token from GitHub's OIDC provider, setting ``audience`` to the base URL of the Phalanx environment it wants to talk to (for example ``https://roundtable.lsst.cloud``).
2. GitHub returns a JWT signed with its current key and carrying the claims described below.
3. The job posts the JWT to Gafaelfawr's token exchange endpoint in that environment.
4. Gafaelfawr fetches GitHub's JWKS (caching it between requests), verifies the signature, and checks that the issuer is GitHub, the audience is Gafaelfawr's own base URL, and the token is within its validity window.
5. Gafaelfawr looks up the token's repository, workflow, environment, and ref claims in its registry of GitHub projects.
   If a project matches, the registry entry gives the bot username, the permitted scopes, and the maximum lifetime.
6. Gafaelfawr mints a ``github`` token for that bot user and returns it to the job.
7. The job calls the Phalanx service (Ook, Docverse, and so on) with the Gafaelfawr token in the ``Authorization`` header.
   The service's ingress validates the token exactly as it validates any other Gafaelfawr token; the service itself never sees the GitHub ID token.

The ID token is used exactly once, for step 3, and is never sent to the Phalanx service.
This two-phase design mirrors PyPI's: the OIDC-specific logic is isolated in one exchange endpoint, and everything downstream keeps working with the token type it already understands.

Claims in the GitHub ID token
-----------------------------

The payload of a GitHub Actions ID token carries the standard JWT claims plus a set of GitHub-specific claims.
The table lists the claims that matter for this design, with example values for a push to the ``main`` branch of ``lsst-sqre/ook``.

.. list-table:: Claims used by the token exchange
   :header-rows: 1
   :widths: 20 40 40

   * - Claim
     - Example
     - Role in this design
   * - ``iss``
     - ``https://token.actions.githubusercontent.com``
     - Must equal GitHub's issuer
   * - ``aud``
     - ``https://roundtable.lsst.cloud``
     - Must equal Gafaelfawr's base URL
   * - ``sub``
     - ``repo:lsst-sqre/ook:ref:refs/heads/main``
     - Informational; matching uses the structured claims below
   * - ``repository``
     - ``lsst-sqre/ook``
     - Registry lookup key
   * - ``repository_id``
     - ``<numeric repository ID>``
     - Pinned at registration to defeat renames and resurrection
   * - ``repository_owner``
     - ``lsst-sqre``
     - Informational
   * - ``repository_owner_id``
     - ``<numeric owner ID>``
     - Pinned at registration, as PyPI does
   * - ``workflow_ref``
     - ``lsst-sqre/ook/.github/workflows/ci.yaml@refs/heads/main``
     - Optional workflow restriction
   * - ``job_workflow_ref``
     - ``lsst-sqre/ook/.github/workflows/ci.yaml@refs/heads/main``
     - Same, for jobs that call a reusable workflow
   * - ``environment``
     - ``production``
     - Optional environment restriction; absent unless the job uses an environment
   * - ``ref``
     - ``refs/heads/main``
     - Optional ref restriction
   * - ``ref_type``
     - ``branch``
     - Informational
   * - ``event_name``
     - ``push``
     - Recorded for audit; may be restricted in future
   * - ``actor``, ``run_id``, ``run_attempt``, ``workflow_sha``
     - ``<GitHub login>``, ``<run ID>``, ``1``, ``<commit SHA>``
     - Recorded in the exchange audit log and metrics event
   * - ``exp``, ``iat``, ``nbf``, ``jti``
     - timestamps, unique ID
     - Validity window and replay protection

The ``sub`` claim is a string GitHub composes from other claims: ``repo:ORG/REPO:environment:NAME`` when the job uses an environment, ``repo:ORG/REPO:pull_request`` for pull request events, and ``repo:ORG/REPO:ref:REF`` otherwise.
Repositories created after July 15, 2026 use an immutable form that embeds numeric IDs, such as ``repo:octo-org@123456/octo-repo@456789:ref:refs/heads/main``, and the format can be customized per organization.
Because the composition rules vary, this design matches on the individual structured claims rather than parsing ``sub``.

Reusable workflows need particular attention because the ID token then carries two workflow paths.
``workflow_ref`` names the top-level workflow in the calling repository, the file that was triggered, such as ``lsst-sqre/sqr-062/.github/workflows/ci.yaml@refs/heads/main``.
``job_workflow_ref`` names the reusable workflow the job is executing, which may live in another repository, such as ``lsst-sqre/shared-workflows/.github/workflows/docs.yaml@refs/heads/main``.
For a job that does not call a reusable workflow, both claims name the same file.
This design matches on ``job_workflow_ref`` because it always identifies the YAML that is actually running the job's steps, which is what a workflow restriction should pin.
The ``repository`` claim, by contrast, always names the calling repository, never the one hosting the reusable workflow.
A registry entry for a repository that uses a shared workflow must therefore name the shared workflow's full path, as described under matching rules.

Comparison with PyPI trusted publishers
---------------------------------------

PyPI's trusted publishers feature is the best-known application of this pattern and the model for this design.
The table summarizes where the Gafaelfawr design follows PyPI and where it differs.

.. list-table:: PyPI trusted publishers compared with the Gafaelfawr design
   :header-rows: 1
   :widths: 18 41 41

   * - Aspect
     - PyPI trusted publishers
     - Gafaelfawr GitHub token exchange
   * - Audience
     - The fixed string ``pypi`` (``testpypi`` for TestPyPI), discoverable at ``/_/oidc/audience``
     - Gafaelfawr's base URL, one per Phalanx environment
   * - Who configures trust
     - Project owners, in PyPI's web UI, per project
     - Gafaelfawr token administrators, through an admin API, per project, each naming a bot user
   * - Matching claims
     - Repository owner, repository name, workflow filename, optional environment; ``repository_owner_id`` always checked
     - Repository (with pinned ``repository_id`` and ``repository_owner_id``), optional workflow, optional environment, optional ref
   * - Exchange endpoint
     - ``POST /_/oidc/mint-token`` with ``{"token": "<jwt>"}``
     - A new Gafaelfawr endpoint; two candidate shapes are described later
   * - Minted credential
     - A ``pypi-`` API token valid for 15 minutes, scoped to the matching projects
     - A Gafaelfawr token of type ``github`` for the configured bot user, with configured scopes and a lifetime up to the configured maximum
   * - Identity downstream
     - Upload permission on the project
     - An ordinary bot user, visible to services as the caller
   * - Secret scanning
     - ``pypi-`` prefix detectable by GitHub secret scanning
     - ``gt-`` prefix, as for every Gafaelfawr token

Two aspects of PyPI's model are worth copying deliberately.
First, PyPI stores the numeric ``repository_owner_id`` when a publisher is registered and compares it at mint time, so a deleted and re-created organization or user with the same name cannot inherit the trust.
This design pins both ``repository_owner_id`` and ``repository_id``.
Second, PyPI matches claims exactly rather than with patterns, which keeps the trust configuration auditable.
This design does the same for repository, workflow, and environment, and allows only a narrow prefix form for refs so that release tags can be covered.

The registry of GitHub projects
===============================

The registry is the trust configuration: it says which GitHub workflows may obtain Gafaelfawr tokens, as whom, with which scopes, and for how long.
It is stored in Gafaelfawr's SQL database and managed through a new admin API, so that adding a repository does not require a Phalanx change and deploy.
Every change to the registry is attributed to the administrator who made it.
Each Phalanx environment's Gafaelfawr has its own registry, so a repository that needs tokens in two environments, for example Roundtable for Ook and an RSP environment for Noteburst, is registered in both.

Domain model
------------

The unit of configuration is a GitHub project.
A project binds a set of claim constraints to a bot user and a token policy.
The bot user follows Gafaelfawr's existing convention for service identities, with a narrower prefix: its username begins with ``bot-gh-``, so that an identity minted from GitHub is recognizable wherever a username appears, and it exists implicitly as the username on the tokens minted for it, so no separate user record is created.
Gafaelfawr enforces the ``bot-`` prefix only for service tokens created through the admin token API, a path the exchange does not use, so the registry enforces the ``bot-gh-`` prefix itself when a project is created or modified.

.. list-table:: Fields of a GitHub project
   :header-rows: 1
   :widths: 18 12 10 60

   * - Field
     - Type
     - Required
     - Description
   * - ``id``
     - integer
     - generated
     - Primary key; the project's only identifier, used in admin API URLs, the exchange response, and audit logs
   * - ``description``
     - string
     - no
     - Free text for administrators
   * - ``username``
     - string
     - default
     - Bot username tokens are minted for; must begin with ``bot-gh-`` and be a valid Gafaelfawr username.
       Defaults to ``bot-gh-<owner>-<repo>`` derived from ``repository`` when omitted.
       Several projects may share one
   * - ``repository``
     - string
     - yes
     - GitHub repository as ``owner/name``, matched case-insensitively against the ``repository`` claim
   * - ``repository_id``
     - integer
     - pinned
     - Numeric GitHub ID of the repository, recorded at registration and compared against ``repository_id``
   * - ``repository_owner_id``
     - integer
     - pinned
     - Numeric GitHub ID of the owner, recorded at registration and compared against ``repository_owner_id``
   * - ``workflow``
     - string
     - no
     - Workflow file path, either relative to the repository root, such as ``.github/workflows/docs.yaml``, or a full path in another repository for reusable workflows; compared against the path part of ``job_workflow_ref``
   * - ``environment``
     - string
     - no
     - GitHub environment name, compared against the ``environment`` claim; when set, tokens without that claim never match
   * - ``ref``
     - string
     - no
     - Git ref, compared against the ``ref`` claim; an exact value such as ``refs/heads/main``, or a prefix ending in ``/`` such as ``refs/tags/``
   * - ``scopes``
     - list of strings
     - yes
     - Maximum scopes a minted token may carry; each must be a known scope and may not be ``admin:token`` or another reserved ``admin:`` scope
   * - ``max_lifetime``
     - duration
     - yes
     - Maximum lifetime of a minted token; defaults to 1 hour and is capped by a Gafaelfawr configuration setting
   * - ``enabled``
     - boolean
     - yes
     - Disabled projects are kept but never match; disabling a project revokes its outstanding tokens
   * - ``created``, ``modified``
     - timestamps
     - generated
     - Audit metadata
   * - ``created_by``, ``modified_by``
     - string
     - generated
     - Username of the administrator who made the change

The three pinned fields are what make the registry robust against GitHub-side changes.
A repository can be renamed or transferred, and an organization or user name can be deleted and later re-registered by someone else.
Because GitHub's numeric IDs never change or get reused, comparing them at exchange time means the trust follows the original repository and owner, not whoever currently holds the name.
This copies PyPI's defense against account resurrection and extends it to the repository itself.

The IDs are resolved when a project is created.
The admin API accepts the repository name, looks up its numeric IDs through GitHub's REST API (``GET /repos/{owner}/{repo}``), and stores them.
For public repositories this needs no credentials; for private repositories the administrator can supply the IDs explicitly in the request.
Whether Gafaelfawr should instead hold a GitHub credential for this lookup is listed under open questions.

A project may also carry the optional identity attributes that Gafaelfawr's admin token request accepts for service tokens: ``uid``, ``gid``, and ``groups``.
They matter only where a service resolves the caller's identity through Gafaelfawr's user-info routes and the bot user has no entry in the environment's identity source.
The services in the motivating use cases authorize on scopes alone, so most projects leave them unset.

Nothing makes ``username`` unique, and sharing a bot user is intended for one case: several projects for the same repository with different policies, such as a project that grants ``read:ook`` on any ref alongside one that grants ``write:docverse`` only on ``main``.
Both mint tokens for the same bot, and the service sees one identity for that repository.
Sharing a bot user across repositories is allowed but should be a deliberate choice.
A service sees only the username, never the GitHub claims, so if two repositories mint as the same bot the service cannot tell them apart or make per-project authorization decisions on the token alone; only the exchange audit log retains the distinction.
The natural grain is therefore one bot user per repository, with a shared bot reserved for a group of repositories that is one logical client.

To make that grain the easy default, ``username`` is optional when creating a project.
When omitted, the admin API derives ``bot-gh-<owner>-<repo>`` from ``repository``, lowercasing both parts and replacing any character that is not a letter or digit with a hyphen, with runs of hyphens collapsed, since GitHub names allow dots, underscores, and mixed case that Gafaelfawr usernames do not.
The derivation never includes the workflow, environment, or ref: those are policy, not identity, and may be edited later.
Because the join is ambiguous once hyphens are involved (``lsst-sqre`` with ``docs`` and ``lsst`` with ``sqre-docs`` both derive ``bot-gh-lsst-sqre-docs``), the API refuses to create a project whose derived username is already used by a project for a different repository and asks for an explicit ``username`` instead.
The response always echoes the resolved username.
Renaming a repository does not rename its bot user; identity should not move when a label does.

Matching rules
--------------

An incoming ID token matches a project when every constraint the project sets is satisfied:

- ``repository`` equals the project's repository (case-insensitively), and ``repository_id`` and ``repository_owner_id`` equal the pinned values.
- If ``workflow`` is set, the path part of ``job_workflow_ref`` (the text before ``@``) equals it.
  A value given relative to the repository root, such as ``.github/workflows/docs.yaml``, is compared as ``repository/workflow``; a value given as a full ``owner/repo/.github/workflows/file.yaml`` path is compared whole, which is how a reusable workflow hosted in another repository is pinned.
- If ``environment`` is set, the token carries an ``environment`` claim equal to it.
- If ``ref`` is set, the token's ``ref`` claim equals it, or, when the configured value ends in ``/``, starts with it.
- The project is enabled.

Unset constraints match anything, so a project with only a repository accepts every workflow, event, and branch of that repository, including pull requests.
Administrators are expected to set the narrowest constraints the use case allows.
A deploy-style project, for example, should pin the workflow and restrict the ref to ``refs/heads/main`` or to ``refs/tags/``.

Matching is exact rather than pattern-based so that reading a project's configuration is enough to know what it trusts.
The trailing-slash prefix for refs is the one deliberate exception, because release workflows run on tags whose names cannot be enumerated in advance.

Several projects may match the same token, for example one that constrains only the repository and another that also pins a workflow.
The exchange chooses the most specific match, defined as the project with the most constraints set.
If two matching projects tie on specificity, the exchange fails with an error that names both, since the token could legitimately be minted for two different bot users with two different policies.
The admin API refuses to create a project whose constraints are identical to an existing one, which removes the most common source of such ties.

Admin API
---------

The registry is managed through a resource collection at ``/auth/api/v1/github/projects``.
Every route requires the ``admin:token`` scope, matching the rule that only token administrators may create service tokens for arbitrary bot users.

.. list-table:: Admin API routes
   :header-rows: 1
   :widths: 45 55

   * - Method and path
     - Purpose
   * - ``GET /auth/api/v1/github/projects``
     - List projects; supports ``?repository=`` and ``?username=`` filters
   * - ``POST /auth/api/v1/github/projects``
     - Create a project; responds with ``201 Created`` and the stored project, including the resolved username and pinned IDs
   * - ``GET /auth/api/v1/github/projects/{id}``
     - Retrieve one project
   * - ``PATCH /auth/api/v1/github/projects/{id}``
     - Modify any field except ``repository_id`` and ``repository_owner_id``; changing ``repository`` re-resolves them, and setting ``enabled`` to false revokes the project's outstanding tokens
   * - ``DELETE /auth/api/v1/github/projects/{id}``
     - Delete a project and revoke its outstanding tokens; responds with ``204 No Content``

A request to create a project for Docverse uploads from the SQR-062 repository, restricted to the ``docs.yaml`` workflow on the default branch and on tags:

.. code-block:: json

   {
     "description": "Docverse uploads for SQR-062",
     "username": "bot-gh-docverse-sqr-062",
     "repository": "lsst-sqre/sqr-062",
     "workflow": ".github/workflows/docs.yaml",
     "ref": "refs/heads/main",
     "scopes": ["write:docverse"],
     "max_lifetime": "PT1H"
   }

The response echoes the project with the resolved username, ``id``, the pinned ``repository_id`` and ``repository_owner_id``, ``enabled`` defaulting to ``true``, and the audit fields.
Durations use ISO 8601 (``PT1H``, ``PT15M``) in the API and are stored as seconds.

Every token the exchange mints records the ID of the project that authorized it, so deleting a project, or disabling it by setting ``enabled`` to false, revokes all of that project's outstanding tokens in the same transaction.
Each revocation is written to the token change history with the acting administrator as the actor, and the number of tokens revoked is logged.
Changing a project's other fields, such as its scopes or constraints, does not affect tokens already minted; an administrator who tightens a policy and wants it to take effect at once disables and re-enables the project.
A single token can still be deleted through the existing ``/auth/api/v1/users/{username}/tokens`` routes.

Registry changes are recorded in a new ``github_project_history`` table in the same style as Gafaelfawr's token change history: each row stores the action (create, modify, delete), the acting administrator, the time, the client IP address, and the before and after values.
This gives the team the same audit trail for trust configuration that it has for tokens.

Configuration
-------------

The feature is enabled per Phalanx environment through Gafaelfawr's Helm values.
The proposed settings, under a new ``config.github.tokenExchange`` block, are ``enabled`` (default ``false``), ``audience`` (default: Gafaelfawr's base URL), ``maxLifetime`` (the ceiling for any project's ``max_lifetime``, default 6 hours to match the longest a GitHub-hosted job can run), and ``jwksCacheLifetime`` (how long GitHub's signing keys are cached, default 1 hour).
The issuer is fixed to ``https://token.actions.githubusercontent.com`` and is not configurable, since GitHub Enterprise Server is not in scope.

The token exchange endpoint
===========================

The exchange endpoint is the only new route a workflow calls.
It is unauthenticated in Gafaelfawr's usual sense: no Gafaelfawr token is presented, because the GitHub ID token is the credential.
It is served directly by Gafaelfawr, like the rest of ``/auth/api/v1``, and is enabled only when ``config.github.tokenExchange.enabled`` is true.

The endpoint lives at ``POST /auth/api/v1/github/token``.
Two shapes for its request and response are described below.
Both carry the same information and are processed identically; the choice between them is a decision for review and is listed under open questions.

Option A: OAuth 2.0 token exchange (RFC 8693)
---------------------------------------------

RFC 8693 defines a standard grammar for trading one security token for another.
The request is form-encoded, as for every OAuth 2.0 token endpoint:

.. code-block:: http

   POST /auth/api/v1/github/token HTTP/1.1
   Content-Type: application/x-www-form-urlencoded

   grant_type=urn:ietf:params:oauth:grant-type:token-exchange
   &subject_token=eyJhbGciOiJSUzI1NiIs...
   &subject_token_type=urn:ietf:params:oauth:token-type:id_token
   &scope=write:docverse

The ``scope`` parameter is optional and narrows the scopes to a subset of what the matching project allows.
RFC 8693 defines no parameter for the requested lifetime, so lifetime narrowing would need a non-standard parameter (for example ``lifetime=1800``) or would be unavailable in this shape.
A successful response follows RFC 6749 and RFC 8693:

.. code-block:: json

   {
     "access_token": "gt-vHj2M...",
     "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
     "token_type": "Bearer",
     "expires_in": 3600,
     "scope": "write:docverse"
   }

Errors use the OAuth 2.0 error document with HTTP 400: ``invalid_request`` for a malformed request, ``invalid_grant`` for an ID token that fails verification or matches no project, and ``invalid_scope`` for a scope the project does not allow.
Each carries an ``error_description`` with a human-readable reason.

The case for this shape is familiarity.
Reviewers and future maintainers can read the RFC instead of a Gafaelfawr-specific description, the error vocabulary is already defined, and the pattern matches how cloud providers consume GitHub ID tokens.
The case against is that no client library is needed either way (a workflow issues one HTTP POST), the form encoding and URN-valued parameters are noisier than JSON for a bespoke endpoint, and Gafaelfawr's existing API is JSON throughout.

Option B: JSON request and response
-----------------------------------

The second shape follows PyPI's mint-token endpoint and Gafaelfawr's own API conventions:

.. code-block:: json

   {
     "token": "eyJhbGciOiJSUzI1NiIs...",
     "scopes": ["write:docverse"],
     "lifetime": "PT30M"
   }

``scopes`` and ``lifetime`` are optional.
When omitted, the token receives the project's full scope list and maximum lifetime.
The response is ``201 Created``:

.. code-block:: json

   {
     "token": "gt-vHj2M...",
     "username": "bot-gh-docverse-sqr-062",
     "scopes": ["write:docverse"],
     "expires": "2026-10-28T19:03:11Z",
     "project": 14
   }

Errors use Gafaelfawr's existing error format, a ``detail`` list whose entries carry a ``type`` such as ``invalid_token``, ``no_matching_project``, or ``invalid_scopes``, with HTTP 400 for malformed requests, 401 for an ID token that fails verification, and 403 for a verified token that matches no enabled project or requests scopes it is not allowed.

The case for this shape is consistency with the rest of Gafaelfawr's API and with PyPI, the model most contributors already know; the response can also carry useful context such as the bot username and the ID of the matched project.
The case against is that it is one more bespoke token API.

Processing
----------

Whichever shape is chosen, the endpoint processes a request in the following steps.
Each failure is logged with the reason and the claims that could be read, so that a workflow author can be helped without seeing Gafaelfawr's logs.

1. Decode the JWT header and read ``kid`` and ``alg``.
   Reject anything but an asymmetric algorithm GitHub uses (``RS256``).
2. Load GitHub's JWKS from the URL in its OpenID configuration document, from a cache with the configured lifetime.
   If ``kid`` is not in the cached set, refresh the cache once before failing, since GitHub rotates keys.
3. Verify the signature, then check that ``iss`` is ``https://token.actions.githubusercontent.com``, that ``aud`` equals the configured audience, and that the current time is within ``nbf`` and ``exp``, allowing a small clock skew.
4. Read the claims the registry needs: ``repository``, ``repository_id``, ``repository_owner_id``, ``job_workflow_ref``, ``ref``, and, if present, ``environment``.
   A token missing any required claim is rejected.
5. Find the enabled projects for that repository and evaluate the matching rules, selecting the most specific match.
   No match, or an ambiguous match, is a rejection.
6. Determine the token's scopes and lifetime.
   Requested scopes must be a subset of the project's scopes; a request for a scope outside that set is rejected rather than silently narrowed, so that a misconfigured workflow fails visibly.
   The lifetime is the requested lifetime if given and no longer than the project's ``max_lifetime``, otherwise the maximum.
7. Mint a token of type ``github`` for the project's username with those scopes and that expiration, recording the project's ID on the token, and return it.

The ``github`` token type
-------------------------

The minted token is a Gafaelfawr token of a new type, ``github``, alongside the existing ``session``, ``user``, ``notebook``, ``internal``, ``service``, and ``oidc`` types.
A distinct type follows the precedent of the ``oidc`` type, which Gafaelfawr introduced for the access tokens issued by its OpenID Connect server because those tokens have different security semantics from the tokens they are derived from.
Tokens minted from a GitHub ID token differ from service tokens in the same way: they are created without an administrator, by an unattended remote system, on the strength of GitHub's attestation rather than a stored credential.
Recording that difference in the token type makes these tokens identifiable in the token change history, which can already be filtered by type, and in logs, and it leaves room to treat them differently later without changing the registry or the exchange.

In every other respect a ``github`` token behaves like a service token.
It carries the same scopes, the ingress validates it the same way, it appears in the token admin interfaces under its bot username and can be listed, inspected, and deleted there, and a service that receives one can delegate internal tokens from it as from any other token.
Like service and ``oidc`` tokens, it cannot be modified after creation.
It also cannot be created through the admin token API, whose request model accepts only the ``service`` and ``user`` types; the exchange endpoint is the only way to mint one.

Adding the type to Gafaelfawr requires a one-line database migration, since token types are stored as a PostgreSQL enum, and a token-service method modeled on the one that mints ``oidc`` tokens rather than on the admin token request path.
The token table gains a nullable column holding the ID of the GitHub project that authorized the token, indexed so that a project's outstanding tokens can be found and revoked when the project is deleted or disabled; the same column is added to the token change history so that a revocation can be traced to its project.
Both schema changes land in one migration.
The ``rubin.gafaelfawr`` client library defines its own subset of token types and gains ``github`` if its helpers inspect token information.

Audit trail
-----------

The token change history records the creation with the ``github`` token type, the project ID, the bot username as the actor, and the GitHub runner's address as the IP address.
Because the history can be filtered by type through the existing ``token_type`` query parameter on ``/auth/api/v1/history/token-changes``, every token the exchange has ever minted can be listed in one query, and the project ID separates the tokens of projects that share a bot user.
In addition, the exchange emits a structured log entry and a metrics event carrying the project ID, the bot username, the token key, and the ID token's ``repository``, ``job_workflow_ref``, ``ref``, ``environment``, ``event_name``, ``actor``, ``run_id``, ``run_attempt``, and ``jti``.
This is enough to answer, for any token, exactly which workflow run obtained it and who triggered that run.
It also preserves the provenance information that Ook currently derives from the ID token itself.

Using the exchange from a workflow
==================================

From a workflow author's point of view the exchange is two steps: obtain an ID token from GitHub with the right audience, and post it to Gafaelfawr.
The job must request the ``id-token: write`` permission, which is best granted at the job level so that other jobs in the workflow cannot mint tokens.
Using the JSON shape, a shell step looks like this:

.. code-block:: yaml

   jobs:
     upload:
       runs-on: ubuntu-latest
       permissions:
         id-token: write
         contents: read
       steps:
         - uses: actions/checkout@v5

         - name: Get a Gafaelfawr token
           id: gafaelfawr
           shell: bash
           run: |
             id_token=$(curl -sSf -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
               "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=https://roundtable.lsst.cloud" | jq -r .value)
             token=$(curl -sSf -X POST https://roundtable.lsst.cloud/auth/api/v1/github/token \
               -H "Content-Type: application/json" \
               -d "{\"token\": \"$id_token\", \"scopes\": [\"write:docverse\"]}" | jq -r .token)
             echo "::add-mask::$token"
             echo "token=$token" >> "$GITHUB_OUTPUT"

         - name: Upload to Docverse
           uses: lsst-sqre/docverse-upload@v1
           with:
             token: ${{ steps.gafaelfawr.outputs.token }}

The ``::add-mask::`` line keeps the minted token out of the job log.
Nothing in the repository's secrets is used.

Writing this shell by hand in every repository is error-prone, so the exchange should be packaged once.
The proposal is a composite GitHub Action in the ``lsst-sqre`` organization, provisionally ``lsst-sqre/gafaelfawr-token-action``, that takes the Phalanx environment URL and optional scopes and lifetime as inputs, performs both steps, masks the result, and exposes it as a ``token`` output.
SQuaRE's existing TypeScript actions that already talk to these services, such as ``docverse-upload``, can also perform the exchange internally using ``@actions/core``'s ``getIDToken()``, so that a workflow needs no separate step at all.
For Python clients, the ``rubin.gafaelfawr`` client library can gain a helper that performs the exchange when it detects the GitHub Actions environment variables.

Whichever packaging is used, the audience must be the exact base URL of the Phalanx environment, because Gafaelfawr in each environment accepts only its own audience.
A workflow that talks to services in two environments requests two ID tokens and performs two exchanges.

Security considerations
=======================

The trust boundary of this design is write access to a registered repository.
Anyone who can get a workflow to run in that repository on a matching workflow, environment, and ref can obtain a token with the project's scopes.
This is the same boundary PyPI accepts, and it is a better one than a stored secret, which anyone with access to the repository's settings can read and use from anywhere.
Administrators tighten the boundary with the registry constraints: pinning the workflow file, restricting the ref to the default branch or release tags, and, where review before use is wanted, requiring a GitHub environment, since environments can be configured with required reviewers and deployment branch rules.

Pull requests and forks
-----------------------

GitHub does not issue ID tokens to workflows triggered by pull requests from forks, and the ``repository`` claim always names the base repository, never the fork.
A ``pull_request`` run from a branch within the repository does get a token, with ``ref`` set to the merge ref and ``event_name`` set to ``pull_request``.
A project with no workflow or ref constraint therefore grants its scopes to pull-request builds of the same repository.
For read-only scopes such as link checking this is intended; for write scopes the ref should be constrained.
Workflows using ``pull_request_target`` run with the base repository's permissions against untrusted code and should never be registered with write scopes.
The implementation should verify the fork behaviour against current GitHub documentation.

Audience confusion
------------------

Each Phalanx environment's Gafaelfawr accepts only its own base URL as the audience, so an ID token obtained for one environment cannot be exchanged in another.
A workflow that mistakenly requests a generic audience fails at exchange rather than succeeding somewhere unintended.

Replay
------

A GitHub ID token is valid for minutes and the exchange is idempotent in effect: replaying one yields another token for the same bot user under the same policy, which the holder of the first token could already use.
Recording the ``jti`` of exchanged tokens in Redis until their ``exp`` would make each ID token single-use at low cost; whether this is worth doing is an open question.

Renames and resurrection
------------------------

Pinning ``repository_id`` and ``repository_owner_id`` means that transferring or renaming a repository, or re-creating a deleted organization under the same name, does not transfer trust.
An administrator who intentionally moves a repository updates the project, which re-resolves the IDs.

Reusable workflows
------------------

The ``job_workflow_ref`` claim names the reusable workflow's file when one is called, not the caller's workflow, so a project constraint on the workflow must name the file that is actually running.
For a reusable workflow hosted in another repository, the project's ``workflow`` field holds that repository's full path, as described under matching rules, while the ``repository`` constraint still refers to the calling repository.
This is a documentation point for workflow authors and a reason the exchange logs ``job_workflow_ref`` on every failure.

Scope limits
------------

The registry refuses ``admin:token`` and the other reserved ``admin:`` scopes, so no workflow can obtain token-administrator powers.
Gafaelfawr could additionally restrict registrable scopes to a configured allowlist per environment; this is listed as an open question.

Key handling
------------

Gafaelfawr needs outbound HTTPS to ``token.actions.githubusercontent.com`` to fetch the JWKS, which network policies must allow.
Keys are cached and refreshed on an unknown ``kid``, so rotation by GitHub is handled without a deploy.
Only ``RS256`` is accepted; a token whose header names any other algorithm is rejected before signature checking.

An unauthenticated endpoint
---------------------------

The exchange route accepts requests from anyone.
Its request body is bounded (an ID token is a few kilobytes), each request costs one signature verification and one database query, and failures are logged with the client address, so the exposure is comparable to Gafaelfawr's existing login routes.
Per-address rate limiting can be added if abuse is observed.

Lifetime and revocation
-----------------------

Minted tokens are capped by the project's ``max_lifetime`` and the environment-wide ceiling.
Deleting or disabling a project revokes every outstanding token minted for it, because each token records its project, so the registry is the one place an administrator needs to act to cut off a repository.
Short lifetimes, one hour by default, keep the exposure from a leaked token small in the time before anyone acts.

A distinct token type
---------------------

Because these tokens have their own type, Gafaelfawr could later apply restrictions to them that do not apply to service tokens, for example requiring an ingress to opt in to accepting ``github`` tokens.
This design does not propose such a restriction.
The alternative to the exchange is a service token stored as a repository secret and used by the same unattended workflow, so any restriction worth imposing on ``github`` tokens applies equally to service tokens held by automation, and the right fix is to tighten what a token without scopes can do rather than add another authorization mechanism.
The type keeps the option open at no cost.

.. _user-owned-projects:

Future work: user-owned projects
================================

The design above serves SQuaRE's own services, with trust configured by administrators and tokens minted for bot users.
The same exchange could let a Rubin Science Platform user register a repository themselves and have workflows in it obtain tokens under their own account, so that a GitHub Actions job can, for example, run a TAP query and render a figure for a paper draft.
Because the token would be the user's, the compute is charged against their quota and their data rights apply, exactly as if they had created the token by hand.
This section sketches what that extension requires so that the first phase does not preclude it; it is not part of the proposal under review.

A user-owned project differs from an admin-owned one in three ways.
It records an owner, the username of the user who registered it.
It mints a ``user`` token for that username rather than a ``github`` token for a bot, so the token appears in the owner's token list in the Gafaelfawr UI and can be revoked there, and it carries the owner's uid, gid, and groups like any other user token; whether such tokens should instead keep the ``github`` type, and lose that integration, is a question for that phase.
Its scopes are bounded by what the owner is entitled to rather than by what an administrator configured: the owner may choose any subset of the scopes their session holds when registering, mirroring the rule for user tokens, and at exchange time Gafaelfawr re-derives the owner's current scopes from group membership and intersects them with the project's scopes, so that a user removed from a data-rights group stops receiving tokens.

Registration would be self-service through routes under ``/auth/api/v1/users/{username}/github-projects``, authorized by the ``user:token`` scope, with a panel in the token UI beside the existing user-token form.
A user can only register projects that mint for themselves.
The project model needs a nullable ``owner`` column, with the token type derived from whether it is set, and a per-environment setting that enables user-owned projects at all.

That setting is needed because the exchange must look up a user without a login.
In environments whose identity source is LDAP, such as ``data.lsst.cloud`` and USDF, Gafaelfawr can fetch a user's groups by username on demand, which is the lookup the ``admin:userinfo`` scope already exercises.
In GitHub-authenticated environments such as Roundtable, group membership exists only inside session token data, so there is nothing to consult and user-owned projects cannot be offered.
Since science users live in the RSP environments, this restriction costs nothing for the motivating use case.

Two questions are new rather than inherited from the first phase.
First, several users may register the same repository, and the exchange cannot guess which one a workflow run means, so the exchange request gains an optional ``username`` parameter and a user-owned project matches on repository and owner together.
Second, a user can register a repository they do not control.
This harms nobody but themselves, but the converse must be stated plainly in the UI: registering a repository means that anyone who can run workflows in it can act as that user on the platform with the chosen scopes.
The defaults should therefore restrict the ref to the repository's default branch and keep the lifetime short, and the user's ability to see and revoke the resulting tokens is the backstop.
Verifying that the user controls the repository is hard to do well when the platform identity is not a GitHub identity, so trust at registration with a clear warning is the likely outcome.

To keep this phase additive, the first phase should include the nullable ``owner`` column in the project table and accept an optional ``username`` parameter on the exchange request, ignored when no user-owned projects exist.
Everything else, the user-facing routes, the UI, the group lookup at exchange time, and the environment setting, can be added later without changing the admin-owned flow.

Open questions and decisions for review
=======================================

The following decisions need to be made before implementation starts.
The first is the only one that changes the external interface; the rest can be settled during review or deferred with a recorded default.

- **Endpoint shape.**
  RFC 8693 token exchange (Shape A) or a JSON request and response (Shape B)?
  Shape A is standard but has no lifetime parameter; Shape B matches Gafaelfawr's API and PyPI.
- **Resolving pinned IDs.**
  Should the admin API look up ``repository_id`` and ``repository_owner_id`` through GitHub's public API, require administrators to supply them, or should Gafaelfawr hold a GitHub credential so private repositories can be resolved too?
- **Overlapping projects.**
  Is most-specific-match-wins with an error on ties acceptable, or should the admin API refuse any new project whose constraints overlap an existing one?
- **Event constraints.**
  Should a project be able to restrict ``event_name`` (for example to exclude ``pull_request``), or is the ref restriction sufficient?
- **Replay protection.**
  Record exchanged ``jti`` values in Redis to make each ID token single-use, or rely on the ID token's short validity?
- **Registrable scopes.**
  Beyond refusing ``admin:`` scopes, should each environment configure an allowlist of scopes that GitHub projects may be granted?
- **Ref matching.**
  Is the trailing-slash prefix form enough, or do any use cases need glob patterns on refs?
- **Ook's provenance.**
  Should Ook keep verifying ID tokens itself for provenance, or rely on the audit trail Gafaelfawr records at exchange time and accept only Gafaelfawr tokens?
- **Client packaging.**
  A standalone composite action, exchange built into individual SQuaRE actions (like `lsst-sqre/docverse-upload <https://github.com/lsst-sqre/docverse-upload>`__), a ``rubin.gafaelfawr`` helper, or all three?
- **Default lifetimes.**
  Are 1 hour per project and a 6 hour environment ceiling the right defaults?
- **Fork and pull-request behaviour.**
  Confirm against current GitHub documentation that pull requests from forks receive no ID token and how ``ref`` is set for ``pull_request`` runs, and confirm the 6 hour job limit assumed for the lifetime ceiling.
- **Multiple environments.**
  Registries are per environment.
  Is registering a repository in each environment acceptable operational overhead, or should Phalanx values seed or mirror projects across environments?
- **Room for user-owned projects.**
  Include the nullable ``owner`` column and the optional ``username`` exchange parameter in the first phase, as :ref:`user-owned-projects` section prposes, or defer both?

References
==========

- `OpenID Connect in GitHub Actions: reference <https://docs.github.com/en/actions/reference/security/oidc>`__: the ID token's claims, ``sub`` formats, and how a job requests a token.
- `About security hardening with OpenID Connect <https://docs.github.com/en/actions/concepts/security/openid-connect>`__: GitHub's conceptual overview.
- `PyPI trusted publishers: internals and technical details <https://docs.pypi.org/trusted-publishers/internals/>`__: the two-phase exchange and the ``repository_owner_id`` resurrection defense.
- `PyPI trusted publishers: using a publisher <https://docs.pypi.org/trusted-publishers/using-a-publisher/>`__: the ``/_/oidc/audience`` and ``/_/oidc/mint-token`` endpoints.
- `RFC 8693: OAuth 2.0 Token Exchange <https://www.rfc-editor.org/rfc/rfc8693>`__: the grammar for Shape A.
- `Gafaelfawr: service tokens <https://gafaelfawr.lsst.io/user-guide/service-tokens.html>`__: bot usernames and the ``GafaelfawrServiceToken`` resource.
- `Gafaelfawr: token administrators <https://gafaelfawr.lsst.io/user-guide/administrators.html>`__: the ``admin:token`` scope and the admin token API.
- `Gafaelfawr: scopes <https://gafaelfawr.lsst.io/dev/scopes.html>`__: reserved scopes and delegation rules.
- `Gafaelfawr: token types <https://gafaelfawr.lsst.io/dev/internals/gafaelfawr.models.enums.TokenType.html>`__: the existing token types, including the ``oidc`` type that is the precedent for the ``github`` type proposed here.
