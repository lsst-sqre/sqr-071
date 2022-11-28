:tocdepth: 1

Abstract
========

In normal operation, the Rubin Science Platform will authenticate a user via an external identity provider and ensure that all actions by that user are associated with that identity.
However, there are cases, such as debugging user problems or dealing with certain types of security issues, where it is invaluable for an administrator to be able to access the Science Platform as if they were another user.
They can then see exactly what that user would see and make changes as the user to fix problems.

This tech note proposes a design for implementing user impersonation in Gafaelfawr_, the authentication system for the Science Platform.
It focuses on user impersonation to services used via the browser, such as the Notebook Aspect and the Portal Aspect.
Impersonation of users to API services using bearer tokens is already possible by creating a new token for that user.
This will not be changed by this proposal.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are :dmtn:`234`, which describes the high-level design; :dmtn:`224`, which describes the implementation; and :sqr:`069`, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

Terminology
===========

For the purposes of this tech note, an *administrator* is a user who is flagged as an administrator in Gafaelfawr_.
Those users are automatically granted the ``admin:token`` scope for their session token when they log in.

An *impersonated user* is the apparent user from the perspective of the service.
This is the user that the administrator is temporarily pretending to be.

The *impersonator* is the administrator doing the impersonation.

The *impersonation token* is the additional token that the impersonator obtains and uses during impersonation.
This token will be used as the basis for authentication and access control while impersonation is enabled.

Requirements
============

It is already possible, using a token with ``admin:token``, to create a user token for an arbitrary user.
This allows impersonation of arbitrary users to API services authenticated with bearer tokens.
However, it is not possible for an administrator to put such a token in a browser cookie, and it is therefore difficult to use such tokens for access to browser-based services such as the Notebook Aspect or the Portal Aspect.

Via a UI in a web browser, administrators should be able to start impersonating an arbitrary user.
While that impersonation is active, any service they visit in that browser session should believe that the administrator is the impersonated user in every respect except (optionally) logging and a UI display.
For example, new notebook lab pods should be launched with the impersonated user's UID, GID, and groups; TAP queries should be saved in the impersonated user's history; all Portal actions should be done as the impersonated user; and so forth.

This impersonation should honor the following constraints:

#. ``admin:token`` scope is required to impersonate another user.
#. User impersonation should result in a Slack alert when it is initiated, identifying the administrator and the user being impersonated.
#. Authenticated web requests during impersonation should be logged (by Gafaelfawr_) with both the impersonated user and the impersonator.
#. Impersonation should be time-limited and automatically expire, reverting back to accessing services as the impersonator themselves.
   The lifetime should be on the order of hours, not days, and thus much shorter than the normal expiration period for user web sessions.
#. All delegated tokens (notebook and internal tokens) should expire at the end of the impersonation period, not at the end of the expiration period of the impersonator's underlying authentication token (which may be longer).
#. The impersonator should be able to stop the impersonation at any time via some web UI.
#. To the extent possible, the impersonator should see a graphical indication that they're currently impersonating another user, such as a banner at the top of the screen.
   This may not be possible in every service.
#. Applications should be able to detect that impersonation is happening so that they can, for example, alter their UI or logging.
   However, applications must not change non-cosmetic behavior during impersonation, since that defeats the point of seeing exactly what the user would see.
#. If the token used to start the impersonation is revoked, all subsequent impersonated requests should fail and all delegated tokens created by the impersonation should immediately be revoked.

Proposed implementation
=======================

At a high level, starting impersonation will add an additional token to the impersonator's cookie alongside their normal authentication cookie.
That token will be for the identity being impersonated.
It will have a short expiration time (matching the maximum impersonation period).
That token will be used for authentication and for creation of delegated tokens when the impersonator authenticates to any service.
Ending impersonation revokes the additional token.

The Gafaelfawr APIs will return the impersonator as an additional field in responses to information requests about the impersonation token.

Storage
-------

While impersonation is in progress, the session cookie of the impersonator will gain an additional key:

- **impersonation**: Session token for the user being impersonated.

The contents will be a session token for the impersonated user.
Its Redis record will have an additional field:

- **impersonator**: Username of the administrator who obtained this token.

The presence of that field will flag that this token was created via impersonation.

The SQL database schema for tokens will have an additional column, ``impersonator``, which will be null except for tokens created via impersonation.
If the token was created via impersonation, it will be set to the username of the impersonator.

The Redis field and SQL database column value will be inherited by all delegated tokens created from the impersonated session token.
This in turn requires that the impersonator field be part of the cache key used to cache and reuse internal and notebook tokens so that non-impersonated tokens won't be cached and used during impersonated authentication requests and vice versa.

The token history table will similarly be extended to add an ``impersonator`` column with the same semantics as that column in the token table.

Token API
---------

All API requests authenticated using a session cookie with the ``impersonation`` field set will use the token in that field, instead of the ``token`` field, for authentication and authorization.

