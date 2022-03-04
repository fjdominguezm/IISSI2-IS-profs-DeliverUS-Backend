# Introduction
We will learn how to define and implement the validation middleware and others on our backend Node.js Apps. Middlewares are intented to run some specific parts of our business logic such as:
* Validation of data from clients (the software that performs operations against the backend).
* Checking authorization.
* Checking permissions.
* Checking ownership of resources.

Secondly, we will learn how a Validation package will help us performing validation of data coming from clients.

## Prerequisites
* Keep in mind we are developing the backend software needed for DeliverUS project. Please, read project requirements found at: https://github.eii.us.es/IISSI2-IS/DeliverUS-ProjectRequirements/wiki/DeliverUS---Project-Requirements
* Software requirements for the developing environment con be found at: https://github.eii.us.es/IISSI2-IS/IISSI2-IS-Backend/wiki
  * The template project includes EsLint configuration so it should auto-fix formatting problems as soon as a file is saved.
  * The template project also includes the complete model of the App, which was completed in the previous lab.


# Exercices

## 1. Use repository as template, clone and setup
Press on "Use this template" to create your own repository based on this template. Afterwards clone your own repository by opening VScode and clone the base lab repository by opening Command Palette (Ctrl+Shift+P or F1) and `Git clone` this repository, or using the terminal and running
```PowerShell
git clone <url>
```

It may be necessary to setup your git username by running the following commands on your terminal:
```PowerShell
git config --global user.name "FIRST_NAME LAST_NAME"
git config --global user.email "MY_NAME@example.com"
```

In case you are asked if you trust the author, please select yes.

As in previous labs, it is needed to create a copy of the `.env.example` file, name it `.env` and include your environment variables.

Run `npm install` to download and install packages to the current project folder.

Check and run mariaDB server.
* Macos:
```Powershell
mysql.server start
```
* Windows:
  * If installed as service run `services.msc` and start the mariadb service
  * If installed as binary, locate your mariaDB binary and start.


## 2. Remember project structure

You will find the following elements. During this lab we will focus our attention on the `middleware` and `controllers\validation` folders:
* **`middlewares` folder: various checks needed such as authorization, permissions and ownership.**
* **`controllers\validation` folder: validation of data included in client requests. One validation file for each entity**
* `routes` folder: where URIs are defined and referenced to middlewares and controllers
* `controllers` folder: where business logic is implemented, including operations to the database
* `package.json`: scripts for running the server and packages dependencies including express, sequelize and others. This file is usally created with `npm init`, but you can find it already in your cloned project.
    * In order to add more package dependencies you have to run `npm install packageName --save` or `npm install packageName --save-dev` for dependencies needed only for development environment (p. e. nodemon). To learn more about npm please refer to [its documentation](https://docs.npmjs.com/cli/v7/commands/npm).
* `package-lock.json`: install exactly the same dependencies in futures deployments. Notice that dependencies versions may change, so this file guarantees to download and deploy the exact same tree of dependencies.
* `backend.js`: run http server, setup connections to Mariadb and it will initialize various components
* `.env.example`: example environment variables.
* `models` folder: where models entities are defined
* `migrations` folder: where the database schema is defined
* `seeders` folder: where database sample data is defined
* `config` folder: where some global config files are stored (to run migrations and seeders from cli)
* `example_api_client` folder: will store test requests to our Rest API
* `.vscode` folder: VSCode config for this project


## 3. Middlewares and validation middleware.
You will find middlewares at `middlewares` folder. One for each entity, one for handling file upload, and another for authentication/authorization.

At `Auth.js` file you will find two methods:
*  `hasRole` receives an array of roles names and check is there is a logged in user and if the user has the role needed.
* `isLoggedIn` checks if the user is logged in (the request includes a valid bearer token).

Next, you will find a middleware file for each entity. Depending on the entity and the functional requirements we will need to check if the current logged-in user has enough privileges to accomplish the requested operation.

For instance, when a user sends a request for creating a new product we will need to check that:
* the user is logged in
* the user has the role owner (since customers cannot create products)
* the product belongs to a restaurant the he/she owns (data includes a restaurantId which belongs to the owner that makes the request)
* the product data include valid values for each property in order to be created according to our information requirements.

Moreover, if the data may include files, you will find an `upload` middleware that will handle this and check that files are smaller than a particular size and are valid images.

In order to check all these requirements, we have to include each middleware method in the corresponding route:
```Javascript
app.route('/products')
    .post(
      middlewares.isLoggedIn,
      middlewares.hasRole('owner'),
      upload,
      middlewares.checkProductRestaurantOwnership,
      ProductValidation.create(),
      ProductController.create
    )
```

### 3.1. Validation middlewares
Validation middlewares are intended to check if the data that comes in a request fulfils the information requirements. Most of this requirements are defined at the database level, and were including when creating the schema on the migration files.
Some other requirements, are checked at the application layer. For instance, if you want to create a new restaurant, some images can be provided: logo image and hero image. These files should be image files and its size should be less than 10mbs. In order to check these other requirements we will use the `express-validator` package.

Notice that we will create a method for each endpoint that would require validation, usually a `create()` method for creating new data and a `update()` method for updating data.

More info about **using** middlewares can be found at Express documentation: https://expressjs.com/en/guide/using-middleware.html

More info about **writting** middlewares can be found at Express documentation: https://expressjs.com/en/guide/writing-middleware.html

### 3.2. Defining middlewares and validation middlewares for Restaurant routes
Open the file `routes/RestaurantRoutes.js`. You will find that routes are defined, but it is needed to define which middlewares will be called for each route. Notice that the route `PUT restaurants/:restaurantId` has been completed as an example.

Include middlewares needed for Restaurant routes according to the requirements of Deliverus project. For each route you should determine if:
* is it needed that a user is logged in?
* is it needed that the user has a particular role?
* is it needed that the restaurant belongs to the logged-in user (restaurant data should include a userId which belongs to the owner of that restaurant)
* is it needed that the restaurant data include valid values for each property in order to be created according to our information requirements.

### 3.3. Implement validation middleware for Restaurant create()
Open the file `controllers/validation/RestaurantValidation.js`. You will find the methods for validating data when creating `create` and when `updating`.
Restaurant properties are defined at database level. You can check the corresponding migration. Some validations are done at the app level, for instance we will include validations for check that email data is a valid email.

In order to add validations, follow this snippet:
```Javascript
create: () => {
    return [
        // array of checks
        check('attributeName').validationMethod(),
        check('otherAttributeName').validationMethod(),
    ]

```

For a comprehensive list of validations methods, see https://github.com/validatorjs/validator.js#validators


### 3.4. Check validation in controllers
When validation fails, it is passed to the following method in the middleware chain. In this case, the next method should be the controller method.

Within the controller method, we can check if any validation rule has been violated, and return the appropriate response. To this end, we can include the following at the beginning of the controller method:

```Javascript
const err = validationResult(req)

  if (err.errors.length > 0) {
    res.status(422).send(err)
  } else {
     // Controller business logic


````

Inspect `RestaurantController.js`, and see how validation is handled.


## 4. Test Restaurant routes, controllers and middlewares
Open ThunderClient extension (https://www.thunderclient.io/), and reload the collections by clicking on Collections → _**≡**_ menu→ reload. Collections are stored at

Click on Collections folder and you will find a set of requests with tests for all endpoints. Run all the collection, you will find at the right side if a test is successful or not. Some requests perform more than one test.
