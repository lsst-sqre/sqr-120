################################################
GitHub Actions OIDC token exchange in Gafaelfawr
################################################

.. abstract::

   This technote proposes a design for authenticating GitHub Actions workflows to Phalanx-deployed services without long-lived secrets.
   A workflow presents the OpenID Connect (OIDC) ID token that GitHub issues to its job, and Gafaelfawr exchanges it for a short-lived Gafaelfawr token of a new ``github`` type whose scopes and lifetime are set by a registry of trusted GitHub projects.
   Administrators manage the registry through an admin API, and a narrower provisioning scope lets Templatebot register projects for newly created technotes from administrator-defined templates.

Introduction
============

GitHub Actions workflows increasingly need to call services deployed on Phalanx.
Today the only way for a workflow to authenticate is a long-lived Gafaelfawr token created by an administrator and stored as a GitHub Actions secret.
Such secrets have well-known problems: they do not expire unless someone remembers to rotate them, they are not bound to the workflow that is supposed to use them, anyone who can run a workflow in the repository can exfiltrate them, and provisioning one for every new repository (or organization) is a manual chore for the SQuaRE team.

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
Once the exchange exists, Ook should move to it as well and accept only Gafaelfawr tokens, so that one mechanism serves both purposes.

The token exchange is implemented directly in Gafaelfawr rather than in a separate broker service.
Minting tokens for arbitrary bot users requires the ``admin:token`` scope, so a separate broker would itself have to hold a long-lived, highly privileged Gafaelfawr token, reproducing the problem this design is meant to remove.
Gafaelfawr also already owns the token store, the token change history used for auditing, the admin API conventions, and a deployment in every Phalanx environment, so the registry and the exchange endpoint fit naturally there.

The rest of this technote first explains how OIDC works in the GitHub Actions context, then describes the registry of GitHub projects in Gafaelfawr, the token exchange endpoint, how workflows use it, how Templatebot provisions projects for new technotes, and the security considerations.
It closes with a sketch of a possible extension to user-owned projects that is not part of this proposal.

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
     - ``POST /auth/api/v1/github/token`` using the OAuth 2.0 token exchange grammar of RFC 8693
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
Seeding or mirroring projects across environments from Phalanx values was considered and is not needed; registering a repository in each environment separately is acceptable overhead.

Domain model
------------

The unit of configuration is a GitHub project.
A project binds a set of claim constraints to a bot user and a token policy.
The bot user follows Gafaelfawr's existing convention for service identities, with a narrower prefix: its username begins with ``bot-gh-``, so that an identity minted from GitHub is recognizable wherever a username appears, and it exists implicitly as the username on the tokens minted for it, so no separate user record is created.
Gafaelfawr enforces the ``bot-`` prefix only for service tokens created through the admin token API, a path the exchange does not use, so the registry enforces the ``bot-gh-`` prefix itself when a project is created or modified.

In the table, the Required column distinguishes fields the administrator must supply (yes), may supply (no), may omit in favor of a derived value (default), and fields the administrator never supplies: generated fields are set by Gafaelfawr, and pinned fields are looked up from GitHub when the project is created and then compared against the ID token at every exchange.

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
   * - ``template``
     - string
     - generated
     - Name of the project template the project was provisioned from, or null for projects created directly through the admin API; see :ref:`provisioning`
   * - ``description``
     - string
     - no
     - Free text for administrators
   * - ``requested_by``
     - string
     - no
     - Opaque identifier of the person on whose behalf the project was created, as ``<source>:<identifier>``, such as ``slack:U024BE7LH`` or ``gafaelfawr:jsick``; recorded in the history and filterable, never interpreted
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
     - Username from the token that authenticated the request, whether an administrator's or a provisioner's; never taken from the request body

.. _pinned-ids:

The two pinned fields are what make the registry robust against GitHub-side changes.
A repository can be renamed or transferred, and an organization or user name can be deleted and later re-registered by someone else.
Because GitHub's numeric IDs never change or get reused, comparing them at exchange time means the trust follows the original repository and owner, not whoever currently holds the name.
This copies PyPI's defense against account resurrection and extends it to the repository itself.

The IDs are resolved when a project is created.
The admin API accepts the repository name, looks up its numeric IDs through GitHub's REST API (``GET /repos/{owner}/{repo}``) without credentials, and stores them.
Only public repositories can therefore be registered in the first phase.
Support for private repositories, either by accepting the IDs explicitly in the request or by giving Gafaelfawr a GitHub credential for the lookup, can be added later if a use case appears.
There is in principle a race between the lookup and a rename or transfer of the repository, but an administrator registering a project by hand is unlikely to lose it, and the response echoes the pinned IDs for the administrator to check.

