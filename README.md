![image_squidhome@2x.png](http://i.imgur.com/RIvu9.png)

# sails-api-rest

### NOTE: This version has some limitations. It is very much a work in progress.


This module is a Waterline/Sails adapter, an early implementation of a rapidly-developing, tool-agnostic data standard.  Its goal is to provide a set of declarative interfaces, conventions, and best-practices for integrating with data sources through an API call to resource external to the service layer. Adapter configuration is distributed between the connection definition and the model that serves as an abstraction of the external API resource.

Strict adherence to an adapter specification enables the (re)use of built-in generic test suites, standardized documentation, reasonable expectations around the API for your users, and overall, a more pleasant development experience for everyone. And that's great, but this version isn't doesn't adhere so strictly. It mostly focuses on getting the job done in a small community of developers.

NOTE: The current version works only in the context of a Sails application. It will not work in a standalone Waterline implementation.


### Installation

To install this adapter, run:

```sh
$ npm install sails-api-rest
```

### Usage

This adapter exposes the following methods:

###### `find()`

+ **Status**
  + Ready for use; follow the configuration guidance below

###### `create()`

+ **Status**
  + Ready for use; follow the configuration guidance below

###### `update()`

+ **Status**
  + Not Planned. You can fork this and build out this method to your heart's content.

###### `destroy()`

+ **Status**
  + Not Planned. You can fork this and build out this method to your heart's content.

So how does this adapter work in the context of your Sails app? Like other Sails adapters, when applied to a model through its connection name, the find() and create() methods are immediately available to your model, in your controller code. For example, if you have a model named User, in your controller you can call

  ```javascript
  User.create().then(function(response) {
    res.send(response);
  })
  ```
The create method is defined in the adapter. Your model uses it as if it were defined there. The Configuration section below tells you how to set up your model so that it plays well with the adapter methods.

### Configuration

Configuration is distributed between the connection definition and the model that represents the API call. The connection definition is very simple. Here's the default that would go into your Sails environment file (/config/local.js, /config/env/development.js, qa.js or production.js):

```javascript
    "api_rest": {
        "adapter": "sails-api-rest",
        "protocol": "http"
    };

```
In this case, the object name, api_rest, can be anything you choose. This is the value you use in your model definition to ensure that Sails binds the model to this connection as a part of bootstrapping your app. The adapter must be sails-api-rest, because that is the name of the module directory in /node_modules. The protocol can be either http or https. This is the one value that can be overrridden in the model definition. This is handy especially for your local development environment, where you would conceivably be testing local services using the http protocol; yet a given API call might require https. In all cases, the CRUD method overrides this value with the model protocol value, if one is present in the model. If you are using https, name the certificate in the model, as shown below. __The certificate must be in a top level directory named 'certs'.__

The model configuration gives you a lot of flexibility; this standard is very much in progress. Here's an example of a model named User that is testing a Drupal web service in the same domain as the Sails app. It contains a small set of attributes, and three custom methods: requestBuilder, authHeaderBuilder, and responseHandler. These methods have a standard signature that must be followed, but the actual work of the methods is pretty customizable:

```javascript
module.exports = {

    connection: 'api_rest',
    attributes: {
        path: "/api/testUser/getUser.json",
        host: {
            localHost: "localhost",
            developmentHost: "your_dev_host",
            qaHost: "your_qa_host"
        },
        productionHost: "your_prod_host",
        sslCert: "your_cert"
    },

    /**
     * This API-specific request builder allows the adapter methods to be
     * merely declarative. This method is required, with the signature and functionality described below.
     *
     * This method includes only GET and POST requests. We don't expect tenant application will
     * use the XRAE service layer for data management, only data fetches.
     *
     * Here's where you build out the API route and query string or POST body, based on values
     * in the options argument. This method is called from the adapter verb, which has access to this
     * model and the Sails request object. Thus, the options argument can be pretty much whatever this
     * method requires.
     *
     * The execution begins with handling any parameter injection required in the route. Next is handling
     * the CRUD verb-specific requirements. A GET might need some query string parameters that are derived
     * from the Sails request object. A POST should need some body key-value pairs, at least. This method
     * return a request object that the adapter method uses to construct the node Superagent request.
     *
     * The adapter methods require the GET query string and POST body to have a defined format. While the node
     * Superagent can handle differently formatted objects, the adapter methods are written to handle one
     * format, in the interest of simplicity.
     *
     * The GET query string format is an object. For example:
     *  {metric_id: 4, timeframe: 'daily', word: 'to_your_mother'}
     *
     * The POST body format is identical to the query string object. It's an object containing
     * key-value pairs, where the key is not quoted, but the value, if a string, is. Of course,
     * the value could end up being an object, in which case you would follow the same convention.
     *
     *
     *
     * @param crudVerb
     * @param options
     * @returns {*}
     */


    requestBuilder: function(crudVerb, options) {
        var locCrudVerb = crudVerb.toLowerCase();
        var request = {};
        var crudOperations = {

            get: function() {

                var queryString = {};

                //Build query string parameters, based on the incoming options
                // Refer to the query string format in the block comment for this method.

                //Set the request.queryString to the query string you just built.
                // Since we're not doing anything here, we are returning the default value.
                if(request.endpoint.indexOf('?') === -1) {
                    request.endpoint += queryString;
                }
                return request;
            },

            post: function() {
                var body = {};

                //Build the POST body, based on the incoming options
                // Refer to the body format in the block comment for this method.

                //Set request.body to the body you just built
                request.body = body;
                return request;
            },


        };

        //Use this method to inject route parameters if needed, based on values in the options argument object
        var routeBuilder = function(cb) {
            //Since most routes aren't super duper complex, it's okay to use string manipulation
            // to handle the find - replace operation. Do that here.

            //Set request.endpoint
            //Here we return the default, since we aren't doing anything for this API call

            //Derive the api host, based on the current server environment. The model attributes
            // define this api call's host, based on whether the environment is local, development, qa or production.
            // Since we are defining the api host key in the attributes at the beginning of this model file, the
            // developer has some flexibility on how this is named.
            var procHost = process.env.HOSTNAME;
            var env = process.env.NODE_ENV;
            var apiHost;

            //TODO: figure out a better way to determine whether this is a local environment.
            if(procHost === 'xrae.local') {
                apiHost = options.host.localHost;
            }
            else {
                var hostIndex = env + "Host";
                apiHost = options.host[hostIndex];
            }

            request.route = options.path;
            request.endpoint = options.protocol + "://" + apiHost + request.route;
            return cb();
        };

        return routeBuilder(crudOperations[locCrudVerb]);
    },



    //Use this method to generate an authentication header. Here's where you can extract
    // a token from the Sails request and construct the header object used by the adapter
    // when it makes the API request. This also sends a basic authentication header, if required.
    authHeaderBuilder: function() {
        var authHeader = {};
        var basicAuthString = new Buffer("johnson:foo").toString('base64');
        authHeader.authentication = UserService.fullSessionId;
        authHeader.authorization = 'basic ' + basicAuthString;
        return authHeader;
    },

    responseHandler: function(response) {
        var result = {};
        result.name = response.body.name;
        result.uid = response.body.uid;
        return result;
    }
};
```
While this might look a bit daunting, this is pretty simple boilerplate. Let's look separately at the properties (connection and attributes) and custom methods.

 PROPERTIES

 The connection property is required, and uses the name of the connection defined in the Sails application environment configuration file.

 Two of the three attributes here are required, host and path. The host attribute is an object that contains paths for local, development, qa and production environments. This allows developers to define endpoints for all of their environments in a single configuration file. You are not limited to these environment names, but keep in mind that the prefix for the host name must be identical to the process.env.NODE_ENV value. The path is everything required for the request url path, not including a GET query string. For example, if the path were /api/user, the final endpoint would be http://localhost/api/user. Note that this does not include any GET query string parameters, or a POST body; constructing those is handled in the requestBuilder method. If the path is a parameterized route, e.g., api/user/:userId, the private routeBuilder method can be customized to inject the appropriate value. The sslCert attribute is not required. Nothing breaks if the attribute is not present. If it is present, the certification file __must be in a Sails app root directory named 'certs' (at the same level as /api, /config, etc)__.

 METHODS

We have three required methods: requestBuilder; authHeaderBuilder; and responseHandler. Let's look at each.

```javascript
requestBuilder: function(crudVerb, options) {
        var locCrudVerb = crudVerb.toLowerCase();
        var request = {};
        var crudOperations = {

            get: function() {

                var queryString = {};

                //Build query string parameters, based on the incoming options
                // Refer to the query string format in the block comment for this method.

                //Set the request.queryString to the query string you just built.
                // Since we're not doing anything here, we are returning the default value.
                if(request.endpoint.indexOf('?') === -1) {
                    request.endpoint += queryString;
                }
                return request;
            },

            post: function() {
                var body = {};

                //Build the POST body, based on the incoming options
                // Refer to the body format in the block comment for this method.

                //Set request.body to the body you just built
                request.body = body;
                return request;
            },


        };

        //Use this method to inject route parameters if needed, based on values in the options argument object
        var routeBuilder = function(cb) {
            //Since most routes aren't super duper complex, it's okay to use string manipulation
            // to handle the find - replace operation. Do that here.

            //Set request.endpoint
            //Here we return the default, since we aren't doing anything for this API call

            //Derive the api host, based on the current server environment. The model attributes
            // define this api call's host, based on whether the environment is local, development, qa or production.
            // Since we are defining the api host key in the attributes at the beginning of this model file, the
            // developer has some flexibility on how this is named. Note that the host prefix must match the value of process.env.NODE_ENV, if you use the approach below. The adapter doesn't care, since it relies on the value returned here.
            var procHost = process.env.HOSTNAME;
            var env = process.env.NODE_ENV;
            var apiHost;

            //TODO: figure out a better way to determine whether this is a local environment.
            if(procHost === 'xrae.local') {
                apiHost = options.host.localHost;
            }
            else {
                var hostIndex = env + "Host";
                apiHost = options.host[hostIndex];
            }

            request.route = options.path;
            request.endpoint = options.protocol + "://" + apiHost + request.route;
            return cb();
        };

        return routeBuilder(crudOperations[locCrudVerb]);
    }
```

There is a fair amount of commenting in the code itself, but here's some overview. This method is called in the adapter CRUD method. For example, for a GET request, the adapter find() method call requestBuilder, and send in two arguments: the verb itself (GET or POST; case does not matter), and an options object. Here's an example of the options argument:

```javascript
options = {
 protocol: protocol,
 host: model.attributes.host,
 path: model.attributes.path
}
```
You don't have to worry about this; the adapter constructs the options object. This just gives you some visibility. requestBuilder defines a set of private methods. The two crudOperation methods are well commented. These methods are customizable, and allow you to build out the GET query string or the POST body. In both cases, the CRUD-related methods change and return the entire request object.

requestBuilder's entry point is routeBuilder. This is a customizable method where you can inject values into a parameterized route. routeBuilder takes care of calling the appropriate crudOperation method.

The model has an authHeaderBuilder method, called separately from the adapter:

 ```javascript
 //Use this method to generate an authentication header. Here's where you can extract
// a token from the Sails request and construct the header object used by the adapter
// when it makes the API request. This also sends a basic authentication header, if required.
authHeaderBuilder: function() {
  var authHeader = {};
  var basicAuthString = new Buffer("johnson:foo").toString('base64');
  authHeader.authentication = UserService.fullSessionId;
  authHeader.authorization = 'basic ' + basicAuthString;
  return authHeader;
}

 ```
 This method is customizable, returning an object that satisfies the request authentication requirements. The documentation here gives examples of both an Authentication key-value pair, and the base64 username:password conversion for basic authentication. You can do whatever you need here to extract the token from the request. Now keep in mind that if you need a specific token, you will need to extract it in a controller or service to make it available to the model in this method. For example, as a part of bootstrapping the app, you could add some middleware that gets the token from the request, and sets it to a user service property. All services are accessible in the models; the request object is not.

Finally, the model has a responseHandler method, called upon the successful API call completion. This customizable method allows you to format the response body so that it is client-ready:

```javascript
responseHandler: function(response) {
        var result = {};
        result.name = response.body.name;
        result.uid = response.body.uid;
        return result;
    }
```

What you see here is an example of a response from a user API request that has among its properties a name and uid key. The result extracts these keys and returns this result.



