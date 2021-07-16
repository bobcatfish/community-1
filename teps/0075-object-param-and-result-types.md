---
status: proposed
title: Object/Dictionary param and result types
creation-date: '2021-07-14'
last-updated: '2021-07-14'
authors:
- '@bobcatfish'
- '@skaegi' # I didn't check with @skaegi before adding his name here, but I wanted to credit him b/c he proposed most of this in https://github.com/tektoncd/pipeline/issues/1393 1.5 years ago XD
---

# TEP-0075: Object/Dictionary param and result types


<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases](#use-cases)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Notes/Caveats](#notescaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [User Experience](#user-experience)
- [Design Details](#design-details)
- [Test Plan](#test-plan)
- [Design Evaluation](#design-evaluation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Upgrade &amp; Migration Strategy](#upgrade--migration-strategy)
- [Implementation Pull request(s)](#implementation-pull-request-s)
- [References](#references)
<!-- /toc -->

## Summary

_Recommendation: read
[TEP-0076 (array support in results and indexing syntax)](https://github.com/tektoncd/community/pull/477)
before this TEP, as this TEP builds on that one._

This TEP proposes adding dictionary (aka object) types to Tekton Task and Pipeline
[results](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#emitting-results)
and [params](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#specifying-parameters),
as well as adopting a small subset of [JSONPath](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html)
syntax (precedent in
[the array expansion syntax we are using](jhttps://github.com/tektoncd/pipeline/issues/1393#issuecomment-561476075)) 
for accessing values in these dictionaries.

This proposal is dependent on
[TEP-0076 (array support in results and indexing syntax)](https://github.com/tektoncd/community/pull/477)
in that this TEP should follow the precedents set there and also in that if we decide not to support array results, we
probably won't support dictionary results either.

This proposal also includes adding support for limited use of
[json object schema](https://json-schema.org/understanding-json-schema/reference/object.html),
to express the expected structure of the dictionary.

_Dictionary vs. object: The intended feature was supporting "dictionaries", but json schema calls these "objects" so
this proposal tries to use "objects" as the term for this type._

## Motivation

Tasks declare workspaces, params, and results, and these can be linked in a Pipeline, but external tools looking at
these Pipelines cannot reliably tell when images are being built, or git repos are being used. Current ways around this
are:
* [PipelineResources](https://github.com/tektoncd/pipeline/blob/main/docs/resources.md) (proposed to be
  deprecated in TEP-0074)
* Defining param names with special meaning, for example
  [Tekton Chains type hinting](https://github.com/tektoncd/chains/blob/main/docs/config.md#chains-type-hinting))

This proposal takes the param name type hinting a step further by introducing object types for results: this will
allow Tasks to define structured results. These structured results can be used to define interfaces, for example the
values that are expected to be produced by Tasks which do `git clone`. This is possible with just string results, but
without a grouping mechanisms, all results and params are declared at the same level and it is not obvious which results
are complying with a general interface and which are specific to the task.

### Goals

* Tidy up Task interfaces by allowing Tasks to group related parameters (similar to the
  [long parameter list](https://www.arhohuttunen.com/long-parameter-list/) or
  [too many parameters](https://dev.to/thinkster/code-smell-too-many-parameters-435e) "code smell")
* Enabled defining well known structured interfaces for Tasks (typing), for example, defining the values that a Task should
  produce when it builds an image, so that other tools can interface with them (e.g.
  [Tekton Chains type hinting](https://github.com/tektoncd/chains/blob/main/docs/config.md#chains-type-hinting))
* Take a step in the direction of allowing Tasks to have even more control
  over their params and results ([see pipelines#1393](https://github.com/tektoncd/pipeline/issues/1393))

### Non-Goals

* Adding complete [JSONPath](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html)
  syntax support
* Adding support for nesting, e.g. object params where the values are themselves objects or arrays
  (no reason we can't add that later but trying to keep it simple for now)
* Adding complete [JSON schema](https://json-schema.org/) syntax support 
* Supporting use of the entire object in a Task or Pipeline field, i.e. as the values for a field that requires a
  dictionary, the way we do
  [for arrays](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#substituting-array-parameters). Keeping
  this out of scope because we would need to explictly decide what fields we want to support this for. We'll likely
  add it later; at that point we'd want to support the same syntaxes we use
  [for arrays](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#substituting-array-parameters) (including
  `[*]`)

### Use Cases (optional)

1. Grouping related params and results such that users and tools can make inferences about the tasks. For example
   allowing [Tekton Chains](https://github.com/tektoncd/chains/blob/main/docs/config.md#chains-type-hinting) and
   other tools to be able to understand what types of artifacts a Pipeline is operating on.
   
   Tekton could also define known interfaces, to make it easier to build these tools. For example:
     * Images - Information such as the URL and digest
     * Wheels and other package formats (see below)
   
   For example [the upload-pypi Task](https://github.com/tektoncd/catalog/tree/main/task/upload-pypi/0.1)
   defines 4 results which are meant to express the attributes of the built
   [wheel](https://www.python.org/dev/peps/pep-0427/):
   
   ```yaml
     results:
     - name: sdist_sha
       description: sha256 (and filename) of the sdist package
     - name: bdist_sha
       description: sha256 (and filename) of the bdist package
     - name: package_name
       description: name of the uploaded package
     - name: package_version
       description: version of the uploaded package
   ```
   
   Note some interesting things are happening above already, for example sdist_shaw contains both the filename and the
   sha - an attempt to group related information.
   
   With object results, we could define the above with a bit more structure like this:
   
   ```yaml
     results:
       - name: sdist
         description: |
           The source distribution
            * sha: The sha256 of the contents
            * path: Path to the resulting tar.gz
         schema:
           type: object
           properties:
             sha:
             path: {}
       - name: bdist
         description: |
           The built distribution
            * sha: The sha256 of the contents
            * path: Path to the resulting .whl file
         schema:
           type: object
           properties:
             sha: {}
             path: {}
       - name: package
         description: |
           Details about the created package
            * name: The name of the package
            * version: The version of the package
         schema:
           type: object
           properties:
             name: {}
             version: {}
   ```
   
   Eventually when we have nested support, we could define a `wheel` interface which contains all of the above.

2. Grouping related params to create simpler interfaces, and allowing similar Tasks to easily pieces of their interface.
   For example [the git-clone task has 15 parameters](https://github.com/tektoncd/catalog/blob/main/task/git-clone/0.4/README.md#parameters),
   and [the git-rebase task has 10 parameters](https://github.com/tektoncd/catalog/blob/main/task/git-rebase/0.1/README.md#parameters).
   Some observations:
   * Each has potential groupings which stand out, for example:
     * The git-rebase task could group the `PULL` and `PUSH` remote params; each object would need the same params,
       which could become an interface for "git remotes"
     * Potential groupings for the git-clone task stand out as well, for example the proxy configuration
     * On that note, since the git-rebase task is using git and accessing remote repos, it probably needs the same
       proxy configuration, so what they probably both need is some kind of interface for what values to provide when
       accessing a remote git repo
   Other examples include [okra-deploy](https://github.com/tektoncd/catalog/tree/main/task/orka-deploy/0.1) which is using
   param name prefixes to group related params (e.g. `ssh-`, `okra-vm`, `okra-token`).

## Requirements

* Must be possible to programmatically determine the structure of the object
* Must be possible for a result object to be empty
* Must be possible to use the object results of one Task in a Pipeline as the
  param of another Task in the pipeline which has a object param
* Must be possible to use one specific value in a object result of one Task
  in a Pipeline as the param of another Task in the pipeline which has a string param
  * If there is no value for the specified key, the Pipeline execution should fail
    (or we may be able to use [TEP-0048](https://github.com/tektoncd/community/pull/240)
    to specify a default value to use in that case instead)

## Proposal

* We would add support for object types for results and params, in addition to the existing string and array support
  * Initially we would only support string keys, eventually we can expand this to all values (string, array, object)
    (note that this is the case for arrays as well, we don't yet support arrays of arrays)
* We would use [json object schema syntax](https://github.com/tektoncd/pipeline/issues/1393#issuecomment-558833526)
  to express the object structure
* To support object results, we would support writing results to a `.json` file, as described in
  [TEP-0076 Array results and indexing](https://github.com/tektoncd/community/pull/477)

This feature would be considered alpha and would be
[gated by the alpha flag](https://github.com/tektoncd/pipeline/blob/main/docs/install.md#alpha-features).

### Notes/Caveats

[See also TEP-0076 Array results and indexing](https://github.com/tektoncd/community/pull/477) for notes and caveats
specific to supporting json results.

* What if a Pipeline tries to use a object key that doesn't exist in a param or result?
  * Some of this could be caught at creation time, i.e. pipeline params would declare the keys each object param will
    contain, and we could validate at creation time that those keys exist
  * For invalid uses that can only be caught at runtime, the PipelineRun would fail
* What if a Task tries to write more keys to a result than the result declares it contains?
  * The TaskRun would fail (alternatively we could decide to ignore the extra keys)
* What if a Task tries to write less keys to a result than the result declares it contains?
  * The TaskRun would fail

### Risks and Mitigations

[See also TEP-0076 Array results and indexing](https://github.com/tektoncd/community/pull/477) for risks and mitigations
specific to expanding ArrayOrString to support more types.

### User Experience

* Using the json schema syntax to describe the structure of the object is a bit verbose and might not be intuitive
  to users; see [Alternatives](#alternatives) for discussion of an option where we create our own syntax.

## Design Details

### Declaring object results and params

We would use [JSON object schema syntax](https://json-schema.org/understanding-json-schema/reference/object.html) to
declare the structure of the object.

Params example:

```yaml
  params:
    - name: pull_remote
      description: json schema has no "description" fields, so we'd have to include documentation about the structure in this field
      schema:
        type: object # see question below
        properties:
          url: {
            type: string # for now, all values are strings, so we could imply it
          }
          path: {} # example of implying type: string
```

Results example:

```yaml
  results:
    - name: sdist
      description: json schema has no "description" fields, so we'd have to include documentation about the structure in this field
      schema:
        type: object
        properties:
          sha: {}
          path: {}
```

**Question**: Do we want to keep using [the existing `type` field](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#specifying-parameters)?
i.e. do we use the top level `type: object` syntax, similar to how we declare `type: string` and `type: array` today, or
do we want to move to always using `schema` and using the same syntax for arrays and strings as well (as suggested in
[pipelines#1393](https://github.com/tektoncd/pipeline/issues/1393#issuecomment-558833526))?

**Proposal**: if possible (can find out with a proof of concept), we deprecated `type:`. We would still default to string
when `schema` is not present, but when [array] or dict/object types are used, we use `schema`.

### Emitting object results

As described in
[TEP-0076 Array results and indexing](https://github.com/tektoncd/community/pull/477),
we add support for writing json content to `/tekton/results/resultName.json`, supporting strings, objects and arrays
of strings.

For example, say we want to write to emit a built image's url and digest in a dictionary called `image`:

1. Write the following content to `$(results.image.jsonPath)`:

   ```json
   {"url": "gcr.io/somerepo/someimage", "digest": "a61ed0bca213081b64be94c5e1b402ea58bc549f457c2682a86704dd55231e09"}
   ```

2. This would be written to the pod termination message as escaped json, for example (with a string example included as well):

   ```
   message: '[{"key":"image","value":"{\"url\": \"gcr.io\/somerepo\/someimage\", \"digest\": \"a61ed0bca213081b64be94c5e1b402ea58bc549f457c2682a86704dd55231e09\"}","type":"TaskRunResult"},{"key":"someString","value":"aStringValue","type":"TaskRunResult"}]'
   ```

3. We would use the same
   [ArrayOrString type](https://github.com/tektoncd/pipeline/blob/1f5980f8c8a05b106687cfa3e5b3193c213cb66e/pkg/apis/pipeline/v1beta1/param_types.go#L89)
   (expanded to support dictionaries, and in TEP-0074, arrays) for
   task results, e.g. for the above example, the TaskRun would contain:

   ```yaml
     taskResults:
     - name: someString
       value: aStringValue
     - name: image
       value:
         url: gcr.io/somerepo/someimage
         digest: a61ed0bca213081b64be94c5e1b402ea58bc549f457c2682a86704dd55231e09
   ```

### Using objects in variable replacement

We add support for
[the JSONPath subscript operator](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html#name-jsonpath-examples)
for accessing "child" members of the object by name in variable replacement.

For example:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
...
spec:
  params:
    - name: gitrepo
      value:
        url: github.com/tektoncd/pipeline
        commitish: v0.23.0
...
  tasks:
    - name: notify-slack
      params:
        - name: message
          value: "about to clone $(params.gitrepo.url) at $(params.gitrepo.commitish)"
```

This proposal does not include adding support for any additional syntax (though it could be added in the future!)

## Test Plan

In addition to unit tests the implementation would include:

* At least one reconciler level integration test which includes an object param and an object result
* A tested example of a Task, used in a Pipeline, which:
    * Declares an object param
    * Emits an object result
    * In the Pipeline, another Task consumes a specific value from the object

## Design Evaluation

[See also the TEP-0076 Array results and indexing](https://github.com/tektoncd/community/pull/477) design evaluation.

* [Reusability](https://github.com/tektoncd/community/blob/main/design-principles.md#reusability):
    * Pro: PipelineResources provided a way to make it clear what artifacts a Pipeline/Task is acting on, but
      [using them made a Task less resuable](https://github.com/tektoncd/pipeline/blob/main/docs/resources.md#why-arent-pipelineresources-in-beta) -
      this proposal introduces an alternative that does not involve making a Task less reusable (especially
      once coupled with [TEP-0044](https://github.com/tektoncd/community/blob/main/teps/0044-decouple-task-composition-from-scheduling.md))
* [Simplicity](https://github.com/tektoncd/community/blob/main/design-principles.md#simplicity)
    * Pro: Support for emitting object results builds on [TEP-0076](https://github.com/tektoncd/community/pull/477)
    * Pro: This proposal reuses the existing array or string concept for params
    * Pro: This proposal continues [the precedent of using JSONPath syntax in variable replacement](https://github.com/tektoncd/pipeline/issues/1393#issuecomment-561476075)
* [Flexibility](https://github.com/tektoncd/community/blob/main/design-principles.md#flexibility)
    * Pro: Improves support for structured interfaces which tools can rely on
    * Con: Although there is a precedent for including JSONPath syntax, this is a step toward including more hard coded
      expression syntax in the Pipelines API (without the ability to choose other language options)
    * Con: We're also introducing and committing to json schema! Seems worth it for what we get tho
* [Conformance](https://github.com/tektoncd/community/blob/main/design-principles.md#conformance)
    * Supporting this syntax would be part of the conformance surface; the json schema syntax is a bit verbose for
      simple cases (but paves hte way for the more complex cases we can support later, including letting Tasks
      express how to validate their own parameters)

## Drawbacks

[See also TEP-0076 Array results and indexing](https://github.com/tektoncd/community/pull/477) for more drawbacks.

* In the current form, this sets a precedent for pulling in more json schema support (or whatever format we choose
  for expressing the schema)
* This improves the ability to make assertions about Task interfaces but doesn't (explicitly) include worksapces at all.
  For example, say you had a “git” param for a pipeline containing a commitish and a url. This is likely going to be
  used to clone from git into a workspace, and there will be some guarantees that can be made about the data that ends
  up in the workspace (e.g. the presence of a .git directory). But there is no way of expressing the link between the
  workspace and the params.

## Alternatives

1. Add support for nested types right away
1. Add complete json schema support
1. Use something other than json schema for expressing the object/dictionary structure
   * We could make our own syntax, specific to dictionaries, however if we later add more json schema support we'd
     need to revisit this. For example:
     ```yaml
        results:
          - name: sdist
            description: json schema has no "description" fields, so we'd have to include documentation about the structure here
            type: dict
            keys: ['sha', 'path']
     ```
     This is clean and clear when we're talking about a dictionary of just strings, but what if we want to allow nesting,
     e.g. array or dict values?
     ```yaml
        results:
          - name: sdist
            description: json schema has no "description" fields, so we'd have to include documentation about the structure here
            type: dict
            structure:
               details:
                 type: dict
                 structure: # what if this dictionary has dictionary values, and so on
     ```
     It feels at this point like it's better to adopt a syntax that already handles these cases.
1. Reduce the scope, e.g.:
    * Don't add support for grabbing individual items in a dictionary
        * We need this eventually for the future to be useful
    * Don't let dictionaries declare their keys (e.g. avoid bringing in json schema)
        * Without a programmatic way to know what keys a dictionary needs/provides, we actually take a step backward
          in clarity, i.e. a list of strings has more information than a dictionary where you don't know the keys

## Upgrade & Migration Strategy

[As mentioned in TEP-0076 Array results and indexing](https://github.com/tektoncd/community/pull/477) this update will
be completely backwards compatible; the biggest change is the addition of a `.json` results file, which won't impact
existing results use.

## Implementation Pull request(s)

TBD.

## References

- [pipelines#1393 Consider removing type from params (or _really_ support types) ](https://github.com/tektoncd/pipeline/issues/1393)
