# AWS Lambda deploy heroku buildpack

## The problem

While being a wonderful idea AWS Lambda lacks in terms of continous devlivery tooling and configuration externalisation. In current state Lambda offers us the same codeline and aliases, but there is no way to create a notion of 12Factor app [configuration](http://12factor.net/config). We could either hardcode (supposedly secret) configuration into lambda code, or store it in some external location (S3), but it brings issues in terms of env specific setup, and managing it along with software releases.

## Idea

This is an attempt to make configuration and pipelines management of lambdas a little bit less painful.
The idea is to use Heroku dyno as a basis for lambda creation.
Here is how the whole thing works:

  - The buildpack:
    - Installs AWS CLI
    - Installs a `deploy` process type in `Procfile`
    - Runs a matching buildpack for the software (Java/node.js)
  - To deploy lambda use `heroku run deploy`, that does:
    - Inject all available env variables to `env.properties` file
    - `env.properties` file is being added to lambda artifact (this file should be used in lambda code to retrieve configuration)
    - lambda artifact gets deployed to AWS

## Usage

### Set the buildpack

```
$ heroku buildpacks:set https://github.com/kubek2k/heroku-aws-lambda-buildpack
```

### Set env variables

  * `_LAMBDA_FUNCTION_ARN` - the ARN of lambda that is going to be deployed to. If multiple ARNs, separate with space
  * `_AWS_ACCESS_KEY_ID` - AWS access key id used for deployment
  * `_AWS_SECRET_ACCESS_KEY` - AWS secret key used for deployment
  * `_AWS_DEFAULT_REGION` - AWS region to be used

All env variables starting with `_` sign are not injected into `env.properties` file

**URGENT:** The user owning the AWS access key :point_up: should have permissions to update function code (`Action: ["lambda:UpdateFunctionCode"]`)

#### For Java Lambda

* `_LAMBDA_JAR_FILE` - path to .jar artifact that is a product of mvn invocation (normally it should be something like "target/lambda-with-dependencies.jar")

### Push the code

```
$ git push heroku master
```

### Deploy code with injected configuration to AWS

```
$ heroku run deploy
```

## Examples 

### Sample java lambda

https://github.com/kubek2k/sample-aws-java-lambda

### Sample node.js lambda

https://github.com/kubek2k/sample-aws-nodejs-lambda

## Pipelines support

Pipelines should work out of the box, but after each `promote` you have to remember to invoke `heroku run deploy` on application that was promoted to, so that changes are reflected on AWS (see [What could be done better](#what-could-be-done-better) ).

## Configuration value change

As well as promotion, configuration changes require `heroku run deploy` to be ran after (see [What could be done better](#what-could-be-done-better) ).

## What could be done better

First of all - this solution shouldn't be treated as neither something permanent nor final.
  * Ideally Amazon will someday address issues that this buildpack tries to solve, and I will be able to delete it :),
  * One could see a discrepancy between this solution and aformentioned 12factor app config (the real artifact deployed to lambda differs from env to env), but at least from outside it looks like config is externalized.
  * I can see a huge issue with a requirement of invoking `heroku run deploy` after each deploy/configuration change/promotion. This is something that is really hard to automate within heroku. One way to somehow deal with it is to scale deploy worker to 1 `Free` dyno - it will cause the deploy to be run multiple times within the day, but its `Free` so... ;)