A project may also carry the optional identity attributes that Gafaelfawr's admin token request accepts for service tokens: ``uid``, ``gid``, and ``groups``.
They matter only where a service resolves the caller's identity through Gafaelfawr's user-info routes and the bot user has no entry in the environment's identity source.
The services in the motivating use cases authorize on scopes alone, so most projects leave them unset.

Nothing makes ``username`` unique, and sharing a bot user is intended for one case: several projects for the same repository with different policies, such as a project that grants ``read:ook`` on pull-request refs (``refs/pull/``) alongside one that grants both ``read:ook`` and ``write:docverse`` on ``refs/heads/main``.
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

At most one project can match a given token, because the admin API refuses to create or modify a project whose constraints overlap those of an existing project for the same repository.
Two projects overlap when an ID token could satisfy both: for each of ``workflow``, ``environment``, and ``ref``, the constraint is unset in at least one of them, or the two values are equal, or one is a ref prefix of the other.
An administrator who wants a general policy and a narrower one for the same repository therefore expresses them as disjoint projects, for example one on ``refs/pull/`` and another on ``refs/heads/main``, rather than relying on the exchange to pick the more specific of two matches.
The rule costs some verbosity but preserves the property that a project's configuration alone says what it trusts; with overlapping projects, a narrower project would silently shadow a broader one.
The error names the conflicting project so that the administrator can adjust either.

Admin API
---------

The registry is managed through a resource collection at ``/auth/api/v1/github/projects``.
Every route requires the ``admin:token`` scope, matching the rule that only token administrators may create service tokens for arbitrary bot users.
A second collection of project templates, and a provisioning route that creates a project from a template under a narrower scope, are described in :ref:`provisioning`.

.. list-table:: Admin API routes
   :header-rows: 1
   :widths: 45 55

   * - Method and path
     - Purpose
   * - ``GET /auth/api/v1/github/projects``
     - List projects; supports ``?repository=``, ``?username=``, ``?template=``, and ``?requested_by=`` filters
   * - ``POST /auth/api/v1/github/projects``
     - Create a project; responds with ``201 Created`` and the stored project, including the resolved username and pinned IDs, or with ``409 Conflict`` when its constraints overlap an existing project's
   * - ``GET /auth/api/v1/github/projects/{id}``
     - Retrieve one project
   * - ``PATCH /auth/api/v1/github/projects/{id}``
     - Modify any field except ``repository_id`` and ``repository_owner_id``; changing ``repository`` re-resolves them, changing a constraint is checked for overlap as on creation, and setting ``enabled`` to false revokes the project's outstanding tokens
   * - ``DELETE /auth/api/v1/github/projects/{id}``
     - Delete a project and revoke its outstanding tokens; responds with ``204 No Content``

A request to create a project for Docverse uploads from the SQR-062 repository, restricted to the ``docs.yaml`` workflow on the default branch and on tags:

.. code-block:: json

   {
     "description": "Documentation builds for SQR-062",
     "username": "bot-gh-docverse-sqr-062",
     "repository": "lsst-sqre/sqr-062",
     "workflow": ".github/workflows/docs.yaml",
     "ref": "refs/heads/main",
     "scopes": ["write:docverse", "exec:linkcheck", "write:intersphinx"],
     "max_lifetime": "PT1H"
   }

The response echoes the project with the resolved username, ``id``, the pinned ``repository_id`` and ``repository_owner_id``, ``enabled`` defaulting to ``true``, and the audit fields.
In requests, durations are accepted as a number of seconds or as an ISO 8601 duration (``PT1H``, ``PT15M``), as Pydantic's duration validation allows; responses always use ISO 8601, and durations are stored as seconds.

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

The endpoint lives at ``POST /auth/api/v1/github/token`` and follows the OAuth 2.0 token exchange grammar of RFC 8693.
Gafaelfawr already uses standard interfaces where it talks to third-party software, and this endpoint is consumed by GitHub-hosted tooling rather than by Gafaelfawr's own clients, so the standard grammar is preferred over a Gafaelfawr-specific JSON shape.
That alternative is described at the end of this section.

Request and response
--------------------

RFC 8693 defines a standard grammar for trading one security token for another.
The request is form-encoded, as for every OAuth 2.0 token endpoint:

.. code-block:: http

   POST /auth/api/v1/github/token HTTP/1.1
   Content-Type: application/x-www-form-urlencoded

   grant_type=urn:ietf:params:oauth:grant-type:token-exchange
   &subject_token=eyJhbGciOiJSUzI1NiIs...
   &subject_token_type=urn:ietf:params:oauth:token-type:id_token
   &scope=write:docverse exec:linkcheck write:intersphinx

