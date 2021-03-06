# saml2aws [![Build Status](https://travis-ci.org/Versent/saml2aws.svg?branch=master)](https://travis-ci.org/Versent/saml2aws) [![Build status - Windows](https://ci.appveyor.com/api/projects/status/ptpi18kci16o4i82/branch/master?svg=true)](https://ci.appveyor.com/project/davidobrien1985/saml2aws/branch/master)

CLI tool which enables you to login and retrieve [AWS](https://aws.amazon.com/) temporary credentials using SAML with [ADFS](https://msdn.microsoft.com/en-us/library/bb897402.aspx) or [PingFederate](https://www.pingidentity.com/en/products/pingfederate.html) Identity Providers.

This is based on python code from [
How to Implement a General Solution for Federated API/CLI Access Using SAML 2.0](https://blogs.aws.amazon.com/security/post/TxU0AVUS9J00FP/How-to-Implement-a-General-Solution-for-Federated-API-CLI-Access-Using-SAML-2-0).

The process goes something like this:

* Setup an account alias, either using the default or given a name
* Prompt user for credentials
* Log in to Identity Provider using form based authentication
* Build a SAML assertion containing AWS roles
* Exchange the role and SAML assertion with [AWS STS service](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html) to get a temporary set of credentials
* Save these creds to an aws profile named "saml"

# Requirements

* Identity Provider
  * ADFS (2.x or 3.x)
  * PingFederate + PingId
  * Okta + (Duo, SMS, TOTP)
  * KeyCloak + (TOTP)
* AWS SAML Provider configured

# Caveats

Aside from Okta, most of the providers in this project are using screen scraping to log users into SAML, this isn't ideal and hopefully vendors make this easier in the future. In addition to this there are some things you need to know:

1. AWS only permits session tokens being issued with a duration of up to 3600 seconds (1 hour), this is constrained by the [STS AssumeRoleWithSAML API](http://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithSAML.html) call and `DurationSeconds` field.
2. Every SAML provider is different, the login process, MFA support is pluggable and therefore some work may be needed to integrate with your identity server

# Install

## OSX

If you're on OSX you can install saml2aws using homebrew!

```
brew tap versent/homebrew-taps
brew install saml2aws
```

## Windows

If you're on Windows you can install saml2aws using chocolatey!

```
choco install saml2aws
saml2aws --version
```

# Dependency Setup

Install the AWS CLI see https://docs.aws.amazon.com/cli/latest/userguide/installing.html, in our case we are using [homebrew](http://brew.sh/) on OSX.

```
brew install awscli
```

Configure an empty default profile with your region of choice, note the credentials will be overwritten when you first login and are supplied to unsure `~/.aws/credentials` file is created.

```
$ aws configure
AWS Access Key ID [None]: test
AWS Secret Access Key [None]: test
Default region name [None]: us-west-2
Default output format [None]:
```

# Usage

```
usage: saml2aws [<flags>] <command> [<args> ...]

A command line tool to help with SAML access to the AWS token service.

Flags:
      --help                   Show context-sensitive help (also try --help-long and --help-man).
      --version                Show application version.
      --verbose                Enable verbose logging
  -i, --provider=PROVIDER      This flag it is obsolete see https://github.com/Versent/saml2aws#adding-idp-accounts.
  -a, --idp-account="default"  The name of the configured IDP account
      --idp-provider=IDP-PROVIDER
                               The configured IDP provider
      --mfa="Auto"             The name of the mfa
  -s, --skip-verify            Skip verification of server certificate.
      --url=URL                The URL of the SAML IDP server used to login.
      --username=USERNAME      The username used to login.
      --role=ROLE              The ARN of the role to assume.
      --aws-urn=AWS-URN        The URN used by SAML when you login.
      --skip-prompt            Skip prompting for parameters during login.

Commands:
  help [<command>...]
    Show help.


  configure
    Configure a new IDP account.


  login [<flags>]
    Login to a SAML 2.0 IDP and convert the SAML assertion to an STS token.

        --password=PASSWORD  The password used to login.
    -p, --profile="saml"     The AWS profile to save the temporary credentials

  exec [<flags>] [<command>...]
    Exec the supplied command with env vars from STS token.

        --password=PASSWORD  The password used to login.
    -p, --profile="saml"     The AWS profile to save the temporary credentials

```

# Configuring IDP Accounts

This is the *new* way of adding IDP provider accounts, it enables you to have named accounts with whatever settings you like and supports having one *default* account which is used if you omit the account flag. This replaces the --provider flag and old configuration file in 1.x.

To add a default IdP account to saml2aws just run the following command and follow the prompts.

```
$ saml2aws configure

Please choose the provider you would like to use:

[ 0 ]:  ADFS

[ 1 ]:  ADFS2

[ 2 ]:  JumpCloud

[ 3 ]:  KeyCloak

[ 4 ]:  Okta

[ 5 ]:  Ping

Selection: 3

URL []: https://id.example.com/auth/realms/master/protocol/saml/clients/amazon-aws
Username []: mark@wolfe.id.au

Configuration saved for IDP account: default
```

Then to login using this account.

```
saml2aws login
```

You can also add named accounts, below is an example where I am setting up an account under the `wolfeidau` alias, again just follow the prompts.

```
saml2aws configure -a wolfeidau
```

You can also configure the account alias without prompts.

```
saml2aws configure -a wolfeidau --idp-provider KeyCloak --username mark@wolfe.id.au \
  --url https://keycloak.wolfe.id.au/auth/realms/master/protocol/saml/clients/amazon-aws --skip-prompt
```


Then your ready to use saml2aws.

# Example

Log into a service (without MFA).

```
$ saml2aws login
Using IDP Account default to access Ping https://id.example.com
To use saved password just hit enter.
Username [mark.wolfe@example.com]:
Password: ************

Authenticating as mark.wolfe@example.com ...
Selected role: arn:aws:iam::123123123123:role/AWS-Admin-CloudOPSNonProd
Requesting AWS credentials using SAML assertion
Saving credentials
Logged in as: arn:aws:sts::123123123123:assumed-role/AWS-Admin-CloudOPSNonProd/wolfeidau@example.com

Your new access key pair has been stored in the AWS configuration
Note that it will expire at 2016-09-19 15:59:49 +1000 AEST
To use this credential, call the AWS CLI with the --profile option (e.g. aws --profile saml ec2 describe-instances).
```

Log into a service (with MFA).

```
$ saml2aws login
Using IDP Account default to access Ping https://id.example.com
To use saved password just hit enter.
Username [mark.wolfe@example.com]:
Password: ************

Authenticating as mark.wolfe@example.com ...
Enter passcode: 123456

Selected role: arn:aws:iam::123123123123:role/AWS-Admin-CloudOPSNonProd
Requesting AWS credentials using SAML assertion
Saving credentials
Logged in as: arn:aws:sts::123123123123:assumed-role/AWS-Admin-CloudOPSNonProd/wolfeidau@example.com

Your new access key pair has been stored in the AWS configuration
Note that it will expire at 2016-09-19 15:59:49 +1000 AEST
To use this credential, call the AWS CLI with the --profile option (e.g. aws --profile saml ec2 describe-instances --region us-east-1).
```

# Building

To build this software on osx clone to the repo to `$GOPATH/src/github.com/versent/saml2aws` and ensure you have `$GOPATH/bin` in your `$PATH`.

If you don't have glide installed you can install it using [homebrew](http://brew.sh/).

```
brew install glide
```

Then to build the software just run.

```
make
```

Install the binary to `$GOPATH/bin`.

```
make install
```

To release run.

```
make release
```

# Environment vars

The exec sub command will export the following environment variables.

* AWS_ACCESS_KEY_ID
* AWS_SECRET_ACCESS_KEY
* AWS_SESSION_TOKEN
* AWS_SECURITY_TOKEN
* EC2_SECURITY_TOKEN

# Dependencies

This tool would not be possible without some great opensource libraries.

* [goquery](https://github.com/PuerkitoBio/goquery) html querying
* [etree](https://github.com/beevik/etree) xpath selector
* [kingpin](https://github.com/alecthomas/kingpin) command line flags
* [aws-sdk-go](https://github.com/aws/aws-sdk-go) AWS Go SDK
* [go-ini](https://github.com/go-ini/ini) INI file parser
* [go-ntlmssp](https://github.com/Azure/go-ntlmssp) NTLM/Negotiate authentication

# License

This code is Copyright (c) 2018 [Versent](http://versent.com.au) and released under the MIT license. All rights not explicitly granted in the MIT license are reserved. See the included LICENSE.md file for more details.
