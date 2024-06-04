# Restore S3 Cache

[![Step changelog](https://shields.io/github/v/release/bitrise-steplib/bitrise-step-restore-s3-cache?include_prereleases&label=changelog&color=blueviolet)](https://github.com/bitrise-steplib/bitrise-step-restore-s3-cache/releases)

Restores build cache using a cache key. This Step needs to be used in combination with **Save S3 Cache**.

<details>
<summary>Description</summary>

Restores build cache using a cache key. This Step needs to be used in combination with **Save S3 Cache**.

#### About key-based caching

Key-based caching is a concept where cache archives are saved and restored using a unique cache key. One Bitrise project can have multiple cache archives stored simultaneously, and the **Restore S3 Cache Step** downloads a cache archive associated with the key provided as a Step input.

Caches can become outdated across builds when something changes in the project (for example, a dependency gets upgraded to a new version). In this case, a new (unique) cache key is needed to save the new cache contents. This is possible if the cache key is dynamic and changes based on the project state (for example, a checksum of the dependency lockfile is part of the cache key). If you use the same dynamic cache key when restoring the cache, the Step will download the most relevant cache archive available.

Key-based caching is platform-agnostic and can be used to cache anything by carefully selecting the cache key and the files/folders to include in the cache.

#### Templates

The Step requires a string key to use when downloading a cache archive. In order to always download the most relevant cache archive for each build, the cache key input can contain template elements. The Step evaluates the key template at runtime and the final key value can change based on the build environment or files in the repo.

The following variables are supported in cache keys input:

- `cache-key-{{ .Branch }}`: Current git branch the build runs on
- `cache-key-{{ .CommitHash }}`: SHA-256 hash of the git commit the build runs on
- `cache-key-{{ .Workflow }}`: Current Bitrise workflow name (eg. `primary`)
- `{{ .Arch }}-cache-key`: Current CPU architecture (`amd64` or `arm64`)
- `{{ .OS }}-cache-key`: Current operating system (`linux` or `darwin`)

Functions available in a template:

`checksum`: This function takes one or more file paths and computes the SHA256 [checksum](https://en.wikipedia.org/wiki/Checksum) of the file contents. This is useful for creating unique cache keys based on files that describe content to cache.

Examples of using `checksum`:
- `cache-key-{{ checksum "package-lock.json" }}`
- `cache-key-{{ checksum "**/Package.resolved" }}`
- `cache-key-{{ checksum "**/*.gradle*" "gradle.properties" }}`

`getenv`: This function returns the value of an environment variable or an empty string if the variable is not defined.

Examples of `getenv`:
- `cache-key-{{ getenv "PR" }}`
- `cache-key-{{ getenv "BITRISEIO_PIPELINE_ID" }}`

#### Key matching and fallback keys

The most straightforward use case is that a cache archive is downloaded and restored if the provided key matches a cache archive uploaded previously using the Save Cache Step. Stored cache archives are scoped to the Bitrise project. Builds can restore caches saved by any previous Workflow run on any Bitrise Stack.

It's possible to define more than one key in the cache keys input. You can specify additional keys by listing one key per line. The list is in priority order, so the Step will first try to find a match for the first key you provided, and if there is no cache stored for the key, it will move on to find a match for the second key (and so on).

In addition to listing multiple keys, each key can be a prefix of a saved cache key and still get a matching cache archive. For example, the key `my-cache-` can match an existing archive saved with the key `my-cache-a6a102ff`.

We recommend configuring the keys in a way that the first key is an exact match to a checksum key, and to use a more generic prefix key as a fallback:

```
inputs:
  key: |
    npm-cache-{{ checksum "package-lock.json" }}
    npm-cache-
```

#### Related steps

[Save cache](https://github.com/bitrise-steplib/bitrise-step-save-cache/)

</details>

## 🧩 Get started

Add this step directly to your workflow in the [Bitrise Workflow Editor](https://devcenter.bitrise.io/steps-and-workflows/steps-and-workflows-index/).

You can also run this step directly with [Bitrise CLI](https://github.com/bitrise-io/bitrise).


Check out [Workflow Recipes](https://github.com/bitrise-io/workflow-recipes#-key-based-caching-beta) for platform-specific examples!

#### Restore and save cache using a key that includes a checksum

```yaml
steps:
- restore-s3-cache@1:
    inputs:
    - key: npm-cache-{{ checksum "package-lock.json" }}

# Build steps

- save-s3-cache@1:
    inputs:
    - key: npm-cache-{{ checksum "package-lock.json" }}
    - paths: node_modules
```

#### Use fallback key when exact cache match is not available

```yaml
steps:
- restore-s3-cache@1:
    inputs:
    - key: |-
        npm-cache-{{ checksum "package-lock.json" }}
        npm-cache-
```

The Step will look up the first key (`npm-cache-a982ee8f` for example). If there is no exact match for this key (because there is only a cache archive for `npm-cache-233ad571`), then it will look up any cache archive whose key starts with `npm-cache-`.

#### Separate caches for each OS and architecture

Cache is not guaranteed to work across different Bitrise Stacks (different OS or same OS but different CPU architecture). If a workflow runs on different stacks, it's a good idea to include the OS and architecture in the cache key:

```yaml
steps:
- restore-s3-cache@1:
    inputs:
    - key: |-
        {{ .OS }}-{{ .Arch }}-npm-cache-{{ checksum "package-lock.json" }}
        {{ .OS }}-{{ .Arch }}-npm-cache-
```


## ⚙️ Configuration

<details>
<summary>Inputs</summary>

| Key | Description | Flags | Default |
| --- | --- | --- | --- |
| `key` | Keys used for restoring a cache archive. One cache key per line in priority order.  The key supports template elements for creating dynamic cache keys. These dynamic keys change the final key value based on the build environment or files in the repo in order to create new cache archives. See the Step description for more details and examples.  The maximum length of a key is 512 characters (longer keys get truncated) and you can list at most 8 keys using this input. Commas (`,`) are not allowed in keys. | required |  |
| `verbose` | Enable logging additional information for troubleshooting. | required | `false` |
| `retries` | Number of retries to attempt when downloading a cache archive fails.  The value 0 means no retries are attempted. | required | `3` |
| `aws_bucket` | Bring your own bucket: exercise full control over the cache location.  The provided AWS bucket acts as cache backend for the Restore Cache step.  The step expects either: - CACHE_AWS_ACCESS_KEY_ID, CACHE_AWS_SECRET_ACCESS_KEY secrets to be setup for the workflow - The build is running on an EC2 instance. In this case, the steps expects the instance to have access to the bucket. | required |  |
| `aws_region` | AWS Region specifies the region where the bucket belongs. | required | `us-east-1` |
| `aws_access_key_id` | The access key id that matches the secret access key.  The credentials need to be from a user that has at least the following permissions in the bucket specified bellow `s3:ListObjects`, `s3:PutObject`, `s3:GetObjectAttributes` and `s3:GetObject`.  If the build instance has S3 access via IAM Instance role, this variable can be left empty.  | sensitive | `$CACHE_AWS_ACCESS_KEY_ID` |
| `aws_secret_access_key` | The secret access key that matches the secret key ID.  The credentials need to be from a user that has at least the following permissions in the bucket specified bellow `s3:ListObjects`, `s3:PutObject`, `s3:GetObjectAttributes` and `s3:GetObject`.  If the build instance has S3 access via IAM Instance role, this variable can be left empty.  | sensitive | `$CACHE_AWS_SECRET_ACCESS_KEY` |
</details>

<details>
<summary>Outputs</summary>

| Environment Variable | Description |
| --- | --- |
| `BITRISE_CACHE_HIT` | Indicates if a cache entry was restored. Possible values:  - `exact`: Exact cache hit for the first requested cache key - `partial`: Cache hit for a key other than the first - `false` No cache hit, nothing was restored |
</details>

## 🙋 Contributing

We welcome [pull requests](https://github.com/bitrise-steplib/bitrise-step-restore-s3-cache/pulls) and [issues](https://github.com/bitrise-steplib/bitrise-step-restore-s3-cache/issues) against this repository.

For pull requests, work on your changes in a forked repository and use the Bitrise CLI to [run step tests locally](https://devcenter.bitrise.io/bitrise-cli/run-your-first-build/).

**Note:** this step's end-to-end tests (defined in `e2e/bitrise.yml`) are working with secrets which are intentionally not stored in this repo. External contributors won't be able to run those tests. Don't worry, if you open a PR with your contribution, we will help with running tests and make sure that they pass.


Learn more about developing steps:

- [Create your own step](https://devcenter.bitrise.io/contributors/create-your-own-step/)
- [Testing your Step](https://devcenter.bitrise.io/contributors/testing-and-versioning-your-steps/)