``grant_type`` and ``subject_token_type`` must have exactly the values shown; any other value is rejected.
``subject_token`` is the GitHub ID token.
The ``scope`` parameter is optional and, as in RFC 6749, is a space-separated list; when present it narrows the scopes to a subset of what the matching project allows, and when absent the token receives the project's full scope list.
RFC 8693 defines no parameter for the requested lifetime, so the minted token always receives the project's ``max_lifetime``.
Per-request lifetime narrowing has no use case among the motivating services, and if one appears, OAuth 2.0 permits extension parameters, so a ``lifetime`` parameter in seconds could be added without leaving the standard framing.
The other RFC 8693 parameters, ``resource``, ``audience``, ``requested_token_type``, ``actor_token``, and ``actor_token_type``, are not used and are rejected if present, so that a client cannot believe it has asked for something the endpoint silently ignored.

A successful response is ``200 OK`` and follows RFC 6749 and RFC 8693:

.. code-block:: json

   {
     "access_token": "gt-vHj2M...",
     "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
     "token_type": "Bearer",
     "expires_in": 3600,
     "scope": "write:docverse exec:linkcheck write:intersphinx"
   }

``access_token`` is the Gafaelfawr token, ``expires_in`` is its remaining lifetime in seconds, and ``scope`` is the space-separated list of scopes it carries.
The response carries nothing Gafaelfawr-specific, such as the bot username or the matched project; a workflow does not need them, and the audit trail records both.
As RFC 6749 requires, the response is sent with ``Cache-Control: no-store``.

Errors use the OAuth 2.0 error document with HTTP 400: ``invalid_request`` for a malformed request, including an unsupported ``grant_type`` or ``subject_token_type``; ``invalid_grant`` for an ID token that fails verification or matches no enabled project; and ``invalid_scope`` for a scope the project does not allow.
Each carries an ``error_description`` with a human-readable reason that a workflow author can act on.
Gafaelfawr's OpenID Connect server already implements this error document for its own token endpoint, so the error handling can be shared.

The alternative considered was a JSON request and response in the style of PyPI's mint-token endpoint and the rest of Gafaelfawr's ``/auth/api/v1`` routes: a body of ``{"token": "<jwt>", "scopes": [...]}`` answered with ``201 Created`` and a document carrying the token, the bot username, the scopes, the expiration time, and the matched project ID, with errors in Gafaelfawr's ``detail`` format.
Its advantages were consistency with the rest of the API and a response that could echo the resolved identity.
Neither outweighs having a documented grammar and error vocabulary that reviewers and future maintainers can read in the RFC instead of in Gafaelfawr's documentation, and that matches how cloud providers consume GitHub ID tokens.

Processing
----------

The endpoint processes a request in the following steps.
Each failure is logged with the reason and the claims that could be read, so that a workflow author can be helped without seeing Gafaelfawr's logs.

1. Decode the JWT header and read ``kid`` and ``alg``.
   Reject anything but an asymmetric algorithm GitHub uses (``RS256``).
2. Load GitHub's JWKS from the URL in its OpenID configuration document, from a cache with the configured lifetime.
   If ``kid`` is not in the cached set, refresh the cache once before failing, since GitHub rotates keys.
3. Verify the signature, then check that ``iss`` is ``https://token.actions.githubusercontent.com``, that ``aud`` equals the configured audience, that the current time is not before ``nbf``, allowing a small clock skew, and that it is strictly before ``exp``.
4. Read the claims the registry needs: ``repository``, ``repository_id``, ``repository_owner_id``, ``job_workflow_ref``, ``ref``, and, if present, ``environment``.
   A token missing any required claim is rejected.
5. Find the enabled projects for that repository and evaluate the matching rules.
   No match is a rejection; because the admin API refuses overlapping projects, at most one can match.
6. Determine the token's scopes and lifetime.
   Requested scopes must be a subset of the project's scopes; a request for a scope outside that set is rejected rather than silently narrowed, so that a misconfigured workflow fails visibly.
   The lifetime is the project's ``max_lifetime``.
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
It also preserves the provenance information that Ook currently derives by verifying the ID token itself, so Ook can stop doing so and accept only Gafaelfawr tokens: the repository, workflow, and commit an upload came from are supplied in the upload request and attributed to the bot user, and the exchange audit trail ties that bot token to the exact workflow run.

Using the exchange from a workflow
==================================

