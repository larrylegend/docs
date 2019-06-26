---
description: Best practices for writing and managing Auth0 Rules.
topics:
  - best-practices
  - rules
contentType: reference
useCase:
  - best-practices
  - rules
---

# Rules Best Practices

[Extensibility](/topics/extensibility) provides the capability to add custom logic in Auth0 as a mechanism for building out last mile solutions for Identity and Access Management (IAM). Auth0 extensibility comes in several forms: Rules, Hooks, and scripts for both custom database donnection and custom database connection migration. Each is implemented using `node.js` running on the Auth0 platform in an Auth0 tenant. 

Whatever the use case, Auth0 extensibility provides comprehensive and sophisticated capability to tailor IAM operations to your exact requirements. However, if not utilized in the right way, this can open up the potential for improper or unintended use - which can lead to problematic situations down the line. In an attempt to address matters ahead of time, this document provides best practice guidance to both designers and implementers, and we recommend reading it in its entirety at least once, even if you've already started your journey with Auth0. 

## General

Auth0 extensibility executes at different points in the IAM pipeline. Rules, for example, run when artifacts for authenticity are generated (i.e. an ID Token in OpenID Connect (OIDC), an Access Token in OAuth 2.0, or an assertion in SAML. Hooks on the other hand provide additional extensibility where such artifacts are exchanged) or where user identities are created (see pre- and post-user registration Hooks for further details). Custom database scripts can be used to integrate with an existing user identity store, or can be used where automatic user migration (from a legacy identity store) is required. 

TBD

## Rules

Rules can be used in a variety of situations as part of the pipeline where artifacts for authenticity are generated - i.e. an ID Token in OpenID Connect (OIDC), an Access Token in OAuth 2.0, or an assertion in SAML. A new pipeline is created for each authentication request.

A number of pre-existing Rules/Rule templates are provided out-of-box to help you achieve your goal(s). However there are times when you will want to build your own Rule(s) in support of your  specific functionality/requirements. You may choose to extend or modify a pre-existing Rule/Rule template, or you may choose to start from scratch (using one of our samples to guide you). Either way, there are a number of best practices that you’ll want to adopt in order to ensure that you achieve the best possible outcome.

**TODO: add screenshot here of rules page**

The image above depicts an Auth0 Dashboard showing a number of enabled and disabled rules for a specific Auth0 Tenant. Enabled rules - those with the green toggle - are those rules that are active and will execute as part of a pipeline. Disabled rules - those with the greyed-out toggle - on the other hand, won't. 

### Anatomy

A rule is essentially an anonymous JavaScript function that is passed 3 parameters: a user object, a `context` object, and a `callback` function. 

```js
    function (user, context, callback) {
        // TODO: implement your rule
        callback(null, user, context);
    }
 ```

