[![Torii Build Status](https://circleci.com/gh/Vestorly/torii.png?circle-token=9bdd2f37dbcb0be85f82a6b1ac61b9333b68625b "Torii Build Status")](https://circleci.com/gh/Vestorly/torii)

Torii is a set of clean abstractions for authentication in [Ember.js](http://emberjs.com/)
applications. Torii is built with **endpoints** (authentication against a platform), a
**session manager** (for maintaining the current user), and **adapters** (to persist
authentication state).

The API for endpoints and adapters in Torii is to **open**, by which we mean creating a new
authorization or authenticating a new session, **fetch**, by which we mean validating
and existing authorization (like a session stored in cookies), or **close**, where an
authorization is destroyed.

An endpoint in Torii is anything a user can authenticate against. This could be an
OAuth 2.0 endpoint, your own login mechanism, or an SDK like Facebook Connect.
Authenticating against an **endpoint** is done via the `torii` property, which is injected
on to routes:

```hbs
{{! app/templates/post.hbs }}
{{if hasFacebook}}
  {{partial "comment-form"}}
{{else}}
  <a href="#" {{action 'signInToComment'}}>
    Sign in to comment
  </a>
{{/if}}
```

```JavaScript
// app/routes/post.js
export default Ember.Route.extend({
  actions: {
    signInToComment: function(){
      var controller = this.controllerFor('post');
      // The endpoint name is passed to `open`
      this.get('torii').open('facebook-connect').then(function(authorization){
        // FB.api is now available. authorization contains the UID and
        // accessToken.
        controller.set('hasFacebook', true);
      });
    }
  }
});
```

```
torii.open('facebook') -> #open hook on the facebook endpoint -> returned authorization
```

This is authentication only against an endpoint. If your application provides
an **adapter**, then Torii can also peform **session management** via the
`session` property, injected onto routes and controllers. This example uses
Facebook's OAuth 2.0 API directly to fetch an authorization code.

```hbs
{{! app/templates/login.hbs }}
{{if session.isWorking}}
  One sec while we get you signed in...
{{else}}
  {{error}}
  <a href="#" {{action 'signInViaFacebook'}}>
    Sign In with Facebook
  </a>
{{/if}}
```

```JavaScript
// app/routes/login.js
export default Ember.Route.extend({
  actions: {
    signInViaFacebook: function(){
      var route = this,
          controller = this.controllerFor('login');
      // The endpoint name is passed to `open`
      this.get('session').open('facebook-oauth2').then(function(){
        route.transitionTo('dashboard');
      }, function(error){
        controller.set('error', 'Could not sign you in: '+error.message);
      });
    }
  }
});
```

```JavaScript
// app/torii-adapters/application.js
export default Ember.Object.extend({
  open: function(authentication){
    var authorizationCode = authentication.get('authorizationCode');
    return new Ember.RSVP.Promise(function(resolve, reject){
      Ember.$.ajax({
        url: 'api/session',
        data: { 'facebook-auth-code': authorizationCode },
        dataType: 'json',
        success: Ember.run.bind(null, resolve),
        error: Ember.run.bind(null, reject)
      });
    }).then(function(user){
      // The returned object is merged onto the session (basically). Here
      // you may also want to persist the new session with cookies or via
      // localStorage.
      return {
        currentUser: user
      };
    });
  }
});
```

```
session.open('facebook') -> #open hook on the facebook endpoint -> #open hook on the application adapter -> updated session
```

Note that the adapter section is left entirely to your application.

## Using Torii

Using Torii currently requires an AMD-compatible module loader. [Ember-App Kit](https://github.com/stefanpenner/ember-app-kit)
and [Ember-CLI](http://iamstef.net/ember-cli/) provide this out of the box.

First, **obtain a distribution version of torii**:

While the torii repo is still private, to obtain a distribution version of torii you'll need to either
get access to the repository, pull the code and build a dist yourself, or obtain the amd build from a friend.

To build a dist version of torii yourself, assuming you have access to the repo, you'll want to clone the repo and
build torii:

```
git clone git@github.com/Vestorly/torii.git
cd torii
npm install
grunt build:dist
```

Copy `dist/torii.amd.js` to `vendor/torii/index.js` in your ember-cli project.

If you have a built version of torii you can skip to the step of copying it into your project as described above.

Next, **add Torii to your build pipeline**. In Ember-App-Kit you do this
in `app/index.html`. In Ember-CLI, you add the package to the `Brocfile.js`:

```
app.import('vendor/torii/index.js', {
  // If you want to subclass a Torii class in your application, whitelist
  // that module here. For instance, the base endpoint class:
  //
  // 'torii/endpoints/base': ['default']
  //
  // See http://iamstef.net/ember-cli/#managing-dependencies
});
```

**Add Torii's intialization code to your app**. Torii exports an amd module named `torii/ember`, which will
add the appropriate application initializers to do Torii's container registrations and injections.
You will want to add `require('torii/ember');` to your `app.js` file after you've defined your app.
Here is an [example app.js](https://gist.github.com/bantic/b86787ed315c5ef98323).

**Configure a Torii endpoint**. Here, we set the `window.ENV` with a
configuration for the Facebook Connect endpoint:

```JavaScript
// In Ember-App-Kit you will set this in app/index.html
// In Ember-CLI you will set these values in config/environment.js
window.ENV = window.ENV || {};
window.ENV['torii'] = {
  endpoints: {
    'facebook-connect': {
      appId: 'xxxxx-some-app-id',
      scope: 'email,birthday'
    }
  }
};
```

With those values, we can authenticate the user against Facebook Connect
via the `torii` property injected onto _routes_, or the `session` property
injected onto routes and controllers (using the session management feature
will require you to write an adapter for your application – see notes on session management below).

## Endpoints in Torii

### Writing an endpoint

Endpoints have two hooks that may be implemented. Each *must* return a
promise:

* `open` must create a new authorization. An example of this is logging a
  user in with their username and password.
* `fetch` must refresh an existing authorization. An example of this is
  confirming an access token stored in a session.

Torii will lookup endpoints in the Ember application container, so if you
name them conventionally (put the in the `app/torii-endpoints` directory)
they will be available automatically.

A minimal endpoint:

```JavaScript
// app/torii-endpoints/geocities.js
export default Ember.Object.extend({
  // create a new authorization
  open: function(options) {
    return new Ember.RSVP.Promise(function(resolve, reject){
      // resolve with an authorization object
    });
  },
  // refresh or confirm an authorization
  fetch: function(options) {
    return new Ember.RSVP.Promise(function(resolve, reject){
      // resolve with an authorization object
    });
  }
});
```

Endpoint hooks should return a promise resolving with an authorization
object. Authorization objects should include values like access tokens, or
an Ember-Data model representing a session, or minimal user data like UIDs.
They may return SDK objects, such as an object with an API for making
authenticated calls to an API.

When used via `torii.open`, the authorization object is passed through to
the consumer. An example:

```JavaScript
// app/torii-endpoints/geocities.js
export default Ember.Object.extend({
  // credentials as passed from torii.open
  open: function(credentials){
    return new Ember.RSVP.Promise(function(resolve, reject){
      exampleAsyncLogin(
        credentials.username,
        credentials.password,
        // the promise is resolved with the authorization
        Ember.run.bind(null, resolve, {
          sessionToken: response.token
        });
      );
    });
  }
});
```

```JavaScript
// app/routes/application.js
export default Ember.Route.extend({
  actions: {
    openGeocities: function(username, password){
      var route = this;
      // argument to open is passed into the endpoint
      this.get('torii').open('geocities', {
        username: username,
        password: password
      }).then(function(authorization){
        // authorization as returned by the endpoint
        route.somethingWithGeocitiesToken(authorization.sessionToken);
      });
    }
  }
});
```

The cornerstone of many Torii endpoints is the `popup` object, which is injected
onto all endpoints.

### Built-in endpoints

A minimal endpoint:

Torii comes with several endpoints already included:

  * LinkedIn OAuth2 ([Dev Site](https://www.linkedin.com/secure/developer) | [Docs](http://developer.linkedin.com/))
  * Google OAuth2 ([Dev Site](https://console.developers.google.com/project) | [Docs](https://developers.google.com/accounts/docs/OAuth2WebServer))
  * Facebook Connect (via FB SDK) ([Dev Site](https://developers.facebook.com/) | [Docs](https://developers.facebook.com/docs/))
  * Facebook OAuth2 ([Dev Site](https://developers.facebook.com/) | [Docs](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow/))
  * **Authoring custom endpoints is designed to be easy** - You are encouraged to author your own.

### Supporting OAuth 1.0a

OAuth 1.0a, used by Twitter and some other organizations, requires a significant
server-side component and so cannot be supported out of the box. It can be implemented
following these steps:

  1. Torii endpoint opens a popup to the app server asking for Twitter auth
  2. Server redirects to Twitter with the credentials for login
  3. User enters their credentials at Twitter
  4. Twitter redirects to app server which completes the authentication
  5. Server loads the Ember application with a message in the URL, or otherwise
     transmits a message back to the parent window.
  6. Ember application in the initial window closes the popup and resolves its
     endpoint promise.

## Session Management in Torii

If you want to use torii's session management state machine, you must opt in to it via the torii configuration.
Because of the potential for conflicts, **torii will not inject a `session` property** unless you explicitly ask for
it in your configuration. To do so, specify a `sessionServiceName` in your torii config, example:

```
window.ENV = window.ENV || {};
window.ENV['torii'] = {
  sessionServiceName: 'session', // a 'session' property will be injected on routes and controllers

  // ... additional configuration for endpoints, etc
};
```

Read on about adapters for more information on using torii's session management.

## Adapters in Torii

Adapters in Torii process authorizations and pass data to the session. For
example, an endpoint might create an authorization object from the Facebook
OAuth 2.0 API, then create a session on your applications server. The adapter
would then fetch the user and resolve with that value, adding it to the
`sessions` object.

Again, adapters are looked up on the container, and so if you name them
conventionally (put the in `app/torii-adapters/`) then they are loaded
automatically.

Adapters have three hooks that may be implemented. Each *must* return a
promise:

* `open` - a new session
* `fetch` - a refreshed session
* `close` - a closing session

Adapters are flexible, but a common use would be to fetch a current user
for the session. By default, the `application` adapter will handle all
authorizations. An example application adapter with an `open` hook:

```JavaScript
// app/torii-adapters/application.js
//
// Here we will presume the store has been injected onto torii-adapter
// factories. You would do this with an initializer, and:
//
// application.inject('torii-adapter', 'store', store:main');
//
export default Ember.Object.extend({
  open: function(authorization){
    var userId = authorization.user,
        store  = this.get('store');
    return store.find('user', userId).then(function(user){
      return {
        currentUser: user
      };
    });
  }
});
```

The object containing the `currentUser` is merged onto the session. Because the
session is injected onto controllers and routes, these values will be available
to templates.

Torii will first look for an adapter matching the endpoint name passed to
`torii.open`. If there is no matching adapter, then the session object will
fall back to using the `application` adapter.

## Getting started with the Torii codebase

  * Clone the repo `git clone git@github.com:Vestorly/torii.git`, `cd torii/`
  * `npm install`, which will also run `bower install`
  * `grunt test` for tests.
  * Start the server: `grunt server`
  * Open [http://localhost:8000/example](http://localhost:8000/example) for example usage.

## Generate docs

Use [YUIDoc](http://yui.github.io/yuidoc/).

  * Install: `npm install -g yuidocjs`
  * Generate: `yuidoc lib/`
  * Output will be put into "docs/"

*Initial development of Torii was generously funded by [Vestorly](https://www.vestorly.com/). Vestorly is a technology company solving the client acquisition problem for professionals in wealth management, and the enterprises that support them. Vestorly's user interface is built entirely with Ember.js and modern web technologies. [jobs@vestorly.com](jobs@vestorly.com)*

## How to help

Torii aims to provide a flexible set of primitives for creating your application' own
authentication solution. There are still a few things we could use help with:

* A non-AMD build of the code
* More testing of sessions
* More documentation
* Publish your own endpoint or adapter implementation!

We welcome your contributions.