From a workflow author's point of view the exchange is two steps: obtain an ID token from GitHub with the right audience, and post it to Gafaelfawr.
The job must request the ``id-token: write`` permission, which is best granted at the job level so that other jobs in the workflow cannot mint tokens.
As a shell step, it looks like this:

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
               --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
               --data-urlencode "subject_token_type=urn:ietf:params:oauth:token-type:id_token" \
               --data-urlencode "subject_token=$id_token" \
               --data-urlencode "scope=write:docverse" | jq -r .access_token)
             echo "::add-mask::$token"
             echo "token=$token" >> "$GITHUB_OUTPUT"

         - name: Upload to Docverse
           uses: lsst-sqre/docverse-upload@v1
           with:
             token: ${{ steps.gafaelfawr.outputs.token }}

The ``::add-mask::`` line keeps the minted token out of the job log.
Nothing in the repository's secrets is used.

Writing this shell by hand in every repository is error-prone, so the exchange is packaged once as a composite GitHub Action in the ``lsst-sqre`` organization, provisionally ``lsst-sqre/gafaelfawr-token-action``, that takes the Phalanx environment URL and optional scopes as inputs, performs both steps, masks the result, and exposes it as a ``token`` output.
This action is the first deliverable, and SQuaRE's existing actions that talk to these services, such as ``docverse-upload``, are used alongside it: a workflow runs the token action and passes its output to the service action, as in the example above.
Building the exchange into those actions directly with ``@actions/core``'s ``getIDToken()``, and adding a helper to the ``rubin.gafaelfawr`` client library that performs the exchange when it detects the GitHub Actions environment variables, can follow once the action has proven the interface.

Whichever packaging is used, the audience must be the exact base URL of the Phalanx environment, because Gafaelfawr in each environment accepts only its own audience.
A workflow that talks to services in two environments requests two ID tokens and performs two exchanges.

.. _provisioning:

Automated provisioning of projects
==================================

Most projects in the registry will be technotes, and technotes are not created by hand.
An author asks for one in Slack; `Squarebot <https://github.com/lsst-sqre/squarebot>`__ publishes the request to a Kafka topic; and `Templatebot <https://github.com/lsst-sqre/templatebot>`__, a service deployed in Roundtable, consumes the request, creates the GitHub repository from the technote template, and registers the new document with Docverse.
For the new repository's first documentation build to succeed, Templatebot must also create its project in Gafaelfawr's registry as part of the same step.
This section describes how Templatebot is given that ability without giving it the powers of a token administrator.

Why not ``admin:token``
-----------------------

The registry's admin API requires ``admin:token`` because a project is a standing authorization to mint tokens for a chosen bot user with chosen scopes, which is the power that scope already guards.
Templatebot could hold a service token with ``admin:token``, as Noteburst does today, and call the admin API directly.
But that scope lets its holder view, create, modify, and delete every token in the environment, so a compromise of Templatebot, or of the Kafka topic it consumes, would become a compromise of every identity in Roundtable.
Templatebot needs a far narrower power.
It only ever creates projects of one shape: for a repository it has just created, under one of a handful of GitHub organizations, with the same workflow, ref, scopes, and lifetime every time.
The registry should let an administrator express that shape once and let Templatebot supply only the repository.

Project templates
-----------------

A project template is a registry resource that fixes every field of a project except the repository.
Templates are created and modified only by token administrators, through routes under ``/auth/api/v1/github/project-templates`` that require ``admin:token``, and changes to them are recorded in the project history table alongside changes to projects.
Templates could instead be Helm values in Phalanx, which would give them pull-request review, but keeping them in the registry gives trust configuration one audit history and one API, and lets a template change without a Gafaelfawr deploy.
A template's ``name`` is immutable, and a template cannot be deleted while any project records it: the request fails with ``409 Conflict`` and the number of projects, so that the ``template`` field on projects always names a template that exists and the ``?template=`` filter cannot be made to lie by deleting a template and creating another with the same name.
A template whose projects should remain is retired by disabling it.