::: note
Anonymous functions make it hard in [debugging](#debugging) situations to interpret the call-stack generated as a result of any [exceptional error](#exceptions) condition. For convenience, consider providing a function name using some compact and unique naming convention to assist with diagnostic analysis (e.g. `function MyRule1 (user, context, callback) {...}`). Do not be tempted to add a trailing semicolon at the end of the function declaration as this will break rule execution.
:::

As depicted in the image below, rules execute in what is the pipeline associated with the generation of artifacts for authenticity that forms part of the overall [Auth0 engine](https://cdn.auth0.com/blog/auth0-raises-100m-to-fuel-the-growth/inside-the-auth0-engine-high-res.jpg). 

**TODO: add picture here**

### Size

When a pipeline is executed, all enabled rules are packaged together in the order in which they are listed and sent as one code blob to be executed as a [Webtask](https://webtask.io/). Webtasks currently have a [maximum execution limit](https://webtask.io/docs/limits) of 100 kB of code per instance. So the total size of implementation for all enabled rules must not exceed 100 kB - doing so will have unpredictable results. Note that the 100 kB limit does not include any `npm` modules referenced as part of any `require` statements.  

### Order

The order in which rules are displayed in the [Auth0 Dashboard](/dashboard) (see image above) dictate the order in which the rules will be executed. This is important as one rule may make one or more definitions within the [environment](#environment) associated with execution, that another rule may depend upon. In this case, the rule making the definition(s) should execute before the rule that makes use of them.

### Environment

Rules execute as a series of called JavaScript functions, in an instance of a [Webtask container](https://www.webtask.io/docs/containers). As part of this, a specific environment is provided, together with a number of artefacts supplied by both Webtask, and the Auth0 platform. 

#### NPM modules

Webtask containers can make use of a wide range of [`npm`](https://www.npmjs.com/) modules; npm modules not only reduce the overall size of rule code implementation, but also provide access to a wide range of pre-built functionality.

By default, a large list of publicly available npm modules are [supported out-of-the-box](https://auth0-extensions.github.io/canirequire/). This list has been compiled and vetted for any potential security concerns. If you require an npm module that is not supported out-of-the-box, then a request can be made via the [Auth0 support](https://support.auth0.com/) portal or via your Auth0 representative. Auth0 will evaluate your request to determine suitability. There is currently no support in Auth0 for the user of npm modules from private repositories.

::: Best practice
When using NPM modules to access external services, it’s recommended best practice to [keep API requests to a minimum](/best-practices/rules#reduce-api-requests), [avoid excessive calls to paid services](/docs/best-practices/rules#reduce-calls-to-paid-services), and avoid potential security exposure by [limiting what is sent](/docs/best-practices/rules#don-t-send-the-entire-context-object-to-external-services). For more information on this see the [performance](#performance) and [security](#security) sections below.
:::

#### Environment variables

The Auth0 rule ecosystem supports the notion of environment variables, accessed via what is defined as the globally available configuration object. The configuration object should be treated as read-only, and should be used for storing sensitive information - such as credentials or API keys for external service access. This mitigates having security sensitive values hard coded in a rule. 

It can also be used to support whatever Software Development Life Cycle (SDLC) best practice strategies you employ, by allowing you to define variables that have tenant specific values. This mitigates hard code values in a rule which may change depending upon which tenant is executing it.   

#### 'global' object

Webtask containers also provide the global object, which can be accessed across all rules executing within a Webtask container instance. The global object acts as a global variable and can be used to cache information, or to even define functions, that can be used across all rules that run.    

Rules can run more than once when a pipeline is executed, and this depends on the context of operation. For each context in which a rule is run, a new Webtask container may be instantiated, and for each instantiation of a new Webtask container the global object is reset. Thus any global declaration should also include provision for initialization - with that declaration typically being made as early as possible (i.e. in a rule that runs early in the execution order), e.g: 

```js
    global.tokenVerify = global.tokenVerify || function(token, secret) {
        /* The 'jwt.verify' function is synchronous, however wrapping with a promise
        * provides for better error management and integration within the logic
        * flow.
        */
     return new Promise(function(resolve, reject) {
      jwt.verify(
        token, 
        secret,{
        clockTolerance: 5},  
        function(err, decoded) {
          if (err) {
            reject(err);
          } else {  
            resolve(decoded);
        
      });
    });
   };
```

#### 'auth0' object

The auth0 object is an instance of the Management API Client (defined in the node-auth0 Node.js client library), and provides limited access to the Auth0 Management API. It is primarily used for updating metadata associated with the user object from within a rule. Like the context object (described below), the auth0 object contains security sensitive information, so you should not pass it to any external or 3rd party service. 

Further, the Auth0 Management API is both rate limited and subject to latency, so you should be judicious regarding how often calls are made. 

::: Best practice
It’s recommended best practice to make use of the auth0 object (and any other mechanisms for calling the Auth0 Management API) sparingly, and to always make sure that adequate exception and error handling is employed in order to prevent unexpected interruption of pipeline execution.
:::

### Execution

Each rule is executed as a JavaScript function; these functions are called in the order that the rules are defined. Rules execute sequentially - that is to say the next rule in order won’t execute until the previous rule has completed.

In pipeline terms, a rule completes when the callback function supplied to the rule is called. Failure to call the function will result in a stall of pipeline execution, and ultimately in an error being returned. Each rule must call the callback function at least once.
 
Rule execution supports the asynchronous nature of JavaScript, and constructs such as Promise objects and the like can be used. Asynchronous processing effectively results in suspension of a pipeline pending completion of the asynchronous operation. A webtask container typically has a 30 second execution limit, after which the container may be recycled. A recycle of a container will prematurely terminate a pipeline (suspended or otherwise), ultimately resulting in an error in authentication being returned (as well as resulting in a reset of the global object). 

::: note
Setting `context.redirect` will trigger a [Redirection](#redirection) once all rules have completed (the redirect is not forced at the point it is set). Whilst all rules must complete within the execution limit of the Webtask container for the redirect to occur, the time taken as part of redirect processing can extend beyond that limit. Redirection back to Auth0 via the `/continue` endpoint will cause the creation of a new Webtask container, in the context of the current pipeline, in which all rules will again be run.  
:::

Asynchronous execution will result in a (JavaScript) callback being executed after the asynchronous operation is complete. This callback is typically fired at some point after the main (synchronous) body of a JavaScript function completes. If a rule is making use of asynchronous processing then a call to the (Auth0) supplied callback function must be deferred to the point where asynchronous processing completes - and must be the final thing called. The (Auth0) supplied callback function must be called only once; calling the function more than once within a rule will lead to unpredictable results and/or errors.

#### `context` object

The `context` object provides information about the context in which a rule is run (such as client identifier, connection name, session identifier, request context, protocol, etc). Using the context object, a rule can determine the reason for execution. For example, as illustrated in the sample fragment below, `context.clientID` as well as `context.protocol` can be used to implement conditional processing to determine when rule logic is executed. The sample also shows some best practices for exception handling, use of `npm` modules (for Promise style processing), and the callback object. 

```js
  switch (context.protocol) {
    case 'redirect-callback':
      callback(null, user, context);
    	break;

    default: {
      user.app_metadata = user.app_metadata || {};
      switch(context.clientID) {
        case configuration.PROFILE_CLIENT: {
          user.user_metadata = user.user_metadata || {};
          Promise.resolve(new 
            Promise(function (resolve, reject) {
              switch (context.request.query.audience) {
                case configuration.PROFILE_AUDIENCE: {
                  switch (context.connection) {
  			  .
			  .
                  
                } break;
			  .
			  .
              
		})
      }
          .then(function () {
		.
		.
          })
          .catch(function (error) {
            callback(new UnauthorizedError(“unauthorized”), user, context);
          });
        } break;
          
        default:
          callback(null, user, context);
          break;
      
    } break; 
```
##### Redirection

Redirect from rule provides the ability for implementing custom authentication flows that require additional user interaction (i.e. beyond the standard login form) and is triggered via use of context.redirect. Redirect from rule can only be utilized when using the \authorize endpoint

Redirection to your own hosted user interface is performed before a pipeline completes, and can be triggered only once per context.clientID context. Redirection should only use HTTPS when executed in a production environment, and additional parameters should be kept to a minimum in order to help mitigate common security threats. Preferably the Auth0 supplied state is the only parameter supplied.  

Once redirected, your own hosted user interface will execute in a user authenticated context. You can obtain authenticity artefacts - i.e. an ID Token in OpenID Connect (OIDC), and/or an Access Token in OAuth 2.0 - for a context.clientID context that is not the one which triggered redirect, and this can be achieved via the use of silent authentication. This will create a new pipeline which will cause all rules to execute again, and you can use the context object within a rule to perform conditional processing (as discussed above). 

Upon completion of whatever processing is to be performed, pipeline execution continues by redirecting the user back to Auth0 via the /continue endpoint (and specifying the state supplied). This will cause all rules to execute again within the current pipeline, and you can use the context object within a rule to perform conditional processing checks.  

#### `user` object

The user object provides access to a cached copy of the user account (a.k.a. user profile) record in Auth0. The object provides access to information regarding the user without the need to access the Auth0 Management API - access which is both rate limited and subject to latency.

Whilst the contents of the user object can be modified - for example, one rule could make a change which another rule could use to influence it’s execution - any changes made will not be persisted. There may be occasions when it becomes necessary to persist updates to metadata associated with a user, and the auth0 object can be used to perform such an operation where required. 

::: note
Use of the auth0 object ultimately results in a call to the Auth0 Management API. As the Auth0 Management API is both rate limited and subject to latency, caution should be exercised regarding when and how often updates are performed.
:::

The context object contains the primaryUser property which contains the user identifier of the primary user. This user identifier will typically be the same as user_id property in the root of the user object. The primary user is the user that is returned to the Auth0 engine when the rule pipeline completes, and the user_id is a unique value generated by Auth0 to uniquely identify the user within the Auth0 tenant. The user_id should be treated as an opaque value.

There are occasions when primaryUser must be updated as the primary user may change - i.e. the user returned to the Auth0 engine will be different from the user on rule pipeline entry. Automatic account linking being one of those occasions. On these occasions, a rule must update primaryUser to reflect the new primary user identifier. Note that this change will not affect any subsequent rule executed in the current instance of the pipeline; the user object will remain unchanged.

#### Identities

The user object also contains a reference to the identities associated with the user account. The identities property is an array of objects, each of which contain properties associated with the respective identity as known to the identity provider (for example the provider name, associated connection in Auth0, and the profileData obtained from the identity provider during the last authentication using that identity). Linking user accounts creates multiple entries in the array. 

Each identity in the identities array also contains a user_id property. This property is the identifier of the user as known to the identity provider. Whilst the user_id property in the root of the user object may also include the identifier of the user (as known to the identity provider), as a best practice, use of the user_id property in an array identity should be preferred. The user_id in the root of the user object should be treated as an opaque value and should not be parsed.   

#### Metadata

The `user_metadata` property and the `app_metadata` property refer to the two different aspects of metadata associated with a user. Both the user_metadata property and the app_metadata provide access to cached copies.  

::: warning
Authorization related attributes for a user - such as role(s), group(s), department, job codes, etc - should be stored in app_metadata and not user_metadata. This is because user_metadata can essentially be modified by a user whereas app_metadata cannot.
:::

There may be occasions when it becomes necessary to persist updates to metadata associated with a user, and the auth0 object can be used to perform such an operation where required. When updating either metadata object, it is important to be judicious regarding what information is stored: in line with metadata best practice, excessive use of metadata can result in increased latency due to excessive pipeline processing. Use of the auth0 object also results in a call to the Auth0 Management API so caution should be exercised regarding when and how often updates are performed - the Auth0 Management API being both rate limited and subject to latency too.

#### `callback` function

The callback function supplied to a rule effectively acts as a signal to indicate completion of the rule. A rule should complete immediately following a call to the callback function - either implicitly, or by explicitly executing a (JavaScript) return statement - refraining from any other operation. 

Failure to call the function will result in a stall of pipeline execution, and ultimately in an error condition being returned. Each rule then must call the callback function exactly once. The callback function must be called at least once in order to prevent stall of the pipeline, however it must not be called more than once otherwise unpredictable results and/or errors will occur:

```js
  function (user, context, callback) {
  	.
	.
    // Roles should only be set to verified users.
    if (!user.email || !user.email_verified) {
      return callback(null, user, context);
    } else {
      getRoles(user.email, (err, roles) => {
        if (err) return callback(err);

        context.idToken['https://example.com/roles'] = roles;

        callback(null, user, context);
      });
	  .
	  .
    }
  }
```
As can be seen in the example provided (above), the callback function can be called with up to 3 parameters. The first parameter is mandatory and provides an indication of the status of rule operation. The second and third parameters are optional, and represent the user and the context to be supplied to the next rule in the pipeline. If these are specified, then it is a recommended best practice to pass the user and context objects (respectively) as supplied to the rule. Passing anything else will have unpredictable results, and may lead to an exception or error condition.

The status parameter should be passed as either null, an instance of an Error object, or an instance of an UnauthorizedError object. Specifying null will permit the continuation of pipeline processing, whilst any of the other values will terminate the pipeline; an UnauthorizedError signalling denial of access, and allowing information to be returned to the originator of the authentication operation regarding the reason why access is denied.  Passing any other value for any of these parameters will have unpredictable results, and may lead to an exception or error condition.  

::: note
The example provided (above) also demonstrates best practice use of both early exit as well as email address verification, as described in the Performance and Security sections below. Note: the getRoles function is implemented elsewhere within the rule as a wrapper function to a 3rd party API.
:::