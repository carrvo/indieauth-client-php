IndieAuth Client
================

This is a simple library to help with IndieAuth. There are two ways you may want to use it, either when developing an application that signs people in using IndieAuth, or when developing your own endpoint for issuing access tokens and you need to verify auth codes.

[![Build Status](https://travis-ci.org/indieweb/indieauth-client-php.png?branch=master)](http://travis-ci.org/indieweb/indieauth-client-php)


Quick Start
-----------

If you want to get started quickly, and if you're okay with letting the library store things in the PHP session itself, then you can follow the examples below. If you need more control or want to step into the details of the IndieAuth flow, see the [Detailed Usage for Clients](#detailed-usage-for-clients-detailed) below.

### Create a Login Form

You'll first need to create a login form to prompt the user to enter their website address. This might look something like the HTML below.

```html
<form action="/login.php" method="post">
  <input type="url" name="url">
  <input type="submit" value="Log In">
</form>
```

### Begin the Login Flow

In the `login.php` file, you'll need to initialize the session, and tell this library to discover the user's endpoints. If everything succeeds, the library will return a URL that you can use to redirect the user to begin the flow.

The example below will have some really basic error handling, which you'll probably want to replace with something nicer looking.

Example `login.php` file:

```php
<?php

if(!isset($_POST['url'])) {
  die('Missing URL');
}

// Start a session for the library to be able to save state between requests.
session_start();

// You'll need to set up two pieces of information before you can use the client,
// the client ID and and the redirect URL.

// The client ID should be the home page of your app.
IndieAuth\Client::$clientID = 'https://example.com/';

// The redirect URL is where the user will be returned to after they approve the request.
IndieAuth\Client::$redirectURL = 'https://example.com/redirect.php';

// Pass the user's URL and your requested scope to the client.
// If you are writing a Micropub client, you should include at least the "create" scope.
// If you are just trying to log the user in, you can omit the second parameter.

list($authorizationURL, $error) = IndieAuth\Client::begin($_POST['url'], 'create');
// or list($authorizationURL, $error) = IndieAuth\Client::begin($_POST['url']);

// Check whether the library was able to discover the necessary endpoints
if($error) {
  echo "<p>Error: ".$error['error']."</p>";
  echo "<p>".$error['error_description']."</p>";
} else {
  // Redirect the user to their authorization endpoint
  header('Location: '.$authorizationURL);
}

```

### Handling the Callback

In your callback file, you just need to pass all the query string parameters to the library and it will take care of things! It will use the authorization or token endpoint it found in the initial step, and will check the authorization code or exchange it for an access token as appropriate.

The result will be the response from the authorization endpoint, which will contain the user's final `me` URL as well as the access token if you requested one or more scopes.

If there were any problems, the error information will be returned to you as well.

The library takes care of canonicalizing the user's URL, as well as checking that the final URL is on the same domain as the entered URL.

Example `redirect.php` file:

```php
<?php
session_start();
IndieAuth\Client::$clientID = 'https://example.com/';
IndieAuth\Client::$redirectURL = 'https://example.com/redirect.php';

list($user, $error) = IndieAuth\Client::complete($_GET);

if($error) {
  echo "<p>Error: ".$error['error']."</p>";
  echo "<p>".$error['error_description']."</p>";
} else {
  // Login succeeded!
  // If you requested a scope, then there will be an access token in the response.
  // Otherwise there will just be the user's URL.
  echo "URL: ".$user['me']."<br>";
  if(isset($user['access_token'])) {
    echo "Access Token: ".$user['access_token']."<br>";
    echo "Scope: ".$user['scope']."<br>";
  }

  // You'll probably want to save the user's URL in the session
  $_SESSION['user'] = $user['me'];
}

```


Detailed Usage for Clients
--------------------------

The first thing an IndieAuth client needs to do is to prompt the user to enter their web address. This is the basis of IndieAuth, requiring each person to have their own website. A typical IndieAuth sign-in form may look something like the following.

```
Your URL: [ example.com ]

       [ Sign In ]
```

This form will make a GET or POST request to your app's server, at which point you can begin the IndieAuth discovery.

### Discovering the required endpoints

The user will need to define three endpoints for their URL before a client can perform authorization. Endpoints can be specified by either an HTTP `Link` header or by using `<link>` tags in the HTML head.

#### Authorization Endpoint

```html
<link rel="authorization_endpoint" href="https://indieauth.com/auth">
```

```
Link: <https://indieauth.com/auth>; rel="authorization_endpoint"
```

The authorization endpoint allows a website to specify the location to direct the user's browser to when performing the initial authorization request.

Since this can be a full URL, this allows a website to use an external auth server such as [indieauth.com](https://indieauth.com) as its authorization endpoint. This allows people to delegate the handling and verification of authorization and authentication to an external service to speed up development. Of course at any point, the authorization server can be changed, and API clients and users will not need any modifications.

The following function will fetch the user's home page and return the authorization endpoint, or `false` if none was found.

```php
$url = IndieAuth\Client::normalizeMeURL($url);
$authorizationEndpoint = IndieAuth\Client::discoverAuthorizationEndpoint($url);
```

#### Token Endpoint

```html
<link rel="token_endpoint" href="https://aaronparecki.com/api/token">
```

```
Link: <https://aaronparecki.com/api/token>; rel="token_endpoint"
```

The token endpoint is where API clients will request access tokens. This will typically be a URL on the user's own website, although this can technically be delegated to an external service as well.

The token endpoint is responsible for verifying the authorization code and generating an access token.

The following function will fetch the user's home page and return the token endpoint, or `false` if none was found.

```php
$url = IndieAuth\Client::normalizeMeURL($url);
$tokenEndpoint = IndieAuth\Client::discoverTokenEndpoint($url);
```

#### Micropub Endpoint

```html
<link rel="micropub" href="https://aaronparecki.com/api/post">
```

```
Link: <https://aaronparecki.com/api/post>; rel="micropub"
```

The [micropub](http://indiewebcamp.com/micropub) endpoint defines where API clients will make POST requests to create new posts on the user's website. When an API client makes a request, the request will contain the previously-issued access token in the header, and the micropub endpoint will be able to validate the request given that access token.

The following function will fetch the user's home page and return the token endpoint, or `false` if none was found.

```php
$url = IndieAuth\Client::normalizeMeURL($url);
$micropubEndpoint = IndieAuth\Client::discoverMicropubEndpoint($url);
```

The client may wish to discover all three endpoints at the beginning, and cache the values in a session for later use.


### Building the authorization URL

Once the client has discovered the authorization server, it will need to build the authorization URL and direct the user's browser there.

For web sites, the client should send a 301 redirect to the authorization URL, or can open a new browser window. Native apps must launch a native browser window to the autorization URL and handle the redirect back to the native app appropriately.

#### Authorization URL Parameters
* `me` - the user's URL.
* `redirect_uri` - where the authorization server should redirect after authorization is complete.
* `client_id` - the full URL to a web page of the application. This is used by the authorization server to discover the app's name and icon, and to validate the redirect URI.
* `state` - the "state" parameter can be whatever the client wishes, and must also be sent to the token endpoint when the client exchanges the authorization code for an access token.
* `scope` - the "scope" value is a space-separated list of permissions the client is requesting.
* `code_challenge` - for [PKCE](https://oauth.net/2/pkce/) support, this is the hashed version of a secret the client generates when it starts.
* `code_challenge_method` - this library will always use S256 as the hash method.

The following function will build the authorization URL given all the required parameters. If the authorization endpoint contains a query string, this function handles merging the existing query string parameters with the new parameters.

```php
$url = IndieAuth\Client::normalizeMeURL($url);
$authorizationURL = IndieAuth\Client::buildAuthorizationURL($authorizationEndpoint, $url, $redirect_uri, $client_id, $state, $scope, $code_verifier);
```

Note: Your code should include the plaintext random secret, the `IndieAuth\Client` library will deal with hashing it for you.

### Getting authorization from the user

At this point, the authorization server interacts with the user, presenting them with a description of the request. This will look something like the following typical OAuth prompt:

```
An application, "Quill" is requesting access to your website, "aaronparecki.com"

This application would like to be able to
* **create** new entries on your website

[ Approve ]   [ Deny ]
```

If the user approves the request, the authorization server will redirect back to the redirect URI specified, with the following parameters added to the query string:

* `code` - the authorization code
* `state` - the state value provided in the request


### Exchanging the authorization code for an access token

Now that the client has obtained an authorization code, it needs to exchange it for an access token at the token endpoint.

To get an access token, the client makes a POST request to the token endpoint, passing in the authorization code as well as the following parameters:

* `code` - the authorization code obtained
* `me` - the user's URL
* `redirect_uri` - must match the redirect URI used in the request to obtain the authorization code
* `client_id` - must match the client ID used in the initial request
* `code_verifier` - if the client included a code challenge in the authorization request, then it must include the plaintext secret in the code exchange step here

The following function will make a POST request to the token endpoint and parse the result.

```php
$token = IndieAuth\Client::getAccessToken($tokenEndpoint, $_GET['code'], $_GET['me'], $redirect_uri, $client_id, $code_verifier);
```

The `$token` variable will include the response from the token endpoint, such as the following:

```php
array(
  'me' => 'https://aaronparecki.com/',
  'access_token' => 'xxxxxxxxx',
  'scope' => 'create'
);
```


### Making API requests

At this point, you are done using the IndieAuth client library and can begin making API requests directly to the user's website and micropub endpoint.

To make an API request, include the access token in an HTTP "Authorization" header like the following:

```
Authorization: Bearer xxxxxxxx
```



Usage for Token Endpoints
-------------------------

When issuing access tokens, you will need to verify the authorization code present in the request. In order to verify the authorization code, you'll first need to discover the user's auth server.

### Discovering the authorization endpoint

Requests to your token endpoint will contain the following parameters:
* me
* code
* client_id
* redirect_uri

You will need to verify the code with the user's authorization server, and then create an access token for them.

First, discover the user's authorization server by looking for a `rel="authorization_server"` link on their home page.

```php
$authorizationEndpoint = IndieAuth\Client::discoverAuthorizationEndpoint($_POST['me']);
```

If discovery is successful, `$authorizationEndpoint` will be the full URL, such as `https://indieauth.com/auth`. Now you need to verify the auth code with the auth server. To do this, you'll need to pass in all the parameters that were part of generating the auth code, including `redirect_uri` and `client_id`.

### Verifying the authorization code

```php
$auth = IndieAuth\Client::verifyIndieAuthCode($authorizationEndpoint, $_POST['code'], $_POST['me'], $_POST['redirect_uri'], $_POST['client_id']);
```

The response from the authorization server will be a JSON response containing the `me` parameter and more importantly, the `scope` parameter which is the list of scopes that was part of the authorization request.

```
{
  "me": "https://aaronparecki.com/",
  "scope": "create"
}
```

The `IndieAuth\Client::verifyIndieAuthCode` method parses this and returns it as an array:

```
$auth = array(
  'me' => 'https://aaronparecki.com/',
  'scope' => 'create'
);
```

If there was an error, the authorization server will return an HTTP 400 response with `error=invalid_request` and the `error_description` property indicating what went wrong. Errors may include:

* Missing 'code' parameter - the request did not include the "code" parameter
* Invalid code provided - could be because the code was not created at this authorization server, or just a bad code was provided
* The auth code has expired - Authorization codes are only valid for a short time
* The 'redirect_uri' parameter did not match

The error messages are meant to be read by developers, not by end users. Your token endpoint can pass this error through for debugging purposes, or you can return your own error.

### Issuing an access token

Assuming you get a successful response from the auth server containing the "me" and "scope" parameters, you can now generate an access token and send the reply.

The way you build an access token is entirely up to you. You may want to store tokens in a database, or you may want to use self-encoded tokens to avoid the need for a database. In either case, you must store at least the "me," "scope" and "client_id" values.

The example below generates a self-encoded token by encrypting all the needed information into a string and returning the encrypted string. This example uses a [JWT](https://github.com/firebase/php-jwt) library to encrypt the token, but you could use any method of encryption you wish.

```php
  // $auth is set from the IndieAuth\Client::verifyIndieAuthCode call from before

  $token_data = array(
    'date_issued' => date('Y-m-d H:i:s'),
    'me' => $auth['me'],
    'client_id' => $_POST['client_id'],
    'scope' => array_key_exists('scope', $auth) ? $auth['scope'] : '',
    'nonce' => mt_rand(1000000,pow(2,31))   // add some noise to the encrypted version
  );

  // Encrypt the token with your server-side encryption key
  $token = JWT::encode($token_data, $encryptionKey);

  header('Content-type: application/json');
  echo json_encode([
    'me' => $auth['me'],
    'scope' => $token_data['scope'],
    'access_token' => $token
  ]);
```

Note that you must return the parameters "me", "scope" and "access_token" in your response. This is because API clients will not know the user or scope of the token otherwise, and clients will need to discover the API endpoint using the "me" parameter before they can make API requests.

### Verifying access tokens

Verifying access tokens is not actually part of this library, since the token was generated by your own server. You will need to verify tokens using whatever method you wish.

For example, your API endpoint to create new posts may wish to require a "create" scope, so you would need to verify that the access token is valid and also contains the needed scope value.

The example below illustrates how to verify the self-encoded token we created above.

```php
// Verifies an access token, returning the token on success, or responding with an HTTP 400 error on failure
function requireAccessToken($requiredScope=false) {

  if(array_key_exists('HTTP_AUTHORIZATION', $_SERVER)
     && preg_match('/Bearer (.+)/', $_SERVER['HTTP_AUTHORIZATION'], $match)) {

    // Decode the token using the encryption key
    $token = JWT::decode($match[1], $encryptionKey);

    if($token) {
      // This is where you could add additional validations on specific client_ids. For example
      // to revoke all tokens generated by app 'http://example.com', do something like this:
      // if($token->client_id == 'http://example.com' && strtotime($token->date) <= strtotime('2013-12-21')) // revoked

      // Verify the token has the required scope
      if($requiredScope) {
        if(property_exists($token, 'scope') && in_array($requiredScope, explode(' ', $token->scope))) {
          return $token;
        } else {
          header('HTTP/1.1 401 Unauthorized');
          header('Content-type: application/json');
          echo json_encode([
            'error' => 'invalid_scope',
            'error_description' => 'The token provided does not have the necessary scope'
          ]);
          die();
        }
      } else {
        return $token;
      }
    }
  }

  header('HTTP/1.1 401 Unauthorized');
  header('Content-type: application/json');
  echo json_encode([
    'error' => 'unauthorized',
    'error_description' => 'An access token is required. Send an HTTP Authorization header such as \'Authorization: Bearer xxxxxx\''
  ]);
  die();
}
```



License
-------

Copyright 2013-2019 by Aaron Parecki and contributors

Available under the MIT and Apache 2.0 licenses. See LICENSE.txt