.. list-table:: Fields of a project template
   :header-rows: 1
   :widths: 20 12 10 58

   * - Field
     - Type
     - Required
     - Description
   * - ``name``
     - string
     - yes
     - Identifier, a short slug such as ``technote``; unique, used in URLs and by provisioners
   * - ``description``
     - string
     - no
     - Free text for administrators
   * - ``owners``
     - list of strings
     - yes
     - GitHub owners (organizations or users) whose repositories may be provisioned from this template; each is resolved to its numeric ID through GitHub's REST API (``GET /users/{owner}``) when the template is created or modified and pinned alongside the name
   * - ``repository_pattern``
     - string
     - no
     - Regular expression the repository name, without the owner, must match, such as ``^[a-z]+-[0-9]{3,}$`` for technote series; unset matches any name
   * - ``workflow``, ``environment``, ``ref``
     - as for projects
     - no
     - Constraints copied into every project provisioned from the template
   * - ``scopes``, ``max_lifetime``, ``uid``, ``gid``, ``groups``
     - as for projects
     - as for projects
     - Policy copied into every project provisioned from the template; the same scope restrictions apply
   * - ``enabled``
     - boolean
     - yes
     - Disabled templates cannot be used to provision; projects already provisioned from them are unaffected
   * - ``created``, ``modified``, ``created_by``, ``modified_by``
     - as for projects
     - generated
     - Audit metadata

The template for technotes pins the reusable workflow that every technote repository calls, leaves ``ref`` unset because Docverse publishes an edition for every branch and pull request, and grants the documentation scopes.
Its owners are the GitHub organizations of the technote series offered by the `technote templates <https://github.com/lsst/templates/blob/main/project_templates/technote_rst/templatekit.yaml>`__ in lsst/templates, including the ``lsst-sqre-testing`` organization that test technotes are created in, so that test runs of Templatebot exercise provisioning too:

.. code-block:: json

   {
     "name": "technote",
     "description": "Documentation builds for technotes created by Templatebot",
     "owners": ["lsst-dm", "lsst", "lsst-it", "lsst-pst", "lsst-sims", "lsst-so", "lsst-sqre", "lsst-tstn", "lsst-sqre-testing"],
     "repository_pattern": "^[a-z]+-[0-9]{3,}$",
     "workflow": "lsst-sqre/rubin-sphinx-technote-workflows/.github/workflows/ci.yaml",
     "scopes": ["write:docverse", "exec:linkcheck", "write:intersphinx"],
     "max_lifetime": "PT1H"
   }

:ref:`The pinned owner IDs <pinned-ids>` extend the registry's defense against resurrection to the template: if an organization in the list is deleted and its name re-registered by someone else, repositories under the new organization fail the owner check even though the name still matches.

Provisioning a project
----------------------

A provisioner creates a project from a template with a single request that names the template and supplies the repository, with an optional ``requested_by`` and ``description``:

.. code-block:: text

   POST /auth/api/v1/github/project-templates/technote/projects

.. code-block:: json

   {
     "repository": "lsst-sqre/sqr-120",
     "requested_by": "slack:U024BE7LH",
     "description": "Created by Templatebot for Jonathan Sick, Slack request at 2026-11-04T18:02:11Z"
   }

Gafaelfawr checks that the template is enabled, looks up the repository through GitHub's REST API as the admin API does, and checks that the repository's owner is in the template's ``owners`` list by name and by pinned ID, and that the repository name matches ``repository_pattern`` if the template has one.
It then creates the project exactly as the admin API would from a request carrying the template's fields and the supplied repository: the username is derived as ``bot-gh-<owner>-<repo>`` and may not be chosen, the repository IDs are pinned, overlap with existing projects is checked, and ``created_by`` and the history's actor are set from the token that authenticated the request, so a provisioner cannot claim to be anyone but itself.
The project records the template's name in a new nullable ``template`` field, so that administrators can list what a template has produced with the ``?template=`` filter on the project list route.
The response is the same ``201 Created`` document the admin route returns.

The template's fields are copied into the project rather than referenced from it.
A provisioned project is therefore a complete, self-describing project: the matching rules, the overlap rule, and the statement that a project's configuration alone says what it trusts are all unchanged.
The cost is that editing a template changes only projects provisioned afterwards.
A fleet-wide change, such as a new path for the reusable workflow, is applied by an administrator with a script that lists the template's projects and patches each one.
That is the right place for it: such a change alters what every technote repository is trusted to do, and it should be made by a token administrator, not inherited silently from a template edit.

The route is idempotent, because Templatebot consumes Kafka with at-least-once delivery and its project creation runs as a sequence of steps that may be retried after a failure.
When a project for the same repository already exists and records the same template, the route returns it with ``200 OK`` instead of creating a duplicate or failing with ``409 Conflict``.
A project for the same repository that was created by hand, or from a different template, conflicts as usual, and Templatebot reports the conflict in Slack rather than guessing which policy was intended.

A provisioner cannot choose a username, scopes, constraints, or lifetime, and it cannot modify or delete projects or read the templates' policy beyond what the created project echoes.
The most it can do is extend a policy an administrator wrote to a repository an administrator allowed.

The provisioning scope
----------------------

