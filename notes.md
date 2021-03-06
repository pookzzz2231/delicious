* [setting up mongo](#setting-up-mongo)  
* [variable and start](#variable-and-start)  
* [routing and middleware](#routing-and-middleware)  
* [Templating](#templating)  
  * [helper](#helper)  
  * [mixins](#mixins)  
* [MVC pattern](#mvc-pattern)  
* [Store](#store)  
  * [create store model with mongoose](#create-store-model-with-mongoose)  
  * [get add store page](#get-add-store-page)  
  * [post add store with async await](#post-add-store-with-async-await)  
  * [flash message](#flash-message)  
  * [query db for stores and get stores page](#query-db-for-stores-and-get-stores-page)  
  * [find and edit](#find-and-edit)  
  * [add timestamp and location to Model](#add-timestamp-and-location-to-model)  
  * [geolocation with google map](#geolocation-with-google-map)  
  * [upload and resizing image](#upload-and-resizing-image)  
  * [store show page](#store-show-page)  
  * [unique slug with mongoose hook](#unique-slug-with-mongoose-hook)  
* [tags](#tags)  
  * [data aggregation pipeline and tags page](#data-aggregation-pipeline-and-tags-page)  
  * [Multiple Query Promises with AsyncAwait](#multiple-query-promises-with-async-await)  
* [users](#users)  
  * [create users db and get form](#create-users-db-and-get-form)  
  * [validate user registration](#validate-user-registration)  
  * [post and save user to db](#post-and-save-user-to-db)  
  * [auth-and-login](#auth-and-login)  
  * [logout](#logout)  
  * [gravatar](#gravatar)  
  * [protecting route](#protecting-route)  


## Setting up mongo  
* download mongo community server from mongodb.com  
* install with mongo compass(mongo GUI)  
* open command line from installed folder then cd into bin and run `mongod` command   
> for windows create `C:\data\db` folder for mongo to work  

## Variable and start  
* all backend secret variable store in `variables.env`  
* connect mongo db with mongoose and config mongoose in `start.js`  
* `start.js` use dependency `dotenv` which will point and load secrect variable from `variables.env`  
* start script included in `start.js`  
* run `npm start`; watch with `nodemon`(watch and restart when file changes)  
* data base will auto detect and create with `start.js` and `variables.env` if not exists  

## Routing and middleware
* `app.js` contains all of middleware(scripts that load between request response cycle)  
* after config all middleware, import route from route modules in `routes` folder then use express to specify route  
* handle errors after define routes in `app.js`  

```js
//app.js
const index = require('./routes/index');
const app = express();

//...middleware
app.use('/', index);

//...handle errors after lookup for routes
```

* define request response in node server at `routes/route_path_name.js` then export as module  
```js
//index.js
const express = require('express');
const router = express.Router();

router.get('/', (req, res) => {
  res.send('Hello world!');
});

module.exports = router;
```

> req and res properties are provided by `bodyParser` middleware which parsing JSON, plain text, or just returning a raw Buffer object

* some useful methods for req, res  
```js
// router

// send need to execute only once in router
// Header can only send once per response
res.send("data");
res.json({name: 'abc', age: 20}); //send response as json

//get params with request
//ex url: www.something.com/?name=abc&age=21
req.query; // {name: "abc", age: "21"}
req.query.name; //abc 

//params routes
router.get('/users/:name', (req, res) => {
  req.params.name; //access param :name
});
```

> check express docs for more API

## Templating
* render template for request(will use pug template)  
* all of template will be in `/views`  
* `pug` middleware is configed and pointed to views path in `app.js`  
```js
//router
router.get('/', (req, res) => {
  res.render('hello', {
    name: 'abc',
    age: req.query.name
  });
});
```

* `pug` reuseable parts of template with `block`  
```pug
<!-- layout -->
.content
  block content
```
```pug
<!-- content -->
extends layout

block content
  p Hello
```

* if specified inside `block` will use default for `block`, `extend` and define `block` in content to overwrite the default  
```pug
<!-- layout -->
block header
  header.top
    nav.nav
      p some header
```

### helper
* defined reusable variables, libraries and functions in `helpers.js` to export and easily use in our template or file  

* import `helpers` in `app.js` and define it as `locals` for express middleware to be able to use across express req/res cycle  
```js
// app.js
const h = require('./helpers');

// define locals to use across res before routes
app.use((req, res, next) => {
  res.locals.h = helpers;
  //...
  next();
});
```

> `res.locals` is an object that contains response local variables scoped to the request, and available only to the view rendered during that request / response cycle (if any)

* `app.locals` has properties that are local variables within the application.
```js
app.locals.title = 'My App';
app.locals.strftime = require('strftime');
app.locals.email = 'me@myapp.com';
```

### mixins
* can use as reusable template or script for pug  
* define pug mixins in `views/mixins/_fileName.pug`  
```pug
mixin something(store={})
  p something
```

```pug
extends layout

<!-- include the mixins -->
include mixins/_fileName
```

## MVC pattern  
* Model: query data, create tables, define schema, slug  
* View: template  
* Controller: control route, paths and maybe some custom middleware    

```js
//controller
//controllers/storeController.js
exports.homePage = (req, res, next) => {
  res.render('index');
}

//routes
//...other code
const storeController = require('../controllers/storeController');

// router.get('/', controllerRoutes, [middleware]);
router.get('/', storeController.homePage);
```

## Store
### create store model with mongoose
* create model file `models/store.js`  
* define and config mongo schema with mongoose(package to interface with mongo)  
```js
// models/Store.js

//require mongoose dependency
const mongoose = require('mongoose');
//config mongoose how to get data back from db
//use js promise, asyn await or lib such as bluebird
mongoose.Promise = global.Promise;
//use slug lib for url friendly
const slug = require('slugs');

//define schema; all its properties and data type
const storeSchema = new mongoose.Schema({
  name: {
    type: String,
    trim: true,
    required: 'Please enter a store name', //overwrite mongoose default errors with message
  },
  slug: String,
  description: {
    type: String,
    trim: true
  },
  tags: [String]
});

// mongoose middleware
// mongoose hook to hook in something before mongo save to db
storeSchema.pre('save', function(next) {
  // use es5 not arrow function, need the storeSchema context
  // if use arrow function this will be window 

  // run hook only when name is modified
  if (!this.isModified('name')) {
    //skip and return
    return next(); 
  }

  this.slug = slug(this.name);
  // call next to finish the hook
  next();
});

//export schema as module; export as a function
//mongoose.model('ModelName', modelSchema);
module.exports = mongoose.model('Store', storeSchema);
```

* in `start.js` instantiated connection only once and then reused it; Singleton  
```js
//start.js
//...other code
// Singleton mongo model connection
require('./models/Store');
```
How errorHandler(`errorHandlers.js`) is handled from request:
> when set model schema `required` field  to `true` or overwrite with custom string, if request comes in without those required field, in `app.js` the error handler `catchErrors` which wrapped in routes will call `next`.  <br><br> 
Then `errorHandlers.flashValidationErrors` function will check for request errors. If there is an error, will iterate and display errors as flash messages and redirect back to previous request. Alternatively, if there is no error, will call next and pass `error` to other errorHandler.

### get add store page
* create view, route and controller  

* controller implementation
```js
// storeController.js
// exports as exports properties
exports.addStore = (req, res) => {
  res.render('editStore', { title: 'Add Store' })
}
```

* routes implementation  
```js
// routes/index.js
// ...other code

router.get('/add', storeController.addStore);

// ...other code
```

* view implementation for form mixin
```pug
<!-- views/mixins/_storeForm.pug -->
mixin storeForm(store = {})
  form(action="/add" method="POST" class="card")
    label(for="name") Name
    input(type="text" name="name")
    label(for="description") Description
    textarea(name="description")
    - const tags = ["restaurant", "Wifi", "cheap", "cash only", "credit card"];
    ul.tags
      each tag in tags
        .tag.tag__choice
          input(type="checkbox" id=tag, value=tag name="tags")
          label(for="choice")= tag
    input(type="submit" value="Save" class="button")      
```

### post add store with async await
* implement route to handle post request  
```js
// routes/index.js
// ...other code

router.post('/add', storeController.createStore);
```

* implement controller, use mongoose to import schema and save to db when post asyncronously

> Mongoose is already define in start.js to use promise for db interaction `mongoose.Promise = global.Promise`

Mongoose use promise to interact with db, save data as model obj using schema  
```js
// controllers/storeController.js
// ...other code
const mongoose = require('mongoose');
// with singleton, mongoose need to import model(schema) once, which already did in start.js
// after singleton mongoose model can be reference anywhere right away
const Store = mongoose.model('Store'); // exported from models/Store.js as 'Store'

// controller 
exports.createStore = (req, res) => {
  // use Store model schema to create new Store obj
  const store = new Store(req.body);
  // save store asyncronously with mongoose
  store.save() // store.save() will return the Promise resolve if successful
       .then(stores => {
         // return promise
       })
       .then(stores => {
         res.render('templateView', { stores: stores }); // render with resolve from promise
       })
       .catch(err => {
         throw Error(err); // catch error if has one
       })
};
```

Use async await instead of Promise  
```js
// controllers/storeController.js
// ...other code
const mongoose = require('mongoose');

// exported from models/Store.js as 'Store'
// mongoose connect to Store model and schema 
const Store = mongoose.model('Store'); 

// controller with async await
// mark function that is async
exports.createStore = async (req, res) => {
  try {
    const store = new Store(req.body);
    await store.save(); // mark the async return with await
  } catch(err) {
    // catch error
  }
};
```
Instead of catching errors in all controller wrap the function with higher order function(function that take another func as arg)
```js
// handlers/errorHandlers.js

// We will wrap the routes with catchError, which will call next if errors occur in routes
// after next is call in express router middleware in app.js, it will go and look below app.use("path", "route") for a suitable errorhandler code

// export as properties
exports.catchError = (fn) => {
  return function(req, res, next) {
    return fn(req, res, next).catch(next);
  }
}
```

```js
// routes/index.js
// ...other code
const { catchErrors } = require('../handlers/errorHandlers'); // ES6 decomposition pull prop from obj and assign to const

// use js composition to wrap function
// will return next and find error in app.js if errors occured
router.post('/add', catchErrors(storeController.createStore));
```

```js
// controllers/storeController.js
exports.createStore = async (req, res) => {
  // let error happen which we wrapped with handle errors
  // and push along in middleware in app.js to find suitable errors
  const store = new Store(req.body);
  await store.save();
  res.redirect("/");
};
```

### flash message
* flash is implemented with `connect-flash` middleware  

* use middleware to pass flash request to response's locals before hit the route and response back  

* config middleware to config and store flash message  
```js
// app.js
// ... other code
app.use((req, res, next) => {
  // ... others
  // set flashes as locals variables before hit the routes
  // flashes variable will available to use in all response templates
  res.locals.flashes = req.flash(); 
  next();
})
```

* implement flash in view template
```pug
<!-- layout -->
block messages
  if flashes
    .flash-messages
      
```

* implement flash in controller  
```js
// controllers/storeController.js
// ...

exports.createStore = async (req, res) => {
  // save and return mongo db object as promise resolve and store in store const
  const store = await (new Store(req.body)).save(); 

  // flash middleware
  // req.flash('flashType', 'message')
  req.flash('success', `Successfully created ${store.name}. Leave a review?`);

  // when redirects new request is issued with flashes in req
  // flash in req will then store in res.locals.flashes
  // then it will be available in template as res variables
  res.redirect(`/store/${store.slug}`); // store return db promise resolve obj
}
```

> Flash only works when session is implemented, and is a part of session  

### query db for store and get stores page
* create controller, query the db with mongoose asyncronously  

> don't forget to wrap error handler in `routes` for any controller that use async callback 

```js
// storeController.js
// other code...
const Store = mongoose.model('Store'); // conect to model

exports.getStores = async (req, res) => {
  //1. Query db for all stores
  const stores = await Store.find(); // find all data in Store and return promise

  res.render('stores', { title: 'Stores', stores }); //ES6 stores: stores
}
```

* create reuseable mixin UI then loop and display stores in `stores` view
```pug
<!-- views/mixins/_store.pug -->
mixin store(store={})
  .store
    .store__hero
      .store__actions
        .store__action.store__action--edit
          a(href=`/stores/${store._id}/edit`)
            != h.icon('pencil')
      img(src=`/uploads/${store.photo || 'store.png'}`)
      h2.title
        a(href=`/store/${store.slug}`)= store.name
    .store__details
      p= store.description.split(' ').slice(0, 25).join(' ')
```

```pug
<!-- stores -->
extends layout

include mixins/_store

block content
  .inner
    h2= title
    .stores
      each store in stores
        +store(store)
```

> tips: `!=` in pug will tell pug to render and escape html

### find and edit 
* include ui like for edit  
```pug
.store__action.store__action--edit
  a(href=`/stores/${store._id}/edit`)
    != h.icon('pencil')
```

* Implement the route for GET 
```js
// router.js
// other code...

router.get("/stores/:id/edit", catchErrors(storeController.editStore));
```

* Implement controller  
```js
// storeController.js
// other code...
const mongoose = require('mongoose');
const Store = mongoose.model('Store');

exports.editStore = async (req, res) => {
  // find the store with ID from params url
  const store = await Store.findOne({ _id: req.params.id });

  // render edit form, pass query from DB to view
  res.render('editStore', { title: `Edit ${store.name}`, store }); // ES6 store: store
};
```

* modify mixin `_storeForm` to show the value and config route
```pug
<!-- if no store pass in value is default to {} -->
mixin storeForm(store={})
  form(action=`/add/${store.id || ''}` method="POST" class="card")
    label(for="name") Name
    input(type="text" name="name" value=store.name)
    label(for="description") Description
    textarea(name="description")= store.description
    - const tags = ["restaurant", "Wifi", "cheap", "cash only", "credit card"];
    - const storeTags = store.tags || [];
    ul.tags
      each tag in tags
        - const checked = storeTags.includes(tag);
        .tag.tag__choice
          input(type="checkbox" id=tag, value=tag name="tags" checked=checked)
          label(for="choice")= tag
    input(type="submit" value="Save" class="button")  
```

* Implement the route for POST
```js
// route
// ...
router.post("/add/:id/", catchErrors(storecontroller.updateStore));
```

* Implement the controller to handle POST
```js
// storeController
// ...
exports.updateStore = async (req, res) => {
  // find store from db with params and update in one method
  // Model.findOneAndUpdate({ query }, data, [options]);
  const store = await Store.findOneAndUpdate({ _id: req.params.id }, req.body, {
    new: true, // return updated data not old one from mongo
    runValidators: true // validates the required field in schema not only on create but on update as well
  }).exec(); // method to ensure execution for mongo

  // flash
  req.flash("success", `Successfully updated <a href=/stores/${store.slug}><strong>${store.name}</strong></a>`);

  // redirect
  res.redirect(`/add/${store._id}/edit`);
};
```

### add timestamp and location to Model
* implement more field in Store model  
```js
// models/Store.js
// ...other code
const storeSchema = new mongoose.Schema({
  name: {
    type: String,
    trim: true,
    required: 'Please enter a store name', //overwrite mongoose default errors with message
  },
  slug: String,
  description: {
    type: String,
    trim: true
  },
  tags: [String],
  // add new fields below
  created: {
    type: Date,
    default: Date.now()
  },
  // nested attrbites store in db
  location: {
    type: {
      type: String,
      default: 'Point' // will located point map in mongo compass GUI
    },
    coordinates: [{
      type: Number,
      required: 'Must have coordinates!'
    }],
    address: {
      type: String,
      required: 'Must have address!'
    }
  }
});
``` 

> in middleware config `app.js`:  <br><br>
`app.use(bodyParser.urlencoded({ extended: true }))` will allow us to use rich objects and arrays to retrive data from db or save db into db as nested attributes <br><br>
For example, the string `person[name]=bobby` and `person[age]=3` will be converted to:

```js 
person: {
    name: 'bobby',
    age: 3
}
```

* implement new model fields in view
```pug
<!-- _storeForm.pug -->
mixin storeForm(store={})
  form(action=`/add/${store.id || ''}` method="POST" class="card")
    //- other field
    //- add address
    label(for="address") Address
    input(type="text" id="address" name="location[address]" value=(store.location && store.location.address))
    label(for="lng") Address Lng
    input(type="text" id="lng" name="location[coordinates][0]" value=(store.location && store.location.coordinates[0]) required)
    label(for="lat") Address Lat
    input(type="text" id="lat" name="location[coordinates][1]" value=(store.location && store.location.coordinates[1]) required)
    //- other field
```

* type of `'Point'` will lose when user update query, in controller force to update location type to `'Point'`
```js
// StoreController.js
exports.updateStore = async (req, res) => {
  // force update location to Point
  req.body.location.type = 'Point';

  // other code...
}
```

### geolocation with google map
> will use `bling.js` library which convert Javascipt DOM query to `$` and `$$`, additionally convert `addEventListener` to `on`

* include `layout` view with google API  

* implement client side using google place api to autocomplete when type in address  

* wrap google API with custom module to insert lat ang long coordinates in input with enter address  
```js
// public/javascripts/delicious-app.js
import { $, $$ } from './modules/bling';
import autocomplete from './modules/autocomplete';

// call imported function from custom module autocomplete and pass in input DOM
autocomplete($('#address'), $('#lat'), $('#lng'));
```
```js
// public/javascripts/modules/autocomplete.js
function autocomplete(address, lat, lng) {
  if (!address) { return; } // no input

  const addresses = new google.maps.places.Autocomplete(address); // google api

  // google API; addListener, place_changed, getPlace()
  addresses.addListener('place_changed', () => {
    // get google place obj from google API
    const place = addresses.getPlace();

    // insert lat and lng input dom with google place API
    lat.value = place.geometry.location.lat();
    lng.value = place.geometry.location.lng();
  });

  // prevent enter to submit form
  address.on('keydown', (e) => {
    if (e.keyCode === 13) { e.preventDefault(); };
  });
}

// export as module
export default autocomplete;
```

### upload and resizing image
* use middleware `multer` to upload the file and `jimp` to resize the file  

* store image NAME.EXT in MongoDB, store ACTUAL image file in server(or cloud)

* file needs to be miltipart when uploaded, modify the form  
```pug
//- _storeForm.pug
mixin storeForm(store={})
  form(action=`/add/${store._id || ''}` method="POST" class="card" enctype="multipart/form-data")
  //- ...
``` 

* uploading file process  
  * use `multer` middleware to handle multipart(file) upload request in controller  
  * `multer` will read into temp memory as buffer  
  * then async with `jimp`(implement with promise) for resizing  
    * detect file type with mimetype and rename file, then point the file name to `req.body.fieldName`, which will be save in MongoDB when controller processed  
    * then assign `photo` variable to await for jimp's resolve from reading the buffer  
    * after resolve from await buffer reading, next await for `photo` to resize  
    * await for `photo` to write to the image save path   
```js
//storeController.js
//...
const multer = require('multer'); // save file to server
const jimp = require('jimp'); // resize
const uuid = require('uuid'); // unique id for anything
const imgPath = "./public/uploads";

// config multer middleware
const multerOptions = {
  storage: multer.memoryStorage(), // store into memory, not in server disk
  fileFilter(req, file, next) {
    const isPhoto = file.mimetype.startsWith('image/'); // server side validation image with mimetype
    if(isPhoto) {
      // next(err); pass on error
      // next(null, true); no errors, pass on true 
      next(null, true);
    } else {
      next({ message: 'File is not allowed!' }, false); // error obj, pass on false
    }
  }
}

// set middleware to accpet a single file with the A 'name'(name on req.body from form 'name'), can check for multiple fields as well(array).
// The single file will be stored in req.file
exports.upload = multer(multerOptions).single('photo');

// resize asynchronously 
exports.resize = async (req, res, next) => {
  // multer read and process request as file obj in req.file
  // use req.file to check if there is any file being attached
  if (!req.file) {
    // no any file attached, skip to the next middleware
    return next();
  }

  // check the mime type, rename file, and store that name to mongo DB
  const ext = req.file.mimetype.split('/')[1];
  // assign req.body.photo to filename.ext string, will be posted with other requested fields in controller
  const fileName = req.body.photo = `${uuid.v4()}.${ext}`; 

  // next we concern with saving actual file to db
  // read from temp memory and resize with jimp
  const photo = await jimp.read(req.file.buffer);
  // resize the photo obj, width, height
  await photo.resize(800, jimp.AUTO);
  // write file to server path --> file.write(filePath)
  await photo.write(`imgPath/${fileName}`);
}

// controllers ...
```  

* modify form to upload photo  
```pug
//- _storeForm.pug
mixin storeForm(store={})
  form(action=`/add/${store.id || ''}` method="POST" class="card" enctype="multipart/form-data")
    //- other field
    label(for="photo") Photo
    //- client side validation
    input(type="file" name="photo" id="photo" accept="image/gif, image/png, image/jpeg")
    - if store.photo
      img(src=`uploads/${store.photo}` alt=store.name width=200)
```

* add Model schema to store fileName in DB
```js
// models/Store.js
// ...
const storeSchema = new mongoose.Schema({
  {
    //...
  },
  photo: String
}); 
```

* implement middleware in routes
```js
// routes/index.js
// ...
// express router API --> router.post('path', [middleware..], req, res, [next] => {...})

// implement upload and resize middleware for create
router.post('/add', 
  storeController.upload, 
  catchErrors(storeController.resize), // async
  catchErrors(storeController.createStore) // async
);

// implement upload and resize middleware for update store
router.post('/add/:id', 
  storeController.upload, 
  catchErrors(storeController.resize), // async
  catchErrors(storeController.updateStore) // async
);
```

### store show page
* add route to get store show page  
```js
// routes/index.js
// ...
router.get('store/:slug', catchErrors(storeController.getStoreBySlug));
```

* implement controller for store show  
```js
// storeController.js
// ...

exports.getStoreBySlug = async (req, res) => {
  // query db for a store
  const store = await Store.findOne({ slug: req.params.slug });

  // if slug url does not exists, return next which will reach notFound handler; 404
  if (!store) { return next(); };

  // render template with store and pass in store query from db
  res.render('store', { store, title: store.name });
}
```

* create view to display data from controller, and include a static google map from `helpers.js`    
```pug
//- views/store.pug
extends layout

block content
  .single
    .single__hero
      img.single__image(src=`/uploads/${store.photo || 'store.png'}`)
      h2.title.title--single
        a(href=`/stores/${store.slug}`)= store.name
  
  .single__details.inner
    img.single__map(src=h.staticMap(store.location.coordinates))
    p.single__location= store.location.address
    p= store.description

    - if store.tags
      ul.tags
        each tag in store.tags
          li.tag
            a.tag__link(href=`/tags/${tag}`)
              span.tag__text ##{tag}
```

> pug string interpolate with `#{}`, can combine other string with the interpolation; `tags #{tag}`

### unique slug with mongoose hook
* in Model use `mongoose` pre-save hook to schema to check and create new slug if existed  
```js
// models/Store.js
// ...

// need `this` context; use ES5
storeSchema.pre('save', async function(next) {
  if (!this.isModified('name') {
    return next();
  });

  this.slug = slug(this.name);

  // create regex obj to check for matching in DB
  const slugRegEx = new RegExp(`^(${this.slug})((-[0-9]*$)?)$`, 'i');

  // find slug that match from database with regex, refer `this.constructor` as model, since it is yet not created
  const matched = await this.constructor.find({ slug: slugRegEx });

  // match will return array of query if found
  if (matched.length) {
    // increase the slug number by one 
    this.slug = `${this.slug}-${matched.length + 1}`
  }

  // go to next middleware
  next();
});
```
## tags
### data aggregation pipeline and tags page
> Unlike `relational database` `mongoDB` does not have table and `foreign key` to establish tables relationship for data query.  <br><br>
Use advance query: `data aggregation pipeline` which operations group values from multiple documents together using `mongodb aggregate methods`, and can perform a variety of operations on the grouped data to return a single result 

* create routes for tag page  
```js
// routes/index.js
// ...
router.get('/tags', catchErrors(storeController.getStoresByTag));
router.get('/tags/:tag', catchErrors(storeController.getStoresByTag));
```

* implement controller  
```js
// storeController.js
// ...
exports.getStoreByTag = async (req, res) => {
  // create custom Model query method `getTagsList` and implement in model
  const tags = await Store.getTagsList();
}
```

* implement `getTagList` and add custom query method in Model  
```js
// models/Store.js
// ...

// add custom method to schema, use schema.statics
// use ES5 to get schema context; ES6 'this' context is parent scope
storeSchema.statics.getTagsList = function() {
  //  aggregate method, pipeline multiple aggregate operators in an array
  // pass each pipeline as object in array
  return this.aggregate([
    // specify the field path operation with '$field'
    // $unwind breaks array data into single attributes 
    { $unwind: '$tags' }, 
    // group in "id: tags", and add count field which sum(add) each data += 1 --> { "id: tags, "count": 3" } 
    { $group: { _id: '$tags', count: { $sum: 1 } }},
    // sort by count descending: -1, ascending: 1
    { $sort: { count: -1 } }
  ]);
}
```

* implement data to render in controller  
```js
// storeController.js
// ...
exports.getStoreByTag = async (req, res) => {
  // create custom Model query method `getTagsList` and implement in model
  const tags = await Store.getTagsList();

  // pass current tag params for active UI in view
  const active = req.params.tag

  res.render('tag', { tags, active, title: 'Tags' });
}
```

* implement view  
```pug
//- views/tags.pug
extends layout

block content
  .inner
    h2 #{active || 'Tags'}
    ul.tags
      each tag in tags
        li.tag
          a.tag__link(href=`/tags/${tag._id}` class=(tag._id === active ? 'tag__link--active' : ''))
            span.tag__text= tag._id
            span.tag__count= tag.count
```

### Multiple Query Promises with async await
* Query stores for a specific tag  

* need to get promise resolves for tags and stores in one controller  

* use `async` `await` `Promise.all` which will return array of resolves  
```js
// storeController.js
// ...

exports getStoreByTag = async(req, res) => {
  const tag = req.params.tag;

  // define async promises; each method is already implement with async Promise
  // find matched tag array or return all tags in db; use when has no tag params and would like to return all tags
  const storePromise = Store.find({ tags: (tag || { $exists: true }) });
  const tagsPromise = Store.getTagsList();

  // await for all Promise to return resolves at the same time
  const [stores, tags] = await Promise.all([storePromise, tagsPromise]);

  // pass all data to view
  res.render('tag', { stores, tags, tag, title: 'Tags' });
}
```

* implement in view and use mixins `_store` as partials to render `store` data  
```pug
extends layout

include mixins/_store

block content
  .inner
    h2 #{active || 'Tags'}
    ul.tags
      each tag in tags
        li.tag
          a.tag__link(href=`/tags/${tag._id}` class=(tag._id === active ? 'tag__link--active' : ''))
            span.tag__text= tag._id
            span.tag__count= tag.count

    .stores
      each store in stores
        +store(store)
```

## users
### create users db and get form
* create route for login and register    
```js
// routes/index.js
// ...
const userController = require('../controllers/userController');

router.get('/login', userController.login);
router.get('/register', userController.new);

module.exports = router;
```

* create user controller  
```js
// controllers/userController.js
const mongoose = require('mongoose');

exports.login = (req, res) => {
  res.render('login', { title: "login" });
};

exports.new = (req, res) => {
  res.render('register', { title: "register" });
};
```

* create partial mixins for a form, then implement in user login and register view template  
```pug
//- views/mixins/_loginForm.pug
mixin loginForm()
  form.form(action="/login" method="POST")
    label(for="email") Email
    input(type="email" name="email")
    label(for="password") Password
    input(type="password" name="password")
    input.button(type="submit" value="Log In")    
```
```pug
//- login.pug
extends layout

include mixins/_loginForm

block content
  .inner
    h2= title
    +loginForm()
```
```pug
//- views/mixins/_registerForm.pug
mixin loginForm()
  form.form(action="/register" method="POST")
    label(for="name") Name
    input(type="text" name="name" required)
    label(for="email") Email
    input(type="email" name="email" required)
    label(for="password") Password
    input(type="password" name="password")
    label(for="password-confirm") Confirm password
    input(type="password" name="password-confirm")
    input.button(type="submit" value="Register")    
```
```pug
//- register.pug
extends layout

include mixins/_registerForm

block content
  .inner
    h2= title
    +registerForm()
```

* create use model, then import model once(singleton) in `start.js`    
```js
// models/User.js
const mongoose = require('mongoose');
const Schema = mongoose.Schema;
mongoose.Promise = global.Promise;

const md5 = require('md5');
// validate middleware
const validator = require('validator');
// by default unique and other field in mongo schema will give ugly errors, use this middleware to display nicer errors
const mongodbErrorHandler = require('mongoose-mongodb-errors');
// use as a plugin for mongoose for building username and password for passport
// define User as prefered. Passport-Local Mongoose will add a username, hash and salt field to store the username, the hashed password and the salt value when posted with async User.register(obj, password) method
const passportLocalMongoose = require('passport-local-mongoose');

const User = new Schema({
  email: {
    type: String,
    // will give ugly errors, use mongodbErrorHandler middleware for better errors
    unique: true,
    lowercase: true,
    trim: true,
    // server validation using validator middleware
    validate: [validator.isEmail, "Invalid Email!"],
    // set require with error message
    required: "Email is required!" 
  },
  name: {
    type: String,
    trim: true,
    required: "Name is required!"
  }
});

// add mongoose plugin with passport authenticate middleware API for User schema model
// set option usernameField to User's email
User.plugin(passportLocalMongoose, { usernameField: 'email' });

// middleware to make error nicer
User.plugin(mongodbErrorHandler);

module.exports = mongoose.model('User', User);
```

### validate user registration
* create a middleware which will run before routes, validates the registration on server side, and handle register form submit's request  
```js
// lib/validateRegister.js
const validateRegister = (req, res, next) => {
  // method from express-validator middleware in app.js, remove html and js script before post
  req.sanitizeBody('name');
  // check body field
  req.checkBody('name', 'must have name').notEmpty();
  req.checkBody('email', 'email not valid').isEmail();

  // normalize email instead of using default validation
  // for example, if not normalized, 'abc.def@gmail.com' and 'abcdef@gmail.com' would validate as the same email
  req.sanitizeBody('email').normalizeEmail({
    remove_dots: false,
    remove_extension: false,
    gmail_remove_subaddress: false
  });

  req.checkBody('password', 'Password cannot be blank!').notEmpty();
  req.checkBody('password-confirm', 'Password confirmed cannot be blank!').notEmpty();  

  // check password eq password confirm
  req.checkBody('password-confirm', 'Passwords are not matched!').equals(req.body.password);

  // express-validator middleware will get all validators method above and assign to errors
  const errors = req.validationErrors();

  // check for errors and render flash
  //
  if (errors) {
    // handle error for flash manually, not included in errorHandler middleware
    req.flash('error', errors.map(err => {err.msg}));

    // render the form with flashes and send back the same data to page body if error 
    res.render('register', { title: 'Register', body: req.body, flashes: req.flash() });
    return; // return out of middleware if error;
  };

  next(); // no errors process to next middleware
}

exports default validateRegister;
```

### post and save user to db
* create post route for user register and then logged user in 
```js
// route
router.post('/register',
  userController.validateRegister,
  userController.create,
)
```

* implement userController  
```js
// userController.js
const mongoose = require('mongoose');
const User = mongoose.model('User'); // already imported in start.js

// use async await for non supported libaray
const promisify = require('es6-promisify');

exports.create = async (req, res, next) => {
  const user = new User({ email: req.body.email, name: req.body.name });
  
  // convert promise to async for lib that is not yet supported; promisify(method, bindContext)
  // bindContext is not needed if method is the function in the same scope
  // User.register(obj, password) is a method provided with passportLocalMongoose which binds to Model context
  const register = promisify(User.register, User); // return function
  // now register is an async method which can be call with await
  await register(user, req.body.password); // register method will store user attributes and encrypted password

  next(); // pass to login authentication; auth controller
};
```

### auth and login
* create controller middleware for auth  
```js
// authController.js
const passport = require('passport'); // lib for login

// use passport strategy middleware; define how to log user in
// can be facebook, google, github or local
// if authentication was successful, `req.user` contains the authenticated user, which then will be stored in res.locals.user in app.js
exports.login = passport.authenticate('local', {
  // option for config
  failureRedirect: '/login',
  failureFlash: 'Wrong username or password',
  successRedirect: '/',
  successFlash: "Welcome back!"
});
```

* implement auth in the route  
```js
// route
// ...

const userController = require('../controllers/userController');
const authController = require('../controllers/authController');

router.post('/register', 
  // validate field
  userController.validateRegister,
  // create user in db
  userController.create,
  // log user in
  authController.login
)
```

* create passport config for logged in user in `handlers/passport.js`  
```js
// handlers/passport.js
const passport = require('passport');
const mongoose = require('mongoose');
const User = mongoose.model('User');

// provided by mongoose passport plugin
passport.use(User.createStrategy());

// what to do with user
passport.serializeUser(User.serializeUser());
passport.deserializeUser(User.deserializeUser());
```

* import passport config in `app.js`  
```js
// app.js
// ...
require('./handlers/passport');
```

* by using passport and implementing `req.user = req.user.local` in `app.js`, this will provide use log in user auth data in template  

* in template render login user condition  
```pug
//- sample render for user
- if user
  .nav
    .name user.name
    a(href="logout")
- else
  .nav
    a(href="/register")    
    a(href="/login")
```

* create post route for login  
```js
// route
router.post('/login', authController.login);
```

### logout
* implement logout in `authController`  
```js
exports.logout = (req, res) => {
  req.logout();
  req.flash('success', 'Thank you, logging out!');
  res.redirect('/');
}
```

* create route to handle logout  
```js
// route
router.get('/logout', authController.logout);
```

### gravatar
* use gravatar(cloud service checking email and retrive stored image as avatar)  

* implement user model with mongoose `virtual field`
```js
// User.js
// ...
// use to retrive gravatar with user email; without exposing the actual email
// will hash with md5 and retrive from gravatar server
const md5 = require('md5');


// schema.virtual('field').get(function() { return ... });
// schema.virtual('field').set(function() { return ... });

User.virtual('gravatar').get(function() {
  // hash with email
  // use ES5 to get `this` context from User
  const hash = md5(this.email);
  return `https://gravatar.com/avatar/${hash}?s=200`;
});
```

> Mongo Virtuals are document properties that you can get and set but that do not get persisted to MongoDB. The getters are useful for formatting or combining fields, while setters are useful for de-composing a single value into multiple values for storage.

### protecting route
* protecting route from unauthorized user  
```js
// authController.js
// ...
express.isLoggedIn = (req, res, next) => {
  // check if user is in session; isAuthenticated is a method provided by passport
  if (req.isAuthenticated()) {
    return next(); // go to next middleware
  }
  // flash and redirect
  req.flash('error', 'not a user');
  res.redirect('/login');
}
```

* implement in route  
```js
// route
router.get('/add', authController.isLoggedIn, storeController.addStore);
```