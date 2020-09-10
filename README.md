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
    
    userManager.signinCallback();

A few notes on what's going on:

- The `tenantName`, `policyName` and `clientId` should all be the correct values for your use case, at least in testing. We will supply production values later.
- The `policyName` is versioned. We'll advance the version as time goes on, and we'll ask you to update as well in order that users can see our most up to date theming.
- The `tenantName` and `clientId` are not sensitive information. They can only be used to send the user and token to localhost.
- The `redirect_uri` is the most significant URL field. This is where you intend to redirect to after login completes, your destination page that will receive the ID token.
- The `silent_redirect_uri` isn't important yet. This is the URL that will be used in the hidden iframe for token refreshing. You don't need token refreshing for this page that displays the login link.
- The `post_logout_redirect_uri` isn't important yet either. This is where a logout will send the user. You won't be logging the user out from this page, nor perhaps at all.

TODO