The provisioning route is authorized by a new scope, provisionally ``admin:github-oidc-provision``, in the namespace Gafaelfawr reserves for its own use.
The scope grants exactly one capability, creating a project from a template, and no other registry route; ``admin:token`` also satisfies the route, since token administrators can do everything a provisioner can.
Because the scope begins with ``admin:``, the registry refuses it on projects and templates, as it refuses every reserved scope.
No GitHub workflow can therefore obtain a token that provisions projects, which closes a loop that would otherwise let a workflow extend trust to new repositories.

Templatebot's service token is a ``GafaelfawrServiceToken`` resource in its Phalanx chart, for the bot user ``bot-templatebot`` with the ``admin:github-oidc-provision`` scope and whatever scope Docverse requires for registering documents.
Templatebot will need such a token anyway as Docverse replaces the LSST the Docs administrator password that Templatebot currently holds.
Gafaelfawr's Kubernetes operator creates the token, stores it in a secret in Templatebot's namespace, and rotates it; no human handles it, and it is valid only in the Roundtable environment.

What a compromise of that token yields is bounded by the template.
An attacker holding it can register, for any repository under the template's owners whose name matches its pattern, a project granting the template's scopes to that repository's own bot user, and still needs write access to such a repository to mint a token.
Templatebot's GitHub App credential already allows creating and pushing to repositories in those same organizations, and every technote repository is meant to carry this policy, so the marginal exposure is a non-technote repository with a technote-shaped name receiving documentation-grade scopes.
An administrator who finds such a project disables it, which revokes its tokens, and can find everything a provisioner made with the ``?template=`` filter and the ``created_by`` field.

The person who asked for the technote is not the actor in the registry's audit fields; Templatebot is.
That is deliberate: ``created_by`` names the principal whose token Gafaelfawr verified, which is the identity accountable for the request and the one an administrator filters on to find everything a provisioner did, whereas the requester is a claim the provisioner makes that Gafaelfawr has no way to check.
To keep the human cause of each project visible, the provisioning request carries an optional ``requested_by`` field, stored on the project and in its history row and exposed as a filter on the project list route.
Gafaelfawr treats the value as an opaque string and never resolves or validates it, because the registry cannot know how requests reach a provisioner.
Today they arrive from Slack, where the requester is a Slack user ID; a future provisioner might be an MCP server whose callers are authenticated by Gafaelfawr itself, so the requester would be a Roundtable username.
The provisioner therefore says what kind of identifier it is sending, by the convention ``<source>:<identifier>``: ``slack:U024BE7LH`` from Templatebot, ``gafaelfawr:jsick`` from a service behind Gafaelfawr's ingress, or another prefix for a source that does not exist yet, with no change to Gafaelfawr.
Stable identifiers are preferred over display names, since a Slack handle can change while the user ID cannot; the provisioner may put the display name in ``description`` for readers, as in the example above.
The provisioner's own logs tie the request it received to the project it created.
Authorizing who may create a technote at all remains the concern of whatever system accepts the request, Slack and Squarebot today, and this design does not change it.

Order of operations and other environments
------------------------------------------

Templatebot should provision the Gafaelfawr project after it creates the repository on GitHub and before it pushes the initial commit, because the push triggers the first workflow run and that run must already find its project.
The GitHub lookup that pins the repository IDs succeeds on an empty repository.
Templatebot also compares the pinned IDs echoed in the response with the IDs GitHub returned when it created the repository, which closes mechanically the lookup race the admin API leaves to an administrator's eye.

Like the registry, the provisioning route exists per Phalanx environment, and a Phalanx-managed service token is valid only in the environment whose Gafaelfawr issued it.
Templatebot's token is issued by Roundtable, where Docverse and Ook run, so documentation projects can be provisioned there.
Notebook technotes additionally need a project in a Rubin Science Platform environment for Noteburst, and Templatebot has no token in those environments.
In the first phase those projects are registered by hand, which is acceptable for the small number of notebook technotes.
If that becomes a burden, an administrator can issue Templatebot a service token in the RSP environment carrying only ``admin:github-oidc-provision`` and store it in Templatebot's Vault secrets.
The narrow scope makes that a tolerable long-lived secret in a way that an ``admin:token`` service token would not be.

Alternatives considered
-----------------------

