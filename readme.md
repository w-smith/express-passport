<!--Had to start at 1:30 -->

<!--12:10 5 minutes -->

<!--Hook: -->

# Local Authentication with Express and Passport

## Learning Objectives
*After this class, students should be able to:*
- **Create** a login form with email & password
- **Use** passport-local to find a user & verify their password
- **Restrict** access by authenticating a user

## Preparation
*Before this class, students should be able to:*
- **Create** an express application and add CRUD/REST resources
- **Create** a Mongoose Model
- **Describe** an authentication model

<!--12:15 5 minutes -->
## Passport - Intro

From the [passport website](http://passportjs.org/docs):

"_Passport is authentication Middleware for Node. It is designed to serve a singular purpose: authenticate requests. When writing modules, encapsulation is a virtue, so Passport delegates all other functionality to the application. This separation of concerns keeps code clean and maintainable, and makes Passport extremely easy to integrate into an application._

_In modern web applications, authentication can take a variety of forms. Traditionally, users log in by providing a username and password. With the rise of social networking, single sign-on using an OAuth provider such as Facebook or Twitter has become a popular authentication method. Services that expose an API often require token-based credentials to protect access._"

### Strategies

The main concept when using passport is to register _Strategies_.  A strategy is passport Middleware that will run an authentication action in the background and then execute a callback; the callback will be called with different arguments depending on whether the action that has been performed in the strategy was successful or not. Based on this and on some config params, passport will redirect the request to different paths.  

For instance, if login is not successful, passport may redirect to the `/login` page.  If successful, it may redirect to the homepage.

<!--12:20 5 minutes -->
## Implementing Passport.js - Catch-up

#### Setup/Review Starter Code

First, fork and clone the starter code, and `npm install` to ensure that we have all of the correct dependencies.

The starter-code is structured like this:

```
.
└── app
    ├── app.js
    ├── config
    │   ├── passport.js
    │   └── routes.js
    ├── controllers
    │   └── users.js
    │   └── statics.js
    ├── models
    │   └── user.js
    ├── package.json
    ├── public
    │   └── css
    │       └── bootstrap.min.css
    └── views
    	└── partials
	    ├── header.ejs
	    └── footer.js
        ├── index.ejs
        ├── login.ejs
        ├── secret.ejs
        └── signup.ejs

7 directories, 12 files
```

Now let's open the code up in Sublime.

#### Users & Statics Controller

Let's have a quick look at the `users.js` controller. As you can see, the file has 6 empty route handlers:

```
// GET /signup
// POST /signup
// GET /login
// POST /login
// GET /logout
// Restricted page
```

The `statics.js` controller just has the home action.

#### Routes.js

We have separated the routes into their own `routes.js` file in the `config` folder.

<!--1:30 25 minutes -->

#### Signup

First we will implement the signup logic. For this, we will have:

1. a route action to display the signup form
2. a route action to receive the params sent by the form

<!--CFU: What are the restful names for these routes? -->

When the server receives the signup params, the job of saving the user data into the database, hashing the password and validating the data will be delegated to the strategy allocated for this part of the authentication. This logic will be written in `config/passport.js`

Open the file `config/passport.js` and add:

<!--Point out that we are creating a local strategy, but they may use FB, Google, or another 3rd party for Project 2 auth which would be a different strategy-->

```javascript
var LocalStrategy   = require('passport-local').Strategy;
 var User            = require('../models/user');

 module.exports = function(passport) {
   passport.use('local-signup', new LocalStrategy({
     usernameField : 'email',
     passwordField : 'password',
     passReqToCallback : true
   }, function(req, email, password, callback) {

   }));
};
```

Here we are declaring the strategy for the signup - the first argument given to `LocalStrategy` is an object giving info about the fields we will use for the authentication.

By default, passport-local expects to find the fields `username` and `password` in the request. If you use different field names, as we are doing, you can give this information to `LocalStrategy`.

The third property will tell the strategy to send the request object to the callback so that we can do further things with it.

Then, we pass the function that we want to be executed as a callback when this strategy is called: this callback method will receive the request object; the values corresponding to the authentication fields; and the callback method to execute when this 'strategy' is done.

Now, inside this callback method, we will implement our custom logic to signup a user.

```javascript
  ...
  }, function(req, email, password, callback) {
    // Find a user with this e-mail
    User.findOne({ 'local.email' :  email }, function(err, user) {
      if (err) return callback(err);

      // If there already is a user with this email
      if (user) {
	return callback(null, false, req.flash('signupMessage', 'This email is already used.'));
      } else {
      // There is no user registered with this email
	// Create a new user
	var newUser            = new User();
	newUser.local.email    = email;
	newUser.local.password = newUser.encrypt(password);

	newUser.save(function(err) {
	  if (err) throw err;
	  return callback(null, newUser);
	});
      }
    });
  }));
  ....

```

First we try to find the email in the database.

Once we have the result of this mongo request, we will check if a user document is returned - meaning that a user with this email already exists.  In this case, we will call the `callback` method with the two arguments `null` and `false` - the first argument indicates no server error happened; the second one corresponds to a new user object, which in this case hasn't been created, so we return `false`.

If no user is returned, it means that the email received in the request can be used to create a new `user` object. We will, therefore, create a new `user` object, hash the password, and save the new user object to our mongo collection. When all this is finished, we will call the `callback` method with the two arguments: `null` (no server error) and the new `user` object created.

Based on the second argument (`false` or a `user` object), passport will know if the strategy has been successfully executed, and if the request should redirect to the `success` or `failure` path. (see below).

#### User.js

The last thing is to add the method `encrypt` to the user model to hash the password received and save it as encrypted:

```javascript
  User.methods.encrypt = function(password) {
    return bcrypt.hashSync(password, bcrypt.genSaltSync(8), null);
  };
```

As we mentioned in the previous lesson, we generate a salt token and then hash the password using this new salt.

That's all for the signup strategy.

#### Route Handler

Now we need to use this strategy in the route handler.

In the `users.js` controller, for the method `postSignup`, we will add the call to the strategy we've declared

```javascript
  function postSignup(request, response) {
    var signupStrategy = passport.authenticate('local-signup', {
      successRedirect : '/',
      failureRedirect : '/signup',
      failureFlash : true
    });

    return signupStrategy(request, response);
  }
```

Here we are calling the method `authenticate` (given to us by passport) and then telling passport which strategy (`'local-signup'`) to use.

The second argument tells passport what to do in case of a success or failure.

- If the authentication was successful, then the response will redirect to `/`
- In case of failure, the response will redirect back to the form `/signup`


#### Session

We have talked briefly before about cookies.  Authentication is based on a value stored in a cookie in the browser. This cookie is sent to the server for every request until the session expires or is destroyed. This is a form of [serialization](https://en.wikipedia.org/wiki/Serialization).

To use the session with passport, we need to create two new methods in `config/passport.js` :

```javascript
  module.exports = function(passport) {

    passport.serializeUser(function(user, callback) {
      callback(null, user.id);
    });

    passport.deserializeUser(function(id, callback) {
      User.findById(id, function(err, user) {
          callback(err, user);
      });
    });
  ...

```

What exactly are we doing here? To keep a user logged in, we will need to serialize their user.id to save it to their session. Then, whenever we want to check whether a user is logged in, we will need to deserialize that information from their session, and check to see whether the deserialized information matches a user in our database.

The method `serializeUser` will be used when a user signs in or signs up, passport will call this method, our code will then call the `done` callback, the second argument is what we want to be serialized.

The second method will then be called every time there is a value for passport in the session cookie. In this method, we will receive the value stored in the cookie, in our case the `user.id`, then search for a user using this ID and then call the callback. The user object will then be stored in the request object passed to all controller methods calls.

<!--1:55 5 minutes -->

## Flash Messages - Intro

Flash messages are one-time messages that are rendered in views. When the page is reloaded, the flash is destroyed.  

In our current Node app, back when we created the signup strategy, in the callback we had this code:

```javascript
  req.flash('signupMessage', 'This email is already used.')
```

This will store the message 'This email is already used.' into the response object and then we will be able to use it in the views. This is really useful to send back details about the process happening on the server to the client.

<!--2:00 5 minutes -->

## Incorporating Flash Messages - Codealong

In the view `signup.ejs`, before the form, add:

```ejs
  <% if (message) %>
    <div class="alert alert-danger"><%= message %></div>
```

Finally, we need to render this template when we go to the `'/signup'` page, so let's add some code into `getSignup` in the `users` Controller:

```javascript
  function getSignup(request, response) {
    response.render('signup.ejs', { message: request.flash('signupMessage') });
  }
```

Now, let's start up the app using `nodemon app.js` and visit `http://localhost:3000/signup` and try to sign up two times with the same email. We should see the message "This email is already used." appearing when the form is reloaded.

<!-- 2:05 5 minutes -->

## Test it out - Independent Practice

All the logic for the signup is now set - you should be able to go to `/signup` and create a user.

<!-- 2:10 15 minutes -->

## Sign-in - Codealong

Now we need to write the `signin` logic.

We also need to implement a custom strategy for the login. In `passport.js`, after the signup strategy, add a new LocalStrategy:

```javascript
  passport.use('local-login', new LocalStrategy({
    usernameField : 'email',
    passwordField : 'password',
    passReqToCallback : true
  }, function(req, email, password, callback) {

  }));
```

The first argument is the same as for the signup strategy - we ask passport to recognize the fields `email` and `password` and to pass the request to the callback function.

For this strategy, we will search for a user document using the email received in the request. If a user is found, we will try to compare the hashed password stored in the database to the one received in the request params. If they are equal, then the user is authenticated; if not, then the password is wrong.

Inside `config/passport.js` let's add this code:

```javascript
  ...
  }, function(req, email, password, callback) {

    // Search for a user with this email
    User.findOne({ 'local.email' :  email }, function(err, user) {
      if (err) {
        return callback(err);
      }

      // If no user is found
      if (!user) {
        return callback(null, false, req.flash('loginMessage', 'No user found.'));
      }
      // Wrong password
      if (!user.validPassword(password)) {
        return callback(null, false, req.flash('loginMessage', 'Oops! Wrong password.'));
      }

      return callback(null, user);
    });
  }));
  ...

```

#### User validate method

We need to add a new method to the user schema in `user.js` so that we can use the method `user.validatePassword()`. Let's add:

```javascript
  User.methods.validPassword = function(password) {
    return bcrypt.compareSync(password, this.local.password);
  };
```

#### Adding flash messages to the view

As we are again using flash messages, we will need to add some code to display them in the view:

In `login.ejs`, add the same code that we added in `signup.ejs` to display the flash messages:

```javascript
  <% if (message) %>
    <div class="alert alert-danger"><%= message %></div>
```

#### Login GET Route handler

Now, let's add the code to render the login form in the `getLogin` action in the controller (`users.js`):

```javascript
  function getLogin(request, response) {
    response.render('login.ejs', { message: request.flash('loginMessage') });
  }
```

You'll notice that the flash message has a different name (`loginMessage`) than the in the signup route handler.

#### Login POST Route handler

We also need to have a route handler that deals with the login form after we have submitted it. So in `users.js` lets also add:

```javascript
  function postLogin(request, response) {
    var loginProperty = passport.authenticate('local-login', {
      successRedirect : '/',
      failureRedirect : '/login',
      failureFlash : true
    });

    return loginProperty(request, response);
  }
```

You should be able to login now!

<!--2:25 5 minutes -->

## Test it out - Independent Practice

#### Invalid Login

First try to login with:

- an invalid email (one that hasn't been signed up yet)
- an invalid password

<!--And what should they see the first time? -->

You should see the message 'Oops! Wrong password.' the second time through.

#### Valid Login

Now, try to login with valid details and you should be taken to the index page with a message of "Welcome".

The login strategy has now been setup!


#### Accessing the User object globally

By default, passport will make the user available on the object `request`. In most cases, we want to be able to use the user object everywhere. For that, we're going to add some middleware in `app.js` underneath our passport require statement:

```javascript
  require('./config/passport')(passport);

  app.use(function (req, res, next) {
    res.locals.currentUser = req.user;
    next();
  });
```

Now in the layout, we can add:

```javascript
<ul>
  <% if (currentUser) {%>
    <li><a href="/logout">Logout <%= currentUser.local.email %></a></li>
  <% } else { %>
    <li><a href="/login">Login</a></li>
    <li><a href="/signup">Signup</a></li> 
  <% } %>
</ul>
```

<!--2:30 5 minutes -->

## Signout - Codealong

#### Logout

The last action to implement for our authentication system is to set the logout route and functionality.

In `controllers/users.js`:
```js
function getLogout(request, response) {
  request.logout();
  response.redirect('/');
}
```

<!--2:35 5 minutes -->

## Test it out - Independent Practice

You should now be able to login and logout! Test this out.

<!--2:40 10 minutes -->

## Restricting access

As you know, an authentication system is used to allow/deny access to some resources to authenticated users.

Let's now turn our attention to the `secret` route handler and its associated template.

To restrict access to this route, we're going to add a method at the top of `config/routes.js`:

```javascript
  function authenticatedUser(req, res, next) {
    // If the user is authenticated, then we continue the execution
    if (req.isAuthenticated()) return next();

    // Otherwise the request is always redirected to the home page
    res.redirect('/');
  }
```

Now, when we want to "secure" access to a particular route, we will add a call to the method in the route definition.

For the `/secret` route, we need to add this to the `/config/routes.js` file:

```javascript
  router.route("/secret")
    .get(authenticatedUser, usersController.secret)
```

Now every time the route `/secret` is called, the method `authenticatedUser` will be executed first. In this method, we either redirect to the homepage or go to the next method to execute.

Now test it out by clicking on the secret page link. You should see: "This page can only be accessed by authenticated users"

### Final Challenge

Go into the `controllers.js` file and add a super-secret JSON response to the `secret` function.  Now try logging in and getting to the super-secret JSON message.

<!-- 2:50 5 minutes -->

## Conclusion

Passport is a really useful tool because it allows developers to abstract the logic of authentication and customize it, if needed. It comes with a lot of extensions that we will cover later.

- What does it mean for a user to be "logged in"?
- Briefly describe the authentication process using passport in Express.

## Resources

- [Passport Docs](http://passportjs.org/docs)
