<!--Had to start at 1:30 -->

<!--1:33 WDI 3 -->

<!--WDI5 1:40 -->
<!--12:10 5 minutes -->
<!--12:21 WDI4 -->
<!--Hook: Try to think back to our work in Mongo and Mongoose, saving objects to our database.  Another object we may want to save is our user.  And that process is basically the same as it was with our objects.  Except for a couple pieces that make it a little harder.  Today, we'll talk about those.-->

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

<!--1:36 WDI3 -->
<!--12:25 WDI4 -->
<!--12:15 5 minutes -->
## Passport - Intro

From the [passport website](http://passportjs.org/docs):

"_Passport is authentication Middleware for Node. It is designed to serve a singular purpose: authenticate requests. When writing modules, encapsulation is a virtue, so Passport delegates all other functionality to the application. This separation of concerns keeps code clean and maintainable, and makes Passport extremely easy to integrate into an application._

_In modern web applications, authentication can take a variety of forms. Traditionally, users log in by providing a username and password. With the rise of social networking, single sign-on using an OAuth provider such as Facebook or Twitter has become a popular authentication method. Services that expose an API often require token-based credentials to protect access._"

<!--WDI5 1:45 -->
<!--1:37 actually WDI2 -->

### Strategies

The main concept when using passport is to register _Strategies_.  A strategy is passport Middleware that will run an authentication action in the background and then execute a callback; the callback will be called with different arguments depending on whether the action that has been performed in the strategy was successful or not. Based on this and on some config params, passport will redirect the request to different paths.  

For instance, if login is not successful, passport may redirect to the `/login` page.  If successful, it may redirect to the homepage.

<!--12:30 WDI4 -->
<!--12:20 5 minutes -->
## Implementing Passport.js - Catch-up

#### Setup/Review Starter Code

First, fork and clone the starter code, and `npm install` to ensure that we have all of the correct dependencies.
<!--WDI4 1:32, 1:34 turning over to devs -->
<!--1:40 actually WDI2-->

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

<!--WDI5 1:51   -->
<!--WDI3 1:42 when turning over to devs 1:45 back-->
<!--WDI4 coming back 1:36 -->

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

<!--WDI5 1:52 -->
<!--1:49 WDI3 -->
<!--WDI4 1:39 -->
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

<!--WDI4 1:45 without inside of function -->
<!--WDI5 when done with blank function 2:00  -->
<!--WDI5 when done with inside 2:08  -->
<!--WDI4 1:53 with inside of function -->

Here we are declaring the strategy for the signup - the first argument given to `LocalStrategy` is an object giving info about the fields we will use for the authentication.

By default, passport-local expects to find the fields `username` and `password` in the request. If you use different field names, as we are doing, you can give this information to `LocalStrategy`.

The third property will tell the strategy to send the request object to the callback so that we can do further things with it.

Then, we pass the function that we want to be executed as a callback when this strategy is called: this callback method will receive the request object; the values corresponding to the authentication fields; and the callback method to execute when this 'strategy' is done.

Now, inside this callback method, we will implement our custom logic to signup a user.

<!--2:02 WDI3 -->

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

<!--2:06 WDI4-->

First we try to find the email in the database.

Once we have the result of this mongo request, we will check if a user document is returned - meaning that a user with this email already exists.  In this case, we will call the `callback` method with the two arguments `null` and `false` - the first argument indicates no server error happened; the second one corresponds to a new user object, which in this case hasn't been created, so we return `false`.

If no user is returned, it means that the email received in the request can be used to create a new `user` object. We will, therefore, create a new `user` object, hash the password, and save the new user object to our mongo collection. When all this is finished, we will call the `callback` method with the two arguments: `null` (no server error) and the new `user` object created.

Based on the second argument (`false` or a `user` object), passport will know if the strategy has been successfully executed, and if the request should redirect to the `success` or `failure` path. (see below).

<!--WDI5 2:22 -->
<!--2:16 WDI 3 -->
<!--2:11 actually WDI2 -->

#### User.js

The last thing is to add the method `encrypt` to the user model to hash the password received and save it as encrypted:

```javascript
  User.methods.encrypt = function(password) {
    return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
  };
```

As we mentioned in the previous lesson, we generate a salt token and then hash the password using this new salt.

That's all for the signup strategy.

<!--WDI5 2:26  -->
<!--WDI4 2:10 -->
<!--2:22 actually WDI3 2:27 turning over to devs-->

#### Route Handler

Now we need to use this strategy in the route handler.

In the `users.js` controller, for the method `postSignup`, we will add the call to the strategy we've declared

```javascript
  function postSignup(request, response, next) {
    var signupStrategy = passport.authenticate('local-signup', {
      successRedirect : '/',
      failureRedirect : '/signup',
      failureFlash : true
    });

    return signupStrategy(request, response, next);
  }
```

Here we are calling the method `authenticate` (given to us by passport) and then telling passport which strategy (`'local-signup'`) to use.

The second argument tells passport what to do in case of a success or failure.

- If the authentication was successful, then the response will redirect to `/`
- In case of failure, the response will redirect back to the form `/signup`

<!--WDI5 2:33  -->
<!--WDI4 2:18 -->

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

The second method will then be called every time there is a value for passport in the session cookie. In this method, we will receive the value stored in the cookie, in our case the `user.id`, then search for a user using this ID and then call the callback. The user object will then be stored in the request object passed to all controller method calls.

<!--WDI5 2:40  -->
<!-- Actually 2:25 -->
<!--WDI4 2:22 turning over to devs -->
<!--WDI4 2:26 -->
<!--WDI4 by the time we debugged all but one (goddammit, why all but one?) issue it was 3:30, with break time 3:02-3:15-->

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
  <% if (message.length > 0) { %>
    <div class="alert alert-danger"><%= message %></div>
  <% } %>
```

<!--WDI4 2:45  -->

Finally, we need to render this template when we go to the `'/signup'` page, so let's add some code into `getSignup` in the `users` Controller:

```javascript
  function getSignup(request, response, next) {
    response.render('signup.ejs', { message: request.flash('signupMessage') });
  }
```

Now, let's start up the app using `nodemon app.js` and visit `http://localhost:3000/signup` and try to sign up two times with the same email. We should see the message "This email is already used." appearing when the form is reloaded.

<!--WDI5 2:48 -->
<!-- 2:05 5 minutes -->

## Test it out - Independent Practice

All the logic for the signup is now set - you should be able to go to `/signup` and create a user.

<!--Would probably be good to walk through what each file did in the chain before moving on to login-->

<!-- 2:47 WDI2 turning over to devs -->
<!--3:44 WDI4 -->

<!--WDI5 after break and resolving issues 3:16  -->
<!--3:10 after break and resolving testing issues WDI3 -->
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

<!--WDI5 3:21  -->
<!--3:46 WDI4 turning over to devs -->
<!--3:48 WDI4 coming back -->

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

<!--WDI5 3:30 -->
<!--WDI3 3:21 turning over to devs, return 3:27-->

#### User validate method

We need to add a new method to the user schema in `user.js` so that we can use the method `user.validatePassword()`. Let's add:

```javascript
  User.methods.validPassword = function(password) {
    return bcrypt.compareSync(password, this.local.password);
  };
```

<!--WDI5 3:34 -->
<!--3:51 WDI4 turning over to devs -->
<!--3:56 WDI4 coming back -->
<!--WDI4 Just Flew Through steps...will come back to it for second pass tomorrow -->

<!--3:29 to devs 3:31 return -->

#### Adding flash messages to the view

As we are again using flash messages, we will need to add some code to display them in the view:

In `login.ejs`, add the same code that we added in `signup.ejs` to display the flash messages:

```javascript
  <% if (message.length > 0) { %>
    <div class="alert alert-danger"><%= message %></div>
  <% } %>
```

#### Login GET Route handler

Now, let's add the code to render the login form in the `getLogin` action in the controller (`users.js`):

```javascript
  function getLogin(request, response, next) {
    response.render('login.ejs', { message: request.flash('loginMessage') });
  }
```

You'll notice that the flash message has a different name (`loginMessage`) than in the signup route handler.

<!--3:34 return WDI3-->

#### Login POST Route handler

We also need to have a route handler that deals with the login form after we have submitted it. So in `users.js` lets also add:

```javascript
  function postLogin(request, response, next) {
    var loginStrategy = passport.authenticate('local-login', {
      successRedirect : '/',
      failureRedirect : '/login',
      failureFlash : true
    });

    return loginStrategy(request, response, next);
  }
```

You should be able to login now!

<!--WDI5 3:42  -->
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

<!--Kinda gave up on all the errors, and plowed through around 3:40 -->

#### Accessing the User object globally

By default, passport will make the user available on the object `request`. In most cases, we want to be able to use the user object everywhere. For that, we're going to add some middleware in `app.js` underneath our passport require statement:

```javascript
  require('./config/passport')(passport);

  app.use(function (req, res, next) {
    res.locals.currentUser = req.user;
    next();
  });
```

Now in the header partial, we can add:

```javascript
<ul class="nav navbar-nav">
    <li><a href="/secret">Secret</a></li>
  <% if (currentUser) {%>
    <li><a href="/logout">Logout <%= currentUser.local.email %></a></li>
  <% } else { %>
    <li><a href="/login">Login</a></li>
    <li><a href="/signup">Signup</a></li> 
  <% } %>
</ul>
```

<!--4:00 WDI5 -->
<!--2:30 5 minutes -->

## Signout - Codealong

#### Logout

The last action to implement for our authentication system is to set the logout route and functionality.

In `controllers/users.js`:
```js
function getLogout(request, response, next) {
  request.logout();
  response.redirect('/');
}
```

<!--2:35 5 minutes -->

## Test it out - Independent Practice

You should now be able to login and logout! Test this out.

<!--WDI5 4:03, and just talked out last piece plus basic passport advice for project 2 -->
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

Now test it out by clicking on the secret page link. If you are not logged in, it will simply redirect you to the home page.

### Final Challenge

Go into the `controllers.js` file and add a super-secret JSON response to the `secret` function.  Now try logging in and getting to the super-secret JSON message.

<!-- 2:50 5 minutes -->

## Conclusion

Passport is a really useful tool because it allows developers to abstract the logic of authentication and customize it, if needed. It comes with a lot of extensions that we will cover later.

- What does it mean for a user to be "logged in"?
- Briefly describe the authentication process using passport in Express.

<!--WDI3 Finished 4:03 -->

## Resources

- [Passport Docs](http://passportjs.org/docs)

<!-- Didn't finish till 3:51 -->