Organization-wide projects, whose ``repository`` is a pattern such as ``lsst-sqre/*`` with the bot username derived per repository at exchange time, would remove the need for provisioning altogether.
They are rejected because they change the trust boundary from write access to a registered repository to write access to any repository in the organization, and there are hundreds.
They also move per-repository authorization into the services: Docverse would have to check that the bot username matches the document being uploaded, whereas the services in this design authorize on scopes alone.
Finally, they give up the property that a project's configuration says what it trusts and that a single repository can be disabled.
A template offers the same economy of configuration while keeping the per-repository grain.

Trust on first use, in which Gafaelfawr creates a project automatically when an unknown repository under an allowed owner presents an ID token, has the same problems and a worse one: registration would be triggered by the party being registered.

A per-template list of the usernames allowed to provision from it, in addition to the scope, would let an administrator bind each template to one service.
It is not proposed because there is one template and one provisioner, and the scope alone is the simpler statement; the field is easy to add if a second provisioner appears.

Asking an administrator to approve each provisioning request in Slack would reintroduce the manual step that automating technote creation exists to remove.

Security considerations
=======================

The trust boundary of this design is write access to a registered repository.
Anyone who can get a workflow to run in that repository on a matching workflow, environment, and ref can obtain a token with the project's scopes.
This is the same boundary PyPI accepts, and it is a better one than a stored secret, which anyone with access to the repository's settings can read and use from anywhere.
Administrators tighten the boundary with the registry constraints: pinning the workflow file, restricting the ref to the default branch or release tags, and, where review before use is wanted, requiring a GitHub environment, since environments can be configured with required reviewers and deployment branch rules.

GitHub environments
-------------------

A GitHub environment is a named deployment target in a repository that can carry protection rules: required reviewers, a wait timer, and a rule restricting which branches and tags may deploy to it.
A job that declares an ``environment`` cannot start until those rules are satisfied, and only then does its ID token carry the ``environment`` claim.
Pinning ``environment`` on a project therefore delegates part of the authorization decision to GitHub: a token for that project can be minted only by a job that passed the environment's rules.
A job that declares no environment has no ``environment`` claim, so a project that pins one never matches it; the constraint cannot be bypassed by omission.
PyPI makes the environment optional but strongly recommends it, and this design takes the same position.
It is not required, because several motivating use cases, such as link checking on every push, have no deployment step and nothing for a reviewer to approve.
For projects with write scopes, administrators should prefer an environment over a bare ref constraint where the repository's maintainers will configure one: the deployment branch rule gives the same ref restriction, and required reviewers add a human gate that no claim constraint can express.

Pull requests and forks
-----------------------

GitHub runs a workflow for a pull request under one of two events, and they differ in what an ID token attests.
A ``pull_request`` event runs the workflow file from the pull request's head.
When the pull request comes from a fork, GitHub downgrades every write permission of the job to read, including ``id-token: write``, so such a run cannot obtain an ID token at all.
A ``pull_request`` run from a branch within the repository keeps its permissions and does get a token, with ``event_name`` set to ``pull_request`` and ``ref`` set to the merge ref, ``refs/pull/<number>/merge``.
A ``pull_request_target`` event instead runs the workflow file from the base branch with the base repository's full permissions, even for a pull request from a fork, and its ``ref`` claim is the base branch, such as ``refs/heads/main``.
Such a run does obtain an ID token, and a ``ref`` constraint on the default branch does not exclude it.

The ``repository`` claim always names the base repository, never a fork, so a fork cannot match a project on its own.
The hazard is a registered workflow that uses ``pull_request_target`` and then checks out and executes the pull request's code: it runs untrusted code under the base repository's identity and can obtain the project's token.
This is the same hazard that ``pull_request_target`` poses for repository secrets, and the defense is the same.
Workflows that use ``pull_request_target`` must not run code from the pull request, and projects with write scopes should pin the workflow file so that only workflows the maintainers control can mint tokens.
A project with no workflow or ref constraint grants its scopes to same-repository ``pull_request`` builds, which is intended for read-only scopes such as link checking; for write scopes both the workflow and the ref should be constrained.
An ``event_name`` constraint would exclude pull-request events outright and is the natural extension if pinning the workflow proves insufficient.

Audience confusion
------------------

Each Phalanx environment's Gafaelfawr accepts only its own base URL as the audience, so an ID token obtained for one environment cannot be exchanged in another.
A workflow that mistakenly requests a generic audience fails at exchange rather than succeeding somewhere unintended.

Replay
------

A GitHub ID token is valid for minutes and the exchange is idempotent in effect: replaying one yields another token for the same bot user under the same policy, which the holder of the first token could already use.
Recording the ``jti`` of exchanged tokens in Redis until their ``exp`` would make each ID token single-use, but since a replay gains nothing that the first exchange did not already grant, and the window is minutes long, this design relies on the ID token's short validity and does not track ``jti``.

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

The registry refuses ``admin:token`` and the other reserved ``admin:`` scopes, so no workflow can obtain token-administrator powers or, through ``admin:github-oidc-provision``, the power to register further repositories.
A per-environment allowlist of registrable scopes was considered and is not proposed: the registry's own scope list, which only token administrators can set, is the control.
Gafaelfawr's model of scopes and delegation is expected to be reworked in future, and any further restriction belongs with that work.

Automated provisioning
----------------------

Templatebot's provisioning token is the one long-lived credential this design adds, and it is held by a service in the cluster rather than in a GitHub repository.
Its scope cannot mint tokens, read or modify existing projects, or choose a username, scopes, or constraints; it can only instantiate a template for a repository under an owner the template allows.
The exposure from its compromise is therefore the template's policy applied to a repository that should not have had it, and it is found with the project list's ``?template=`` filter and cut off by disabling the project.
This is the trade the design makes: a narrow, bounded power for Templatebot in exchange for never handling ``admin:token`` outside Gafaelfawr's administrators.

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
This section sketches the extension and the questions it raises; it is not part of the proposal under review.

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

The security model of this extension is not settled, and two problems in it are new rather than inherited from the first phase.

First, a user can register a repository they do not control, and nothing in the first phase lets Gafaelfawr check the claim, since a platform identity is not a GitHub identity.
Registering a repository means that anyone who can run a workflow in it can act as the registering user on the platform with the chosen scopes.
The user bears the consequences, but so does the project: abuse of a science user's credentials consumes platform resources and can expose data the user has rights to, so the platform cannot simply let users accept the risk on their own behalf.
A registration therefore needs proof of control before it is trusted.
One workable form is a challenge: Gafaelfawr issues a nonce for a pending project, and the user must make a workflow in the repository present an ID token whose exchange request carries that nonce, which proves that the registrant can run workflows there.
Another is to limit user-owned projects to repositories under GitHub owners the platform already associates with the user, which is possible only where GitHub is the identity provider.

Second, several users may register the same repository.
Adding a ``username`` parameter to the exchange request so that a workflow can say which owner it means does not resolve this: it lets any workflow in the repository choose any of the registered users, so each registrant can be impersonated by whoever else can run workflows there.
Proof of control at registration narrows this to collaborators impersonating one another, the same boundary as sharing write access, but does not remove it.
Distinguishing registrants by their matching constraints instead, for example by requiring each user-owned project to pin a distinct workflow file, avoids the parameter but leaves collaborators able to edit each other's workflows.

Because these questions are open, the first phase makes no provision for user-owned projects: it includes neither the ``owner`` column nor a ``username`` exchange parameter.
Both are easy to add in a later database migration and API revision, and nothing in the admin-owned flow changes when they arrive.

References
==========

- `OpenID Connect in GitHub Actions: reference <https://docs.github.com/en/actions/reference/security/oidc>`__: the ID token's claims, ``sub`` formats, and how a job requests a token.
- `About security hardening with OpenID Connect <https://docs.github.com/en/actions/concepts/security/openid-connect>`__: GitHub's conceptual overview.
- `PyPI trusted publishers: internals and technical details <https://docs.pypi.org/trusted-publishers/internals/>`__: the two-phase exchange and the ``repository_owner_id`` resurrection defense.
- `PyPI trusted publishers: using a publisher <https://docs.pypi.org/trusted-publishers/using-a-publisher/>`__: the ``/_/oidc/audience`` and ``/_/oidc/mint-token`` endpoints.
- `RFC 8693: OAuth 2.0 Token Exchange <https://www.rfc-editor.org/rfc/rfc8693>`__: the grammar of the exchange endpoint.
- `RFC 6749: The OAuth 2.0 Authorization Framework <https://www.rfc-editor.org/rfc/rfc6749>`__: the token response and error document the exchange endpoint follows.
- `Gafaelfawr: service tokens <https://gafaelfawr.lsst.io/user-guide/service-tokens.html>`__: bot usernames and the ``GafaelfawrServiceToken`` resource.
- `Gafaelfawr: token administrators <https://gafaelfawr.lsst.io/user-guide/administrators.html>`__: the ``admin:token`` scope and the admin token API.
- `Gafaelfawr: scopes <https://gafaelfawr.lsst.io/dev/scopes.html>`__: reserved scopes and delegation rules.
- `Gafaelfawr: token types <https://gafaelfawr.lsst.io/dev/internals/gafaelfawr.models.enums.TokenType.html>`__: the existing token types, including the ``oidc`` type that is the precedent for the ``github`` type proposed here.
