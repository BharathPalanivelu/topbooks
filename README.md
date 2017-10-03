# Overview

We are designing a new service called TopBooks, similar to already existing services like Google Books, Gooreads, etc.

Our service will provide our users with information on authors and books. We already have a Node.js backend that can provide us with the required content for our service. Our objective now will be the design of three RESTful APIs, Identity API (v1), Authors API (v1) and Books API (v1), which will consume the data supplied by the Node.js backend. We will build and deploy two API proxies in Apigee Edge, where we will be offloading common API gateway concerns such as: security, traffic management, etc. For demonstration purposes we will be deploying the Node.js backend as another API proxy. 

We are going to have four git repositories, one for each of the Apigee API proxies that we are going to implement and an additional one with a common set of policies, resources, fragments, etc will be available. This last repository will be added as a git submodule in the Author API (v1) and Book API (v1) proxies.

- [Identity API Proxy](https://github.com/apichick/topbooks-identity-api-v1)
- [Author API Proxy](https://github.com/apichick/topbooks-author-api-v1) (TODO)
- [Book API Proxy](https://github.com/apichick/topbooks-book-api-v1) (TODO)
- [Mock API Proxy](https://github.com/apichick/topbooks-mock-api-v1)
- [Common](https://github.com/apichick/topbooks-common)

We will be using a gulp script to build, deploy and test our API proxies. In addition to that, this script will have some additional tasks for importing/exporting environment configuration in Apigee Edge. The integrations tests will be written using apickli, a REST API integration testing framework.

# API Proxy Structure

The main API proxies (identity-api-v2, book-api-v1, author-api-v1) will have the folder structure depicted below.

```
  +-- book-api-v1
    |-- apiproxy
    |   |-- policies
    |   |   |-- *.xml
    |   |-- proxies
    |   |   |-- default.xml
    |   |-- resources
    |       |-- jsc
    |           |-- *.js
    |   |-- targets
    |   |   |-- default.xml
    |   |-- book-api-v1.xml
    |-- common (submodule)
    |   |-- apiproxy
    |   |  |-- policies
    |   |  |   |-- *.xml
    |   |  |-- resources
    |   |  |   |-- jsc
    |   |          |-- *.js 
    |   |-- partials
    |   |   |-- *.ejs 
    |   |-- test
    |       |-- integration
    |           |-- features
    |               |-- *.feature
    |               |-- step_definitions
    |                   |-- *.js
    |-- test
    |  |-- integration
    |      |-- feature
    |          |-- *.ejs
    |          |-- step_definitions
    |              |-- *.js
    |-- settings.json
    |-- gulpfile.babel.js
    |-- package.json
```

In the settings.json file we will setting the parameters for the API proxy and the tests. We can specify a set of default parameters, which can be overritten for an specific organization and environment. 

```
{
    "default": {
        "scheme": "http",
        "domain": "${org}-${env}.apigee.net/identity/v1${deploymentSuffix}",
        "apiProxyId": "01"
    },
    "ORGANIZATION": {
        "ENVIRONMENT": {
            "clientId": "DEVELOPER APP CONSUMER KEY",
            "clientSecret": "DEVELOPER APP CONSUMER SECRET"
        }
    }
}
```

As shown above, the options passed in the command line (organization, environment and deploymentSuffix) can be used when setting the parameter values.

Under the common/apiproxy folder we will be storing the policies, resources, etc common to both API proxies. Inside common/partials we will keep a set of ejs templates, containing fragments (eg: a sequence of steps used in the proxy PreFlow of some of the proxies, a default fault rules, etc) that can be easily included inside the different API proxy XML files, as it is shown below. 

We could use the following fragment in common/partials/SecurityAndTrafficManagement.ejs...

```
<Step>
    <Name>SpikeArrest</Name>
    <Condition>environment.name = "prod" OR x-reset-spike-arrest-count = true</Condition>
</Step>
<Step>
    <Name>OAuthV2.VerifyAccessToken</Name>
    <Condition>request.verb != "GET" OR proxypath.suffix MatchesPath "/(ping|status)"</Condition>
</Step>
<Step>
    <Name>Quota</Name>
    <Condition>request.verb != "GET" OR proxypath.suffix MatchesPath "/(ping|status)"</Condition>
</Step>
```

... in the proxy endpoint (book-api-v1/apiproxy/proxies/default.xml) of our Book API

```
  <ProxyEndpoint name="default">
    ...
    <PreFlow>
      <%- include(partialsDir + '/SecurityAndTrafficManagement');
    </PreFlow>
    ...
  </ProxyEndpoint>
```

Always use the variable partialsDir + '/<templatePath>' in the includes.

In some cases, it might be neccesary to create a set of policies, proxy / target endpoint descriptors from a template. This can be easily handling following this simple steps:

1. Create an ejs template (eg. Error.ejs) in the common/partials folder or directly in apiproxy/{policies|proxies|targets}. See an example of template below.

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AssignMessage async="false" continueOnError="false" enabled="true" name="<%= name %>">
    <AssignVariable>
        <Name>flow.error.message</Name>
        <Value><%= message %></Value>
    </AssignVariable>
    <AssignVariable>
        <Name>flow.error.code</Name>
        <Value><%= status %>.<%= apiProxyId %>.<%= sequenceNumber %></Value>
    </AssignVariable>
    <AssignVariable>
        <Name>flow.error.status</Name>
        <Value><%= status %></Value>
    </AssignVariable>
    <AssignVariable>
        <Name>flow.error.error</Name>
        <Value><%= error %></Value>
    </AssignVariable>
</AssignMessage>
```

2. Create a JSON file having the same base name as the template (eg. Error.json) in the directory where you want files generated from the template created. This file will contain an array with as many items as files need to be generated from the template created in step 1. Each of the items will be a json object containing an attribute "name", that will be the name of the policy or proxy/target endpoint and the base name of the file that will be created, and the set of other attributes that will need to be replaced in the template. Have a look at a sample JSON file that can be used with the sample template shown above. 

```
[{
    "name": "AssignMessage.Error.BadRequest",
    "message": "Bad Request",
    "status": "400",
    "sequenceNumber": "001",
    "error": "bad_request"
}, {
    "name": "AssignMessage.Error.InvalidClient",
    "message": "Invalid client",
    "status": "400",
    "sequenceNumber": "002",
    "error": "invalid_client"
}, {
    "name": "AssignMessage.Error.Unauthorized",
    "message": "Unauthorized",
    "status": "401",
    "sequenceNumber": "001",
    "error": "unauthorized"
}, {
    "name": "AssignMessage.Error.NotFound",
    "message": "Not Found",
    "status": "404",
    "sequenceNumber": "001",
    "error": "not_found"
}, {
    "name": "AssignMessage.Error.SpikeArrestViolation",
    "message": "Too Many Requests",
    "status": "429",
    "sequenceNumber": "001",
    "error": "spike_arrest_violation"
}, {
    "name": "AssignMessage.Error.ProxyInternalServerError",
    "message": "Internal Server Error",
    "status": "500",
    "sequenceNumber": "001",
    "error": "proxy_internal_server_error"
}]
```

# Gulp tasks

Run the following command to see the available commands in the Gulp script.

```
$ gulp help

Commands:
  create-bundle          Create API proxy / shared flow bundle
  deploy                 Deploy API proxy / shared flow bundle
  deploy-and-test        Deploy API proxy and execute integration tests
  test                   Execute integration tests for API proxy
  export-env-config      Export environment configuration
  import-env-config      Import environment configuration
  export-publish-config  Export publish configuration
  import-publish-config  Export publish configuration

Options:
  --help  Show help                                                    [boolean]

```

# Create API proxy bundle

```
$ gulp create-bundle help

Options:
  --help                   Show help                                   [boolean]
  --deployment-suffix, -s  Deployment suffix       [string] [default: "-user"]
```

Creates the API proxy bundle and saves it to dist/apiproxy.zip. By default the username of the machine where the script is being run prepended by "-" will be added to the proxy name (taken from the name attribute in the API proxy descriptor) and the proxy endpoint base paths. We can pass a different suffix using the command line option --deployment-suffix (-d). The use of the suffix is specially convenient when you have several team members working on different versions of the same API proxy and you don't want conflicts to raise when they are deploying to the same organization and environment.

# Deploy

```
$ gulp deploy help

Options:
  --help                       Show help                               [boolean]
  --organization, -o           Apigee organization           [string] [required]
  --environment, -e            Apigee environment            [string] [required]
  --management-server-url, -m  Management server URL
                      [string] [default: "https://api.enterprise.apigee.com/v1"]
  --user, -u                   Apigee user                              [string]
  --password, -p               Apigee password                          [string]
  --override, -w               Create a new revision of the API proxy
                                                      [boolean] [default: false]
  --deployment-suffix, -s      Deployment suffix   [string] [default: "-user"]
  --debug, -d                  Make the script verbose during the operation
                                                      [boolean] [default: false]
```

Create the API proxy bundle and deploy it to an specific organization and environment that are passed as command line options. The Apigee user and password are not required in case those credentials have already been set in ~/.netrc as follows:

```
  machine <management-api-hostname>
  login <apigee-user>
  password <apigee-password>
```

By default, if the proxy was already deployed, the existing revision will be updated. If the command line option --override (-w) is used, a new revision will be created.

# Test

```
$ gulp test help

Options:
  --help                   Show help                                   [boolean]
  --organization, -o       Apigee organization               [string] [required]
  --environment, -e        Apigee environment                [string] [required]
  --deployment-suffix, -s  Deployment suffix       [string] [default: "-user"]
  --debug, -d              Make the script verbose during the operation
                                                      [boolean] [default: false]
```

The BBD tests will be run against the API proxy that is deployed in an specific organization and environment.

# Deploy & Test

```
$ gulp deploy-and-test help

Options:
  --help                       Show help                               [boolean]
  --organization, -o           Apigee organization           [string] [required]
  --environment, -e            Apigee environment            [string] [required]
  --management-server-url, -m  Management server URL
                      [string] [default: "https://api.enterprise.apigee.com/v1"]
  --user, -u                   Apigee user                              [string]
  --password, -p               Apigee password                          [string]
  --override, -w               Create a new revision of the API proxy
                                                      [boolean] [default: false]
  --deployment-suffix, -s      Deployment suffix   [string] [default: "-user"]
  --debug, -d                  Make the script verbose during the operation
                                                      [boolean] [default: false]
```

# Export Environment Configuration

```
$ gulp export-env-config help

Options:
  --help                       Show help                               [boolean]
  --organization, -o           Apigee organization           [string] [required]
  --environment, -e            Apigee environment            [string] [required]
  --file, -f                   file                          [string] [required]
  --management-server-url, -m  Management server URL
                      [string] [default: "https://api.enterprise.apigee.com/v1"]
  --user, -u                   Apigee user                              [string]
  --password, -p               Apigee password                          [string]
  --debug, -d                  Make the script verbose during the operation
                                                      [boolean] [default: false]
```
Exports the environment configuration (caches, keyvaluemaps, references, targets, virtualhosts) and saves it as a JSON object in the file passed as a command line option.

# Import Environment Configuration

```
$ gulp import-env-config help

Options:
  --help                       Show help                               [boolean]
  --organization, -o           Apigee organization           [string] [required]
  --environment, -e            Apigee environment            [string] [required]
  --management-server-url, -m  Management server URL
                      [string] [default: "https://api.enterprise.apigee.com/v1"]
  --user, -u                   Apigee user                              [string]
  --password, -p               Apigee password                          [string]
  --file, -f                   file                                     [string]
  --keystore-dir, -k           Directory containing the private keys /
                               certificates to upload      [default: "keystore"]
  --debug, -d                  Make the script verbose during the operation
                                                      [boolean] [default: false]
```

Imports the environment configuration (caches, keyvaluemaps, references, targets, virtualhosts) supplied in a file passed as a command line option. In addition to that it is also possible to expecify a directory containing the private keys and certificates of the keystores/truststores to be created. The structure inside that directory has to be the following one

```
+-- keystore
    |-- keystore1
    |    |-- cert.pem
    |    |-- key.pem
    |-- truststore1
    |    |-- cert.pem    
    |-- keystore2
    |    |-- cert.pem
    |    |-- key.pem
    ...
```

The name of the subdirectories will be equal to the name of the keystore/trustore that will be created.

# Export Publish Configuration

```
$ gulp export-publish-config help

Options:
  --help                       Show help                               [boolean]
  --organization, -o           Apigee organization           [string] [required]
  --file, -f                   file                          [string] [required]
  --management-server-url, -m  Management server URL
                      [string] [default: "https://api.enterprise.apigee.com/v1"]
  --user, -u                   Apigee user                              [string]
  --password, -p               Apigee password                          [string]
  --debug, -d                  Make the script verbose during the operation
                                                      [boolean] [default: false]
```

Exports the "Publish" configuration (apiproducts, developers, companies, apps) and saves it as a JSON object in the file passed as a command line option.

# Import Publish Configuration

```
$ gulp import-publish-config help

Options:
  --help                       Show help                               [boolean]
  --organization, -o           Apigee organization           [string] [required]
  --file, -f                   file                          [string] [required]
  --management-server-url, -m  Management server URL
                      [string] [default: "https://api.enterprise.apigee.com/v1"]
  --user, -u                   Apigee user                              [string]
  --password, -p               Apigee password                          [string]
  --debug, -d                  Make the script verbose during the operation
                                                      [boolean] [default: false]
```

Imports the "Publish" configuration (apiproducts, developers, companies, apps) supplied in a file passed as a command line option.
