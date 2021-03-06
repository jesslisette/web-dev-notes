# Saving Register Users to Database
1. We finished validating the registration data (done)
2. Register the user and save them to our database (working on this now)
3. Log user in

## Register user and save them to our Database
### Create the route that points to the controller
`index.js`

```
// 1. Validate the registration data
// 2. Register the user
// 3. We need to log them in
router.post('/register',
  userController.validateRegister,
  userController.register
);

module.exports = router;
```

## Build our register controller
`userController.js`

```
exports.register = async (req, res, next) => {
  
};
```

### Fix stuff
We made a couple mistakes

#### Wrong import
This is wrong

`const passportLocalMongoose = require('password-local-mongoose');`

It is pointing to wrong package (_package.json_) will tell you what our dependencies are and that will let us know the correct spelling of our dependent packages

`package.json`

```
// more code
    "passport-local": "1.0.0",
    "passport-local-mongoose": "4.0.0",
    "pug": "2.0.0-beta6",
    "slugs": "0.1.3",
    "uuid": "3.0.1",
    "validator": "7.0.0"
  },
// more code
```

So we update our require to the correctly spelled package

`User.js`

```
const mongoose = require('mongoose');
const validator = require('validator');
const mongodbErrorHandler = require('mongoose-mongodb-errors');
const passportLocalMongoose = require('passport-local-mongoose'); // fix this line
const md5 = require('md5');

const Schema = mongoose.Schema;
// more code
```

### Import our Model
`start.js`

```
// more code

// import all of our models
require('./models/Store');
require('./models/User');

// more code
```

## Tell our controller about our User model
`userController.js`

```
const mongoose = require('mongoose');
const User = mongoose.model('User');

// more code
```

* How can we do that?
    - Because we already imported `User` in `start.js`

### promisify
We will need this library

`userController.js`

```
const mongoose = require('mongoose');
const promisify = require('es6-promisify'); // add this line
const User = mongoose.model('User');
```

### What happens when we get to `register()` in `userController`?
That means we got passed our `Client side` validation and we have sanitized and checked:

* Our `req.body` is filled with:
    - name
    - email
    - password
    - confirmed password

### Let's start making our user
![access field names](https://i.imgur.com/yHLYeq8.png)

* Those are the three fields we need to access
    - req.body.name
    - req.body.email
    - req.body.password

#### .register(model, password, callback())
* This method will take our password
* Hash it
* Save it to our database
* Why are we using a callback()? Aren't we using asyn-await?
    - Yes but the `.register()` library doesn't return a Promise
    - This is common when using external Libraries that are not using Promises just yet
        + When this happens you have two choice
            * Use their callback
            * Use the `promisify` library
                - This will take the older callback based function and convert it into a Promised based function (so we can keep using async-await!)

##### Where did .register() come from?
`User.js`

```
// more code
userSchema.plugin(passportLocalMongoose, { usernameField: 'email' });
// more code
```

###### `passportLocalMongoose` plugin
* This exposes us to the `.register()` method
* This will take care of all the lower level registration for us

## promisify(method you want to promisify, which object to bind to)
`const registerPromisify = promisify(User.register, User);`

### Tip
If you are ever using this `promisify` Library, if the method you are trying to promisify lives on an object, you also MUST pass the entire object so it knows where to bind itself to

`userController.js`

```
// more code

exports.register = async (req, res, next) => {
  const user = new User({ email: req.body.email, name: req.body.name });
  const register = promisify(User.register, User);
  await register(user, req.body.password);
  next(); // pass to authController.login
};
```

* It will take the user and email and store them in the Database
* It will not store the **password** in the database
    - It will store a **hash** of the password in the database

## Law of User Authentication
Never store the user's password in the Database 

Your password: **"hello"**
md5() -----> **"8b1a9953c4611296a827abf8c47804d7"**

A user registers and the hashed password above is stored in the Database
When that user logs in at some point in the future they will enter "hello" as their password
That is run through `md5()` and if it matches the hashed password in the database, they can log in, if no match, they are denied access

The user doesn't know the sale - they don't know how it is being hashed

`userController.js`

```
// more code
exports.register = async (req, res, next) => {
  const user = new User({ email: req.body.email, name: req.body.name });
  const registerWithPromise = promisify(User.register, User);
  await registerWithPromise(user, req.body.password);
  res.send('it works');
  next(); // pass to authController.login
};
```

### Test in browser
1. Visit `/register`
2. Fill out form completely
3. Submit
4. You should see `it works`

### Check to see if a new user exists in our Database
Open `MongoDB` Compass

![you see the new user in Database](https://i.imgur.com/ngDHTmt.png)

* That password is salted and hashed
