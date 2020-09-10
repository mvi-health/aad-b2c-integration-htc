# Integrating Azure Active Directory B2C

## Library

We recommend the [oidc-client](https://github.com/IdentityModel/oidc-client-js) library. It does everything we need, while libraries used in AAD B2C’s own documentation (MSAL, ADAL) were insufficient.

    npm install --save oidc-client

[Documentation](https://github.com/IdentityModel/oidc-client-js/wiki)

## Use case: Constructing a login link for your own testing

In some cases, this is a link that we will supply when we redirect from TherapyView to your application. You'll probably also need a link like this for users who come directly to the domain (https://games.realsystem.com/ or similar). For testing, you'll need to supply your own page.

You'll need to catch the click event, construct a `UserManager` instance, and use the `signinCallback` method:

    // Top of file
    const { UserManager, WebStorageStateStore } = require('oidc-client');
    
    // Inside click handler
    const tenantName = 'mvi0162Tenant';
    const policyName = 'B2C_1_signin_0004';
    const clientId = '63de197c-a3bc-4b56-95d2-17e098356c77';
    
    const userManager = new UserManager({
      userStore: new WebStorageStateStore({ store: sessionStorage }),
      client_id: clientId,
      authority: `https://${tenantName}.b2clogin.com/tfp/${tenantName}.onmicrosoft.com/${policyName}/v2.0`,
      response_type: "id_token",
      scope: "openid profile",
      response_mode: "fragment",
      redirect_uri: "http://localhost:3000/",
      silent_redirect_uri: "http://localhost:3000/",
      post_logout_redirect_uri: "http://localhost:3000/",
    });
    
    userManager.signinRedirect();

A few notes on what's going on:

- The `tenantName`, `policyName` and `clientId` should all be the correct values for your use case, at least in testing. We will supply production values later.
- The `policyName` is versioned. We'll advance the version as time goes on, and we'll ask you to update as well in order that users can see our most up to date theming.
- The `tenantName` and `clientId` are not sensitive information. They can only be used to send the user and token to localhost.
- The `redirect_uri` is the most significant URL field. This is where you intend to redirect to after login completes, your destination page that will receive the ID token.
- The `silent_redirect_uri` isn't important yet. This is the URL that will be used in the hidden iframe for token refreshing. You don't need token refreshing for this page that displays the login link.
- The `post_logout_redirect_uri` isn't important yet either. This is where a logout will send the user. You won't be logging the user out from this page.
- `signinCallback` returns a promise, but it's not useful, because we're about to leave the page.
- **The URLs above are similar to what we use in testing, and will be honored by our AAD B2C configuration. If you want to use a different URL (https, domain, port, path), you need to tell us so we can configure it. Azure will refuse to redirect to any URL it hasn't been told about in advance.**

After leaving the page, the user will land on our themed Active Directory login page hosted by Microsoft. If the user already has an authenticated session, they will pass invisibly to to the `redirect_uri`.

## Use case: Receiving the token from AAD B2C

When arriving at your `redirect_uri`, the URL will have a token in the URL hash. An easy way to retrieve and remove from the URL using `UserManager` follows:

    // Inside onLoad or equivalent
    // Step 1: Initialize UserManager instance with same configuration as above
    // Step 2 ...
    userManager.signinRedirectCallback().then(result => {
      console.log('Retrieved ID token:', result.id_token);
    });

## Use case: Validating the token signature

The token is signed using RSA encryption and can be validated securely on the server. However, you don't need to do this, as our service will validate on each request that uses the token for authentication. In order to prove authenticity you just need to make a request to our service.

**Validation needs to be completed before using the contents of the token for anything even slightly sensitive.**

## Use case: Parsing the user ID from the token

It's trivial for you to decode the token and grab the userID, but keep in mind it's equally trivial to falsify it. **We need to validate first,** and we might as well parse the user ID for you at the same time.

You need a GraphQL query on our service that returns the userID. This doesn't exist yet, but we'll build it. This will need to be queried immediately after login in order to prove the token is valid and to return any data you need. In the future, if there are fields you need from our database, we can return them here along with the parsed userID.

If it helps you get started, you can parse the JWT (isolate the body by splitting on `.`, base64 decode, parse as JSON) and get the `sub` property, which contains the AAD B2C user ID. We reuse this as the primary key for the user, so it's the only ID you need. This is fine for early development but for release we'll need you to query us so we can prove the token is valid.

## Use case: Passing the token when making a request to our service

All requests to our service must include the token as a string in the `Authorization` header:

    Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJS...

## Use case: Refreshing the token

The ID token will have an expiration configured on it. In testing, that value will be at least an hour, but in production it's significantly less. Without a fresh token your request will be considered unauthenticated.

We recommend refreshing the token every sixty seconds, which gives you a nice buffer against the five minute token expiration configured in production.

### Building your `silent_redirect_uri`

The silent refresh flow will load the AAD B2C screens in an invisible iframe and redirect to the URL you configured in your `silent_redirect_uri`. That page needs to run some code that sends the new token out of the iframe to where you can use it.

Inside the page located at your `silent_redirect_uri`, run the following:

    // Inside onLoad or equivalent
    // Step 1: Initialize UserManager instance with same configuration as above
    // Step 2 ...
    
    userManager.signinSilentCallback();

Similar to `signinRedirect`, this technically returns a promise, but you probably don't need it as the page will unload immediately.

This page can now be used for silent renewal, however you choose to do that. Two options ...

### Strategy: Refresh using `automaticSilentRenew`

The `UserManager` class can be configured with the boolean `automaticSilentRenew` set to `false`. You may want to try this, but for us it was unreliable.

### Strategy: Refresh manually

Using `setInterval` or equivalent, call `signinSilent` on the `UserManager` instance:

    userManager.signinSilent().then(result => {
      console.log('New ID token', result.id_token);
    });