The ``/auth/api/v1/token-info`` and ``/auth/api/v1/user-info`` routes will include the ``impersonator`` field in their response if it is set.
Applications and UI frontends can use this field to determine whether impersonation is in progress.

The ``/auth`` endpoint will similarly use the ``impersonation`` field, if set, as the token for authentication and authorization.
If this field is set but the token contained in it has expired or been revoked, the ``impersonation`` field will be ignored and the normal ``token`` field will be used instead.
If the session token in the ``token`` field is invalid, even if the ``impersonation`` field is present and contains a valid token, the user should be treated as unauthenticated.

If the ``/auth`` endpoint uses an impersonation token, the minimum remaining token lifetime requested by the ``minimum_lifetime`` parameter will be ignored.
(This is necessary since the maximum impersonation lifetime, and thus the lifetime of the impersonation token, is likely to be shorter than the requested ``minimum_lifetime``.)

Three new API routes will be added, intended for use by the UI (see :ref:`ui`):

``GET /auth/api/v1/impersonation``
    Returns the name of the user currently being impersonated.

    .. code-block:: json

       {"username": "<impersonated-user>"}

    If no user is currently being impersonated, responds with a 404 error.

    This route is not strictly necessary, since the same information is returned by the ``/auth/api/v1/token-info`` and ``/auth/api/v1/user-info`` routes.
    It is included just for REST semantics.

``PUT /auth/api/v1/impersonation``
    Start impersonating a user.
    The body of the request should be the username the user wishes to impersonate.

    .. code-block:: json

       {"username": "<impersonated-user>"}

    A cookie containing a session token must be used to authenticate to this route.
    Any other authentication mechanism will return a 403 error.
    If the user does not have the ``admin:token`` scope, the request will return a 403 error.

    If the user is already impersonating a user, the request will return a 409 error.

    Otherwise, a new token (with the ``impersonator`` data field set) will be created for the impersonated user, and the session cookie will be updated to add that token to the ``impersonation`` field.
    This new token will have a lifetime equal to the maximum impersonation lifetime, which will be a configurable setting in Gafaelfawr.
    Gafaelfawr will then reply with 200 and the same JSON body as ``GET /auth/api/v1/impersonation``.

    Gafaelfawr will send a Slack alert with the impersonator, the impersonated user, and the expiration date and time of the impersonation on success.

``DELETE /auth/api/v1/impersonation``
    Stop impersonating a user.

    If no user is currently being impersonated, responds with a 404 error.
    Otherwise, the impersonation token will be revoked, the user's cookie will be updated to remove the extra token, and Gafaelfawr will respond with 204.
    In this case, Gafaelfawr will send a Slack alert saying that the impersonation has ended.

Logging
-------

All Gafaelfawr_ log messages from operations authenticated with an impersonation token will include the ``impersonator`` field from that token in the log message.

Applications that already obtain information about the user's token using the ``/auth/api/v1/token-info`` or ``/auth/api/v1/user-info`` routes may also include the ``impersonator`` information in log messages if it is convenient.
However, they do not need to do this; we can rely on the Gafaelfawr logs to understand what actions were taken by an impersonator.

If the impersonation token was never explicitly revoked (using the ``DELETE /auth/api/v1/impersonation`` API call), but the periodic maintenance cron job detects that an impersonation token has expired, it will send a Slack alert saying that the impersonation has expired.

.. _ui:

User interface
--------------

If it detects that the user has ``admin:token`` scope (via, for example, using the ``/auth/api/v1/login`` route), Squareone_ will provide a user interface to start user impersonation.
Under the hood, this will make the ``PUT`` API call to ``/auth/api/v1/impersonation``.
Since this is authenticated with a session cookie, a CSRF token must be included in the ``X-CSRF-Token`` HTTP header.
This CSRF token can be obtained from the ``/auth/api/v1/login`` route.

.. _Squareone: https://github.com/lsst-sqre/squareone

(While Squareone does not currently use the ``/auth/api/v1/login`` route, it will need to use it once it takes over providing the token management UI, so this work will be reusable as part of that effort.)

If user impersonation is currently enabled, Squareone will display a banner at the top of web pages it generates indicating that the user is currently impersonating another user and including the username of the user being impersonated.
This information can be obtained from the ``/auth/api/v1/user-info`` or ``/auth/api/v1/token-info`` routes.

The banner should provide a way to stop impersonation, which if used should make the ``DELETE`` API call to ``/auth/api/v1/impersonation``.
This call will also require a CSRF token.

The user's tokens shown in the token management UI will include impersonation tokens.
Those lines should include the username of the user doing the impersonation.
Similarly, token history entries should include the impersonator information, if any.

Applications intended for display in the web browser, such as the Notebook Aspect and the Portal Aspect, should display a similar banner as Squareone where possible, or otherwise modify their UI to indicate that another user is being impersonated.
However, they must not modify any other part of their behavior, since that would defeat the point of impersonation.
In particular, no configuration details of a user's lab pod created by the Notebook Aspect should vary based on whether the authenticating token is an impersonation token or not.
