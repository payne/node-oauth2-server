# Node OAuth2 Server [![Build Status](https://travis-ci.org/thomseddon/node-oauth2-server.png?branch=2.0)](https://travis-ci.org/thomseddon/node-oauth2-server)

Complete, compliant and well tested module for implementing an OAuth2 Server/Provider with [express](http://expressjs.com/) in [node.js](http://nodejs.org/)

## Installation

```
npm install node-oauth2-server
```

## Quick Start

The module provides two middlewares, one for authorization and routing, another for error handling, use them as you would any other middleware:

```js
var express = require('express'),
  oauthserver = require('node-oauth2-server');

var app = express();

app.configure(function() {
  app.oauth = oauthserver({
    model: {}, // See below for specification
    grants: ['password'],
    debug: true
  });
  app.use(express.bodyParser()); // REQUIRED
});

app.all('/oauth/token', app.oauth.grant());

app.get('/', app.oauth.authorise(), function (req, res) {
  res.send('Secret area');
});

app.use(app.oauth.errorHandler());

app.listen(3000);
```

After running with node, visting http://127.0.0.1:3000 should present you with a json response saying your access token could not be found.

Note: As no model was actually implemented here, delving any deeper, i.e. passing an access token, will just cause a server error. See below for the specification of what's required from the model.

## Features

- Supports authorization_code, password, refresh_token and extension (custom) grant types
- Implicitly supports any form of storage e.g. PostgreSQL, MySQL, Mongo, Redis...
- Full test suite

## Options

- *string* **model**
 - Model object (see below)
- *array* **grants**
 - grant types you wish to support, currently the module supports `password` and `refresh_token`
  - Default: `[]`
- *boolean* **debug**
 - If true, errors are logged to console
 - Default: `false`
  - Default: `false`
- *number* **accessTokenLifetime**
 - Life of access tokens in seconds
 - If `null`, tokens will considered to never expire
  - Default: `3600`
- *number* **refreshTokenLifetime**
 - Life of refresh tokens in seconds
 - If `null`, tokens will considered to never expire
  - Default: `1209600`
- *number* **authCodeLifetime**
 - Life of auth codes in seconds
  - Default: `30`
- *regexp* **clientIdRegex**
 - Regex to match auth codes against before checking model
 - Default: `/^[a-z0-9-_]{3,40}$/i`
- *boolean* **passthroughErrors**
 - If true, **non grant** errors will not be handled internally (so you can ensure a consistent format with the rest of your api)
- *boolean* **continueAfterResponse**
 - If true, `next` will be called even if a reponse has been sent (you probably don't want this)

## Model Specification

The module requires a model object through which some aspects or storage, retrieval and custom validation are abstracted.
The last parameter of all methods is a callback of which the first parameter is always used to indicate an error.

Note: see https://github.com/thomseddon/node-oauth2-server/tree/master/examples/postgresql for a full model example using postgres.

### Always Required

#### getAccessToken (bearerToken, callback)
- *string* **bearerToken**
 - The bearer token (access token) that has been provided
- *function* **callback (error, accessToken)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *object* **accessToken**
     - The access token retrieved form storage or falsey to indicate invalid access token
     - Must contain the following keys:
         - *date* **expires**
             - The date when it expires
             - `null` to indicate the token **never expires**
         - *string|number* **userId**
             - The user id (saved in req.user.id)

#### getClient (clientId, clientSecret, callback)
- *string* **clientId**
- *string|null* **clientSecret**
 - If null, omit from search query (only search by clientId)
- *function* **callback (error, client)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *object* **client**
     - The client retrieved from storage or falsey to indicate an invalid client
     - Saved in `req.client`
     - Must contain the following keys:
         - *string* **clientId**

#### grantTypeAllowed (clientId, grantType, callback)
- *string* **clientId**
- *string* **grantType**
- *function* **callback (error, allowed)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *boolean* **allowed**
     - Indicates whether the grantType is allowed for this clientId

#### saveAccessToken (accessToken, clientId, expires, user, callback)
- *string* **accessToken**
- *string* **clientId**
- *string|number* **userId**
- *date* **expires**
- *function* **callback (error)**
 - *mixed* **error**
     - Truthy to indicate an error


### Required for `authorization_code` grant type

#### getAuthCode (authCode, callback)
- *string* **authCode**
- *function* **callback (error, authCode)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *object* **authCode**
     - The authorization code retrieved form storage or falsey to indicate invalid code
     - Must contain the following keys:
         - *string|number* **clientId**
             - client id associated with this auth code
         - *date* **expires**
             - The date when it expires
         - *string|number* **userId**
             - The userId

#### saveAuthCode (authCode, clientId, expires, user, callback)
- *string* **authCode**
- *string* **clientId**
- *date* **expires**
- *mixed* **user**
   - Whatever was passed as `user` to the codeGrant function (see example)
- *function* **callback (error)**
 - *mixed* **error**
     - Truthy to indicate an error


### Required for `password` grant type

#### getUser (username, password, callback)
- *string* **username**
- *string* **password**
- *function* **callback (error, user)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *object* **user**
     - The user retrieved from storage or falsey to indicate an invalid user
     - Saved in `req.user`
     - Must contain the following keys:
         - *string|number* **id**

### Required for `refresh_token` grant type

#### saveRefreshToken (refreshToken, clientId, expires, user, callback)
- *string* **refreshToken**
- *string* **clientId**
- *string|number* **userId**
- *date* **expires**
- *function* **callback (error)**
 - *mixed* **error**
     - Truthy to indicate an error

#### getRefreshToken (refreshToken, callback)
- *string* **refreshToken**
 - The bearer token (access token) that has been provided
- *function* **callback (error, accessToken)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *object* **refreshToken**
     - The refresh token retrieved form storage or falsey to indicate invalid refresh token
     - Must contain the following keys:
         - *string|number* **clientId**
             - client id associated with this token
         - *date* **expires**
             - The date when it expires
             - `null` to indicate the token **never expires**
         - *string|number* **userId**
             - The userId


### Optional for Refresh Token grant type

#### revokeRefreshToken (refreshToken, callback)
The spec does not actually require that you revoke the old token - hence this is optional (Last paragraph: http://tools.ietf.org/html/rfc6749#section-6)
- *string* **refreshToken**
- *function* **callback (error)**
 - *mixed* **error**
     - Truthy to indicate an error

### Required for [extension grant](#extension-grants) grant type

#### extendedGrant (req, callback)
- *object* **req**
 - The raw request
- *function* **callback (error, supported, user)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *boolean* **supported**
     - Whether you support the grant type
 - *object* **user**
     - The user retrieved from storage or falsey to indicate an invalid user
     - Saved in `req.user`
     - Must contain the following keys:
         - *string|number* **id**

### Optional

#### generateToken (type, callback)
- *string* **type**
 - `accessToken` or `refreshToken`
- *function* **callback (error, token)**
 - *mixed* **error**
     - Truthy to indicate an error
 - *string|object|null* **token**
     - *string* indicates success
     - *null* indicates to revert to the default token generator
     - *object* indicates a reissue (i.e. will not be passed to saveAccessToken/saveRefreshToken)
         - Must contain the following keys (if object):
           - *string* **accessToken** OR **refreshToken** dependant on type

## Extension Grants
You can support extension/custom grants by implementing the extendedGrant method as outlined above.
Any requests that begin with http(s):// (as [defined in the spec](http://tools.ietf.org/html/rfc6749#section-4.5)) will be passed to it for you to handle.
You can access the grant type via req.oauth.grantType and you should pass back supported as `false` if you do not support it to ensure a consistent (and compliant) response.

## Example using the `password` grant type

First you must insert client id/secret and user into storage. This is out of the scope of this example.

To obtain a token you should POST to `/oauth/token`. You should include your client credentials in
the Authorization header ("Basic " + client_id:client_secret base4'd), and then grant_type ("password"),
username and password in the request body, for example:

```
POST /oauth/token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=password&username=johndoe&password=A3ddj3w
```
This will then call the following on your model (in this order):
 - getClient (clientId, clientSecret, callback)
 - grantTypeAllowed (clientId, grantType, callback)
 - getUser (username, password, callback)
 - saveAccessToken (accessToken, clientId, userId, expires, callback)
 - saveRefreshToken (refreshToken, clientId, userId, expires, callback) **(if using)**

Provided there weren't any errors, this will return the following (excluding the `refresh_token` if you've not enabled the refresh_token grant type):

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"
}
```

## Changelog

See: https://github.com/thomseddon/node-oauth2-server/releases

## Credits

Copyright (c) 2013 Thom Seddon

## License

[Apache, Version 2.0](https://github.com/thomseddon/node-oauth2-server/blob/master/LICENSE)
