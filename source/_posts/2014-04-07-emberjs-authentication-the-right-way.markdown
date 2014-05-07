---
layout: post
title: "Ember.js Authentication - the right way"
date: 2014-04-07 10:24:29 +0100
comments: true
categories: [Coffeescript, Ember.js, Authentication]
---

We all know that Ember.js is a great example of how using a front end framework can make your development workflow more powerful. But to use it correctly means to make your life easier.

There are some great articles demonstrating how you can setup your first EmberJS application, but now you have came to a decisive point, user authentication.

<!--more-->

![Ember.js Authentication - the right way](http://i.imgur.com/BevCJ3W.png)

<div class="info warn">
Not into Coffescript? Want the Javascript version? <a href="{% post_url 2014-04-07-emberjs-authentication-the-right-way-javascript-version %}">Click here</a>
</div>

# So you want to authenticate users with Ember.js
EmberJS's built in objects and structure are great, to use them correctly is to enjoy all it's power. So you wonder, what is the best way to use Ember.js's and it's well grounded structure to build your user's authentication?

# The API
This article will not cover the authentication api part, because it's already well covered on good articles like codeberry's [Authentication With EmberJS - Part 1](http://coderberry.me/blog/2013/07/08/authentication-with-emberjs-part-1/), which we will use in this example, but we'll use the `rails` gem instead of the `rails-api` gem to make use of the assets pipeline.

The part two on that article deals with the authentication task through EmberJS with an AuthManager component, which is clever idea, but it follows a path that leads to deal with the authentication part as a component, which leads to step outside of the framework's path in some cases, like accessing the store object outside of a route or controller. And it can lead to some problems or difficulties in the future.

# Setting up your project
All the code for this article can be found on [EmberJS-Auth-Example](https://github.com/WebCloud/EmberJS-Auth-Example) github project, api and front end code. Before running, make sure you have ruby 2 and bundler installed. Then, to run the project from the project's folder simply run:
    bundle && rake db:migrate && rails s

You will then be able to access <http://localhost:3000>

# First things first, bootstrap your Ember.js app
Now, lets first create our application's main object with the following code:

{% codeblock app/assets/javascript/application.js.coffee lang:coffeescript start:14 %}
window.Auth = Ember.Application.create
  LOG_TRANSITIONS:          true,
  LOG_TRANSITIONS_INTERNAL: true
{% endcodeblock %}

To make our development and debugging easier, It's defined that we will see all our transitions logs on the console, to help you even further I sugges to install the [Ember Inspector](https://chrome.google.com/webstore/detail/ember-inspector/bmdblncegkenkacieihfhpjfppoconhi?hl=en).

After creating the base application object let's create our routes:
{% codeblock app/assets/javascript/router.js.coffee lang:coffeescript start:3 %}
Auth.Router.map ()->
  @resource('sessions')
  @resource('users', ()->
    @route('signup')
    @route('user', { path:'/user/:user_id' })
    return
  )
  @route('secret')
  return
{% endcodeblock %}

The idea is to have the `@route('secret')` only available to authenticated users.

# Using the REST adapter
We will use the RESTAdapter provided by Ember Data (__Ember.DS__) to make our comunication with our REST api much easier and simple. To do so, [overwrite the default](http://emberjs.com/guides/getting-started/using-other-adapters/) adapter:

{% codeblock app/assets/javascript/store.js.coffee lang:coffeescript %}
Auth.ApplicationAdapter = DS.RESTAdapter.extend()
{% endcodeblock %}


# Building your models
The idea is to have the Ember application communicating to the api sending a access_token to validate the authenticated user's session on the protected urls. To do so, we will have two models, User and ApiKey:

{% codeblock app/assets/javascript/models/user.js.coffee lang:coffeescript %}
Auth.User = DS.Model.extend
  name:                  DS.attr('string'),
  email:                 DS.attr('string'),
  username:              DS.attr('string'),
  password:              DS.attr('string'),
  password_confirmation: DS.attr('string'),
  apiKeys:               DS.hasMany('apiKey'),
  errors:                {}
{% endcodeblock %}

{% codeblock app/assets/javascript/models/api_key.js.coffee lang:coffeescript %}
Auth.ApiKey = DS.Model.extend
  accessToken:  DS.attr('string'),
  user:         DS.belongsTo('user', {async:true})
{% endcodeblock %}

The ApiKey model won't communicate with the api, because the only time it's really requested is on the login action. So we will overwrite the rest adapter for that model, using local storage:

{% codeblock app/assets/javascript/store.js.coffee lang:coffeescript start:10 %}
Auth.ApiKeyAdapter = DS.LSAdapter.extend
  namespace: 'emberauth-keys'
{% endcodeblock %}


# Building the Sessions controller
This controller should authenticate any user with a valid login and password through our api and store the access_key to be used on our protected pages. Based on that let's build our login action.

First let's get our data provided from the form and send an POST request to our /sessions api:
{% codeblock loginUser action on app/assets/javascript/controllers/sessions_controller.js.coffee lang:coffeescript start:47 %}
      # get the properties sent from the form and if there is any attemptedTransition set
      data           = @getProperties('username_or_email', 'password')
      attemptedTrans = @get('attemptedTransition')

      # clear the form fields
      @setProperties
        username_or_email: null,
        password:          null

      # send a POST request to the /sessions api with the form data
      Ember.$.post('/session', data).then((response)=>
        # set the ajax header with the returned access_token object
        Ember.$.ajaxSetup
          headers: { 'Authorization': 'Bearer ' + response.api_key.access_token }

        return
      , (error)->
        # if there is a authentication error the user is informed of it to try again
        if error.status is 401
          alert("wrong user or password, please try again")
        return
      )
{% endcodeblock %}

You can notice that it's being used the `Ember.$.post` method, that's no more than a wrapper to the jQuery's `$.post()` method but it returns a Ember promisse back to us, to know more about Ember Promssies see Ember's [Asynchronous Routing](http://emberjs.com/guides/routing/asynchronous-routing/) guide. If the user is signed in we set the ajax headers to carry our access_token returned from the api, and if the user has a invalid login/password combination we send an alert message.

After the login, we should fetch our user's data and link the user to the returned access_token with a ApiKey record, to than take our user to the secret route. So let's expand our authentication action with the following code:
{% codeblock loginUser action on app/assets/javascript/controllers/sessions_controller.js.coffee lang:coffeescript start:62 %}
      # create a apiKey record on the local storage based on the returned object
      key = @get('store').createRecord('apiKey', {accessToken: response.api_key.access_token})

      # find a user based on the user_id returned from the request to the /sessions api
      @store.find('user', response.api_key.user_id).then((user)=>
        # set this controller token & current user properties
        # based on the data from the user and access_token
        @setProperties
          token:       response.api_key.access_token,
          currentUser: user.getProperties('username','name','email')

        # set the relationship between the User and the ApiKey models & save the apiKey object
        key.set('user', user)
        key.save()

        user.get('apiKeys').content.push(key)

        # check if there is any attemptedTransition to retry it or go to the secret route
        if attemptedTrans
          attemptedTrans.retry()
          @set('attemptedTransition', null)
        else
          @transitionToRoute('secret')
{% endcodeblock %}

You can notice that we also set our controller's token and currentUser properties with the data returned from the login action:
    @setProperties
          token:       response.api_key.access_token,
          currentUser: user.getProperties('username','name','email')
That will be used to future verifications from the protected pages to proceed or not with the requests that comes to them and print out our user data.

We will also have to provide a way to the application reset our `SessionController` properties that contains user information when he logs out:

{% codeblock reset function on app/assets/javascript/controllers/sessions_controller.js.coffee lang:coffeescript start:33 %}
  # reset the controller properties and the ajax header
  reset:()->
    @setProperties
      username_or_email: null,
      password:          null,
      token:             null,
      currentUser:       null
    Ember.$.ajaxSetup
      headers: { 'Authorization': 'Bearer none' }
    return
{% endcodeblock %}

It is a good idea to store the authenticated user's data either on local storage or cookies to allow them to keep their session if they close the browser window/tab. In this example we will go with cookies and the jQuery cookie plugin.

{% codeblock tokenChanged on app/assets/javascript/controllers/sessions_controller.js.coffee lang:coffeescript start:21 %}
  # create a observer binded to the token property of this controller
  # to set/remove the authentication tokens
  tokenChanged: (()->
    if Ember.isEmpty(@get('token'))
      Ember.$.removeCookie('access_token')
      Ember.$.removeCookie('auth_user')
    else
      Ember.$.cookie('access_token', @get('token'))
      Ember.$.cookie('auth_user', @get('currentUser'))
    return
  ).observes('token')
{% endcodeblock %}

It's created an observer binded to the token property of the SessionsController object, to automatically remove or instantiate the cookies based on the values of that property.

And finally let's bootstrap our SessionController so when the user tries to access it's route (either being redirected or directly visiting it's path) he can instantiate the properties with the cookies value, avoiding having to re-login if he had a previous session saved.

{% codeblock app/assets/javascript/controllers/sessions_controller.js.coffee lang:coffeescript start:4 %}
  # initialization method to verify if there is a access_token cookie set
  # so we can set our ajax header with it to access the protected pages
  init:()->
    @_super()
    if Ember.$.cookie('access_token')
      Ember.$.ajaxSetup
        headers: { 'Authorization': 'Bearer ' + Ember.$.cookie('access_token') }
    return
  ,

  # overwriting the default attemptedTransition attribute from the Ember.Controller object
  attemptedTransition: null,

  # create and set the token & current user objects based on the respective cookies
  token:               Ember.$.cookie('access_token'),
  currentUser:         Ember.$.cookie('auth_user'),
{% endcodeblock %}


# The ApplicationController
On our application controller we should set up some way to retrieve _application_ wide the authenticated user data, if any. So we will read from the `SessionsController` object's properties.

{% codeblock app/assets/javascript/controllers/application_controller.js.coffee lang:coffeescript start:3 %}
  Auth.ApplicationController = Ember.Controller.extend
  # requires the sessions controller
  needs: ['sessions'],

  # creates a computed property called currentUser that will be
  # binded on the curretUser of the sessions controller and will return its value
  currentUser:(()->
    @get('controllers.sessions.currentUser')
  ).property('controllers.sessions.currentUser'),

  # creates a computed property called isAuthenticated that will be
  # binded on the curretUser of the sessions controller and will verify if the object is empty
  isAuthenticated: (()->
    !Ember.isEmpty(@get('controllers.sessions.currentUser'))
  ).property('controllers.sessions.currentUser')
{% endcodeblock %}

Ember has a clever way to make different controllers comunicate between each other, that is done by the controller's dependencies. That's what we do when we decleare `needs: ['sessions']`, to read more about controlllers dependencies go to the Ember's guide to [Dependencies between controllers](http://emberjs.com/guides/controllers/dependencies-between-controllers/).

With that set up we create [computed properties](http://emberjs.com/guides/object-model/computed-properties/) that will automatically update the `ApplicationController`'s properties.

# Connecting everything with routes
Now it's time to connect all those parts overwriting the routes objects created by Ember to each `@route`or `@resource` that we declared on our `Auth.Router` object. First, let's create our global logout action to reset the `SessionsController`'s properties.

{% codeblock app/assets/javascript/routes/application_route.js.coffee lang:coffeescript %}
Auth.ApplicationRoute = Ember.Route.extend
  actions:
    # create a global logout action
    logout: ()->
      # get the sessions controller instance and reset it to then transition to the sessions route
      @controllerFor('sessions').reset()
      @transitionTo('sessions')
      return
{% endcodeblock %}

With that in mind, in our `SessionsRoute`, it would be good if we clear out the `SessionsController`'s properties each time a user visits the /sessions url to avoid showing the user left over data from the last authentication. For that, Ember's Route object has a special hook to [setup the controller](http://emberjs.com/guides/routing/setting-up-a-controller/) before the transition.

{% codeblock app/assets/javascript/routes/sessions_route.js.coffee lang:coffeescript start:5 %}
  # setup the SessionsController by resetting it to avoid data from a past authentication
  setupController: (controller, context)->
    controller.reset()
    return
  ,
{% endcodeblock %}

Also, it would be a good idea to see if the user isn't already signed, so in that case we can take our user to the secret route instead of showing the login form.

{% codeblock app/assets/javascript/routes/sessions_route.js.coffee lang:coffeescript start:9 %}
  beforeModel: (transition)->
    # before proceeding any further, verify if the token property is not empty
    # if it is, transition to the secret route
    if !Ember.isEmpty(@controllerFor('sessions').get('token'))
      @transitionTo('secret')
    return
{% endcodeblock %}

And for the cases that a unauthenticated user tries to access a protected route, we should be able to redirect him to the login route. Previewing that we might have another routes that will be for logged in users only, it would be better to have a base object to extend this basic functionality, instead of rewriting this on every protected route. Luckly enought Ember object's can be [extended/inherit from others](http://emberjs.com/guides/object-model/classes-and-instances/). So we will create a `AuthenticatedRoute` that will serve as our base route for those cases.

{% codeblock app/assets/javascript/routes/authenticated_route.js.coffee lang:coffeescript start:3 %}
# create a base object for any authentication protected route with the required verifications
Auth.AuthenticatedRoute = Ember.Route.extend
  # verify if the token property of the sessions controller is set before continuing with the request
  # if it is not, redirect to the login route (sessions)
  beforeModel: (transition)->
    if Ember.isEmpty(@controllerFor('sessions').get('token'))
      @redirectToLogin(transition)
  ,

  # Redirect to the login page and store the current transition so we can
  # run it again after login
  redirectToLogin: (transition)->
    @controllerFor('sessions').set('attemptedTransition', transition)
    @transitionTo('sessions')
  ,
{% endcodeblock %}

Based on that, we can also rescue from any error that might happen from unauthorised access to other protected routes on this route. To do so, we simply have to indicate what the default [Ember's error action](http://emberjs.com/guides/routing/loading-and-error-substates/) on that route should do.

{% codeblock app/assets/javascript/routes/authenticated_route.js.coffee lang:coffeescript start:18 %}
  actions:
    # recover from any error that may happen during the transition to this route
    error: (reason, transition)->
      # if the HTTP status is 401 (unauthorised), redirect to the login page
      if (reason.status is 401)
        @redirectToLogin(transition)
      else
        console.log('unknown problem')
      return
{% endcodeblock %}

After that, we simply make our `SecretRoute` inherit from the `AuthenticatedRoute` and define it's model to be a list of the existing users on the api.

{% codeblock app/assets/javascript/routes/secret_route.js.coffee lang:coffeescript %}
Auth.SecretRoute = Auth.AuthenticatedRoute.extend
  model: ()->
    # instantiate the model for the SecretController as a list of created users
    @store.find('user')
{% endcodeblock %}


# So long and thanks for the fish
On the [EmberJS-Auth-Example](https://github.com/WebCloud/EmberJS-Auth-Example) source code you will also find the signup process, not covered in this tutorial as it would become too extensive, feel free to browse around and change as you wish.

If you wish to learn more about Ember.js you will find some good material on the links above:

* [Ember.JS Guides](http://emberjs.com/guides/)
* [EmberWatch](http://emberwatch.com/)
* [Ember.js in Action](http://www.manning.com/skeie/)

<br />
