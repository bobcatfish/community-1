# User Profiles

The purpose of this document is to aid in the development of Tekton features by evaluating Tekton features against those who will be using them.

When considering requirements and implementation solutions it's useful look at who uses Tekton to have a better understanding of how something should work. The Tekton user profiles describe the different types of users and contributors to Tekton along with the order of priority they have relative to each other.

## Preface

What's described here are user profiles as opposed to
[personas](https://en.wikipedia.org/wiki/Persona#In_user_experience_design).
Personas are example actors rather than general categories. A single persona
can potentially match with multiple user profiles at the same time.

## Profiles

Profiles describe a type of role a user may perform. A real person may perform more than one
role and have more than one profile apply to them. How this mapping works between profiles
and real people can vary between companies and other organizations. To handle this variation
we focus on the user profiles rather than how they may map to people in these different
organizations.

### Platform Team Member

Folks on platform teams are trying to create CI/CD solutions that apply across multiple teams
at their company. They are often responsible for building and choosing tools to do this. They
may also be responsible for defining standardized `Tasks` and `Pipelines`, and enforcing
polices related to their execution (e.g. permissions, resource usage limits, code quality
requirements).

### Pipeline executor

Pipeline executor are the folks who are actually executing Pipelines and observing their
progress. They may be executing them manually or they may be creating triggering events, such
as opening pull requests.

### Task Author

Authors are people who package `Tasks` and `Pipelines` for someone else to [execute]

Examples of this would be those who maintain [Tekton's Catalog](https://github.com/tektoncd/catalog) for Kaniko and ArgoCD.

### 3. Platform Builder

Platform builders are people who wrap Tekton's core engine and libraries to extend it. An example of this is the platform being built by [Jenkins X](https://github.com/jenkins-x/jx) using Tekton pipelines.

### 4. Application Developer

An application developer writes the software for an application. An application developer is focused on releasing product updates with high quality as quickly as possible. Examples of this include the developers of WordPress and MySQL.

### 5. Supporting Tool Developer

Supporting tool developers build tools adjacent to Tekton, such as plugins, pipeline visualizations, pipeline debugging tools, Tekton's tkn command line tool, or even kubectl. These are developers building complementary things that can be used along with Tekton.

### 6. Tekton Developer

Tekton developers are those who develop Tekton itself. That includes core maintainers along with anyone else who fixes a bug or updates docs.

Generally speaking, the developers of Tekton and its interfaces consider the end users above themselves when looking at requirements and implementation strategies.

## Profiles Not In Scope

Some user profiles are not considered in scope for Tekton. That does not mean a real person who multiple profiles apply to is not considered a supported user. Rather, the out of scope profiles apply to roles that are not typically supported.

### Cluster Operator

A cluster operator stands up and operates a Kubernetes cluster. This includes elements such as the control plane, nodes, and elements in the stack below these. It does not include pipelines running on Kubernetes as those are handled by the _Pipeline Operator_ profile.

### Application Operator

Application operators take an application and operate it within a Kubernetes cluster. For example, the operation of WordPress and MySQL. These activities are irrelevant to Tekton unless the activity naturally intersects with Tekton as in the _Pipeline Operator_ profile.

