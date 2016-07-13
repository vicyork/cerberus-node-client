# Cerberus Client (node)

This is a client for interacting with a [Cerberus backend](http://bitbucket.nike.com/projects/CPE/repos/cerberus-management-service/browse). It can be used in Amazon EC2 instances and Amazon Lambdas.

# Installation

You have two installation options. The private Nike npm registry, or directly from bitbucket as a git package.

## Private npm

Using the private npm repo for Nike has the advantage of the cerberus client's dependency entry looking like a normal dependency, as well as a simplified installation command. Two use the private repo, you need an `.npmrc` file with the this contents

```
registry=http://artifactory.nike.com/artifactory/api/npm/npm-nike
```

The `.npmrc` file can either be **project-level**, meaning it is in the root of your project, alongside the `package.json` file, or it can be in your user directory `~/.npmrc`. The per-project file simplifies your build process, since the build machine doesn't need any additional configuration, but it must be mode `600` (`chmod 600 .npmrc`) and it must be duplicated in every project you want to use it in. The user directory file means your build machine needs the same `.npmrc` file.

It's up to you which one to use, both work. Once that is done, install from npm as normal.

```
npm install --save cerberus-node-client
```


## Install as git package

Installing with a git package has the advantage of not required **any** additional configuration on your machine or the build machine, but it requires that you have read access to the repository and produces a dependency entry in `package.json` that includes the entire url. If you don't have read permission on this repository, this method won't work for you.

To install, just run the following command

```
npm install --save git+http://bitbucket.nike.com/scm/cer/cerberus-node-client.git
```

## Usage with the AWS SDK

This library does not declare the [AWS Node SDK](https://github.com/aws/aws-sdk-js) as a dependency, but it is required to use. You must pass in the AWS SDK as a constructor parameter

```javascript
var AWS = require('aws-sdk')

var client = cerberus({
    aws: AWS,
    hostUrl: 'https://test.cerberus.nikecloud.com',
    lambdaContext: context
  })
```

This is done to make packaging for Lambdas possible without bundling the AWS SDK, since Lambdas have it provided for them automatically. This helps reduce the size of Lambda's, which are billed partially on their size, and which have a 75GB limit per account.

It also ensures that locally or globally scoped AWS configurations can be easily used by the cerberus client.

## Authentication

The cerberus client supports three different configuration modes.

* Environment Variables - This is useful for running locally without changing out code, since developer machines cannot posses the IAM roles necessary to decrypt Cerberus authentication responses.
* Lambda Context - Pass in the `context` from your Lambda Handler `handler(event, context)` as the `lambdaContext` parameter.
* EC2 - This is the default mode, it will be used if the other two are not present.

These configuration modes determine how the client will authenticate with Cerberus.

Cerberus will attempt to authenticate one its first call. The authentication result will be stored and reused. If the token has expired on a subsequent call, authentication will be repeated with the original configuration. You should not have to worry about authentication or token expiration; just use the client.

# Constructing the client

```javascript
var cerberus = require('cerberus-node-client')
var AWS = require('aws-sdk')

var client = cerberus({
    // REQUIRED -  The AWS SDK
    aws: AWS,

    // string, The cerberus URL to use.
    // OVERRIDDEN by process.env.CERBERUS_ADDR
    // Either this or the env variable is required
    hostUrl: 'https://test.cerberus.nikecloud.com',


    // The context given to the lambda handler
    lambdaContext: context,

    // boolean, defaults to false. When true will console.log many operations
    debug: true,

    // This will be used as the cerberus X-Vault-Token if supplied
    // OVERRIDDEN by process.env.CERBERUS_TOKEN
    // If present, normal authentication with cerberus will be skipped
    // You should normally only be using this in testing environments
    // When developing locally, it is easier to use process.env.CERBERUS_TOKEN
    token: 'Some_Auth_Token'
  })
```

# Using the Client

This client should be compatible with node 0.12.x (this has not yet been tested, please submit bug reports if you run into issues). It supports both node-style `cb(err [, data])` callbacks and promises.

To use the promise API omit the callback parameter (always the last one), and ensure `global.Promise` supports constructing promises with `new Promise()`. Promises will be returned from all client methods in this case; otherwise, `undefined` will be returned.

**KeyPaths** below are relative to the Cerberus root. This is shown as the `path` value when looking at a Safety Deposit Box (SDB). You must include the full path (including `app` or `shared`) not just the name of the SDB.

* `get(keyPath [, callback])` - Read the contents of the **keyPath**. Returns an object
* `set(keyPath, data [, callback])` - Set the contents of the **keyPath**. Returns `data`
* `list(keypath [, callback])` - List the paths available at the **keyPath**. Returns an array
* `delete(keyPath [, callback])` - Delete the contents of the **keyPath**. Returns an object