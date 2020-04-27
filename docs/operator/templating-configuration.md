---
title: Using templates for injecting dynamic configuration
shortTitle: Templating configuration
weight: 10
---

## Background

When configuring a `Vault` object via the `externalConfig` property, sometimes it's convenient (or necessary) to inject settings that are only known at runtime, (e.g. secrets that you don't want to store in source control, or dynamic resources managed elsewhere) or computations based on multiple values (string or arithmetic operations). For these cases, the operator supports parameterized templating that will be evaluated by the `vault-configurer` component and inject the rendered configuration into Vault.

This templating is based on [Go templates](https://golang.org/pkg/text/template/), extended by [Sprig](https://github.com/Masterminds/sprig), with some custom functions available specifically for bank-vaults.

### Special characters

To avoid confusion and potential parsing errors, the templates don't use the default delimiters that Go templates use (`{{` and `}}`). Instead, it uses `${` for the right delimiter, and `}` for the left one. Additionally, to quote parameters being passed to functions, surround them with backticks (`` ` ``) instead. For example, to call the `env` function, you can use this on your manifest:

```
password: "${ env `MY_ENVIRONMENT_VARIABLE` }"
```

In this case, `vault-configurer` will evaluate the value of `MY_ENVIRONMENT_VARIABLE` at runtime (assuming it was properly injected) and set that to the `password` field.

## Sprig

In addition to the default functions in Go templates, the operator also makes the Sprig library of functions available to use in configuration. The documentation for Sprig can be found [here](http://masterminds.github.io/sprig/).

One thing to keep in mind is that some Sprig functions might return values other than strings, like lists or maps. Make sure that the function you're calling returns a string to avoid unintentionally generating an invalid configuration.

## Custom functions

To provide functionality that's more Kubernetes-friendly and cloud-native, bank-vaults provides a few additional funtions not available in Sprig or Go. The functions and their parameters (in the order they should go in the function) are documented below.

### `file`

Reads the content of a file from disk at the given path and returns it. This assumes that the file exists and it's mounted, readable by `vault-configurer`.

Parameter | Type   | Required
----------|--------|---------
path      | String | Yes

### `awskms`

Takes a base64-encoded, KMS-encrypted string and returns the decrypted string. Additionally, the function takes an optional second parameter for any encryption context that might be required for decrypting. If any encryption context is required, the function will take any number of additional parameters, each of which should be a key-value pair (separated by a `=`), corresponding to the full context.

Note: this function assumes that the `vault-configurer` pod has the appropriate AWS IAM credentials and permissions to decrypt the given string. They can be injected via Kubernetes secrets as environment variables, an EC2 instance role, [kube2iam](https://github.com/jtblin/kube2iam), etc.

Parameter         | Type                           | Required
------------------|--------------------------------|---------
encodedString     | Base64-encoded string          | Yes
encryptionContext | Variadic list of strings       | No

### `gcpkms`

Takes a base64-encoded string, encrypted with a GCP symmetric key and returns the decrypted string.

Note: this function assumes that the `vault-configurer` pod has the appropriate GCP IAM credentials and permissions to decrypt the given string. They can be injected via Kubernetes secrets as environment variables, acquired via a service account authentication, etc.

Parameter     | Type                  | Required
--------------|-----------------------|---------
encodedString | Base64-encoded string | Yes
projectId     | String                | Yes
location      | String                | Yes
keyRing       | String                | Yes
key           | String                | Yes