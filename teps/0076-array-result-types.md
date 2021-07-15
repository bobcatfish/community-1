---
status: proposed
title: Array result types
creation-date: '2021-07-14'
last-updated: '2021-07-14'
authors:
- '@bobcatfish'
---

# TEP-0076: Expanded array support: results and indexing

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases](#use-cases)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Notes/Caveats](#notescaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
- [Test Plan](#test-plan)
- [Design Evaluation](#design-evaluation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Upgrade &amp; Migration Strategy](#upgrade--migration-strategy)
- [Implementation Pull request(s)](#implementation-pull-requests)
- [References](#references)
<!-- /toc -->

## Summary

This TEP proposes expanding support for arrays by adding:
* Support for indexing into arrays in variable replacement
  (in addition to [replacing entire lists](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#substituting-array-parameters))
* Support for array [results](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#emitting-results) (for both Pipelines and Tasks)

## Motivation

* See [use cases](#use-cases) - use cases for looping support in pipelines often need an array input;
  being able to produce an array from a Task to feed into a loop would enable some interesting CD use cases
* TEP-0075 proposing adding dictionary support in addition to the existing support strings
  and arrays as params, and results, but today we only support arrays as params and not as
  results. The proposal to add dictionary is mostly motivated by use cases for dictionary
  results (vs params) - but today we only allow string results. It seems confusingly
  inconsistent to imagine adding dictionary results but not array results (even though
  we support array params)
* Without indexing, arrays will only work with arrays. This means you cannot combine arrays with Tasks that
  only take string parameters and this limits the interoperabiltiy between Tasks.

### Goals

* Add two missing pieces of array support
  * As results
  * Indexing into arrays
* Take a step in the direction of allowing Tasks to have even more control
  over their params and results ([see pipelines#1393](https://github.com/tektoncd/pipeline/issues/1393))
* Be consistent in our treatment of arrays in Pipelines and Tasks and other types, e.g.
  dictionaries (TEP-0075)

### Non-Goals

* Adding complete [JSONPath](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html)
  syntax support
* Adding support for nesting, e.g. array params where the items in the array are themselves arrays
  (no reason we can't add that later but trying to keep it simple for now)

### Use Cases

1. Looping (i.e. providing values for an interface such as the task-loop custom task
   [iterateParam](https://github.com/tektoncd/experimental/tree/main/task-loops#specifying-the-iteration-parameter)
   (see the workaround added in [experimental#713](https://github.com/tektoncd/experimental/pull/713) which
   splits a string). For example, in tektoncd/plumbing we have a directory of
   [images](https://github.com/tektoncd/plumbing/tree/main/tekton/images). Each directory contains a Dockerfile -
   in a Pipeline that builds all images in this repo, we would want to: 1) determine which Dockerfiles need to be built
   and 2) for each of those Dockerfiles, build the image. Looping support would allow us to use a Task which builds
   just one image (e.g. [the kaniko task](https://github.com/tektoncd/catalog/tree/main/task/kaniko/0.4)), and run it
   for each Dockerfile, instead of having to the task to take multiple Dockerfiles as input. This missing piece of this
   is supporting the Task which determines with Dockerfiles to build and provides them as an array (instead of
   encoded in a string).

   ```
   filter-dockerfiles
        |
        | result: list of dockerfiles
        v
   (for each: filter-dockerfiles result)
      kaniko build
   ```

## Requirements

* Must be possible for a result array to be empty
* Must be possible to use the array results of one Task in a Pipeline as the
  param of another Task in the pipeline which has an array param
* Must be possible to use one specific value in an array result of one Task
  in a Pipeline as the param of another Task in the pipeline which has a string param
  * If there is no value at the specified index, the Pipeline execution should fail
    (or we may be able to use [TEP-0048](https://github.com/tektoncd/community/pull/240)
    to specify a default value to use in that case instead)

## Proposal

Currently we treat the contents of
[a Task's result file(s)](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#emitting-results)
as a string. The file is located at `$(results.resultName.path)` which expands to `/tekton/results/resultName`.
In this proposal, we add the option to write to a json file instead, available at `$(results.resultName.jsonPath)`
which expands to `/tekton/results/resultName.json`. The contents of this file would be treated as json and in addition
to supporting a string value in this file, we sould support json arrays.

This adds some additional complexity in that there are two paths to which you could write results (the string file
and the json file), but this option has these advantages:

* Completely backward compatible with current use of the results file
* Easy to specify string content which is itself json without needing to add additional logic to escape it
  (this is only the case for string results; not for values within an array which would still need to be escaped)

This feature would be considered alpha and would be
[gated by the alpha flag](https://github.com/tektoncd/pipeline/blob/main/docs/install.md#alpha-features).

### Notes/Caveats

* What if the file contains json we don't support? (e.g. we don't yet support dictionaries, and wouldn't
  initially support arrays of arrays)
    * The TaskRun (or Pipeline Task) would fail
* What if the type of the result content doesn't match the expected type? e.g. an array provided for a string result
  or vice versa?
    * The TaskRun (or Pipeline Task) would fail
* What if the Task writes to both the string file and the json file for the same param?
    * The TaskRun (or Pipeline Task) would fail

### Risks and Mitigations

* Changing the type of Task Result values from string to being ArrayOrString might be backwards incompatible
    * Mitigation: perform a manual test to make sure this isn't backwards incompatible (reconcile a TaskRun with a string
      result both before and after the change - e.g. create a TaskRun before the change, then restart the controller so
      it is re-reconciled; revisit this proposal if we can't find a way to make the change and be backward compatible)
    * Note this will be a change for folks consuming the library, e.g. the CLI, as a result implementing this TEP
      should include working with the CLI team to pull in the updates and make sure they are not negatively impacted

## Design Details

### Emitting array results

We add support for writing json content to `/tekton/results/resultName.json`, supporting strings and arrays of strings.

For example, say we want the value of the Task foo's result `animals` to be "cat", "dog" and "squirrel":

1. Write the following content to `$(results.animals.jsonPath)`:

   ```json
   ["cat", "dog", "squirrel"]
   ```

2. This would be written to the pod termination message as escaped json, for example (with a string example included as well):

   ```
   message: '[{"key":"animals","value":"[\"cat\", \"dog\", \"squirrel\"]","type":"TaskRunResult"},{"key":"someString","value":"aStringValue","type":"TaskRunResult"}]'
   ```

3. We would use the same
   [ArrayOrString type](https://github.com/tektoncd/pipeline/blob/1f5980f8c8a05b106687cfa3e5b3193c213cb66e/pkg/apis/pipeline/v1beta1/param_types.go#L89)
   (which in TEP-0075 could be expanded to support dictionaries as well) for
   task results, e.g. for the above example, the TaskRun would contain:

   ```yaml
     taskResults:
     - name: someString
       value: aStringValue
     - name: animals
       value:
       - cat
       - dog
       - squirrel
   ```

### Indexing into arrays with variable replacement

In addition to supporting [JSONPath](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html)
style [star notation](https://github.com/tektoncd/pipeline/issues/2041), we add support for
[the JSONPath subscript operator](https://tools.ietf.org/id/draft-goessner-dispatch-jsonpath-00.html#name-jsonpath-examples)
for accessing "child" members of the array by index.

For example:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
...
spec:
  params:
    - environments:
        - 'staging'
        - 'qa'
        - 'prod'
...
  tasks:
    - name: deploy
      params:
        - name: environment
          value: '$(params.environments[0])'
```

The resulting taskRun would contain:

```yaml
  params:
    - name: environment
      value: 'staging'
```

This proposal does not include adding support for any additional syntax, e.g. slicing (though it could be added in the
future!)

### Using array results with variable replacement

The same syntax [above](#indexing-into-arrays-with-variable-replacement) could be used to access a specific value in
the results of a previous Task.

For example:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
...
spec:
tasks:
  - name: get-environments
  - name: deploy
    params:
      - name: environment
        value: '$(tasks.get-environments.environments[0])'
```

## Test Plan

In addition to unit tests the implementation would include:

* At least one reconciler level integration test which includes an array result
* A tested example of a Task that produces an array result with:
  * Another Task consuming that array
  * Another Task consuming one specific value from that array

## Design Evaluation

* [Reusability](https://github.com/tektoncd/community/blob/main/design-principles.md#reusability):
  * Pro: This will improve the reusability of Tekton components by enabling the scenario of a Task providing an array result
    and then a Pipeline being able to loop over those value for subsequent Tasks (i.e. reusing Tasks which are designed
    for single inputs, but users want to use 'for each' item in an array)
* [Simplicity](https://github.com/tektoncd/community/blob/main/design-principles.md#simplicity)
  * Pro: This proposal reuses the existing array or string concept for params
  * Pro: This proposal continues [the precedent of using JSONPath syntax in variable replacement](https://github.com/tektoncd/pipeline/issues/1393#issuecomment-561476075)
  * Con: There are now two different files/paths to which users can write results and they have to understand when
    to use which. (We could choose to deprecate the original path if necessary.)
* [Flexibility](https://github.com/tektoncd/community/blob/main/design-principles.md#flexibility)
  * Con: Although there is a precedent for including JSONPath syntax, this is a step toward including more hard coded
    expression syntax in the Pipelines API (without the ability to choose other language options)
* [Conformance](https://github.com/tektoncd/community/blob/main/design-principles.md#conformance)
  * Supporting array results and indexing syntax would be included in the conformance surface

## Drawbacks

* Increases the size of TaskRuns (by including arrays in termination messages and results)
* Only partial support for JSONPath (i.e. just array indexing support, nothing more)

## Alternatives

1. Do not support arrays explicitly - if we do this, we probably shouldn't do TEP-0075 either, i.e. results should
  always just be strings
1. Add support for array results, but don't add support for indexing into arrays
   * Con: In TEP-0075 we propose adding support for dictionaries; if we take the same approach as this proposal, this
     would mean not allowing the ability to get individual values out of those dictionaries which would significantly
     limit the number of Tasks this would be compatible with (i.e. we should be consistent between arrays and
     dictionaries)
   * Con: Without indexing, arrays will only work with arrays. This means you cannot combine arrays with Tasks that
     only take string parameters and this limits the interoperabiltiy between Tasks.
1. Instead of adding support for a `.json` file in addition to the existing path, we use the same file and parse it as
  json.
    * Pro: Avoids the complication of two different paths for result files
    * Con: A backwards incompatible change for Tasks which are currently writing json to the results file and treating
      it as a string. The feature would be behind an alpha flag so they wouldn't be impacted immediately but eventually
      they would need to update their Tasks to escape the json.
1. Instead of failing PipelineTasks which try to refer to values in Arrays that don't exist (e.g. empty arrays or going
  beyond the bounds of an array), we could skip them
  * Con: If this was done unintentionally, the user might not notice the skipped task
1. Adopt a different syntax
  * Con: would likely need to rethink our existing [star notation](https://github.com/tektoncd/pipeline/issues/2041)
    support as well
  * Examples of other options:
    * [JSON pointer](https://datatracker.ietf.org/doc/html/rfc6901) - this syntax is much smaller in scope than JSONPath
      which is appealing, however it doesn't support iteration that we may want to adopt such as array slicing
    * [CEL](https://github.com/google/cel-spec/blob/master/doc/langdef.md) - this syntax is much larger in scope
      than JSONPath which could potentially allow expressing more; see
      [pipelines#2812](https://github.com/tektoncd/pipeline/issues/2812) for more on adding an entire expression
      language
  * Choosing JSONPath as our preferred syntax has the advantage of supporting many use cases around accessing values
    within data structures for variable replacmeent without expanding the Tekton API to include an entire expression
    language.
1. Rely only on custom tasks for accessing values inside params (e.g.
  [this example of the CEL custom task](https://github.com/tektoncd/pipeline/issues/3255#issuecomment-765535707))

## Upgrade & Migration Strategy

By adding an additional file path, we make this a completely backwards compatible change.

## Implementation Pull request(s)

<!--
Once the TEP is ready to be marked as implemented, list down all the Github
Pull-request(s) merged.
Note: This section is exclusively for merged pull requests, for this TEP.
It will be a quick reference for those looking for implementation of this TEP.
-->

TBD

## References

* [pipelines#1393 Consider removing type from params (or _really_ support types)](https://github.com/tektoncd/pipeline/issues/1393)
* [pipelines#3255 Arguments of type array cannot be access via index](https://github.com/tektoncd/pipeline/issues/3255)