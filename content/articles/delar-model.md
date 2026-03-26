---
title: The DELAR Model
date: 2025-02-21
params:
  author: Joseph Wright
---

## Background

The DELAR model is something I developed after years of doing IaC with various tools and finding myself reusing the same patterns. I wanted to explain how it works in a way that didn't involve digging through code. Since everything was patterned in layers, I thought what if I had a diagram similar to the OSI model for networking, but for IaC? I could use that as the starting point and it would give me a vocabulary to discuss it more easily.

I originally had some other names for it, one of them being Infrastructure as Data. A friend coined the term DELAR model after I presented it to his team at work. DELAR is an acronym for the layers of the model: Data, Execution, Library, API, and Resource.

![DELAR Model](/images/delar-model.png)

Why are the layers in that order? It's the order that data flows through the stack. Data at the top represents the resources that are created at the bottom. The data passes through all the layers, in one direction, to achieve that.

## Data Layer

The data layer contains your configuration and is typically where you run your IaC tool. For Terraform, this is the [root module](https://developer.hashicorp.com/terraform/language/modules#hierarchy).

The data layer is not intended to be reusable. It represents actual resources, and, per the model, a change to the data should trigger a change to the resources. If the data were reused, multiple "stacks" or environments also would change simultaneously. Yes, it is possible to have a data layer with separate per-environment variable files and I have done so before, however I prefer the stricter method with no reuse allowed. Tools like [terragrunt](https://terragrunt.com) encourage making entire root modules reusable. I don't agree with that approach at all.

What goes into the data layer:

* Variables: all of the configuration, including environment-specific data, AWS account information, etc.
* Dependencies and their versions: all dependencies must be specified and are always pinned to the exact versions used.
* Tool version: the exact version of the tool to be used, if supported by the tool.

What doesn't go here:

* Bare Terraform resources
* Arbitrary scripts

## Execution Layer

The execution layer is the thinnest of all the layers. In its simplest form, it is you running `terraform apply`. In its more complex forms, it is a CI/CD pipeline that also runs, well, `terraform apply`. But in the CI/CD pipeline, it will typically have more stages to it. For example, you push code to the git repo and open a PR. The first stage of the pipeline triggers and runs `terraform plan` and `opa eval`. If it succeeds, you are allowed to merge the PR. Then the second stage runs `terraform apply`.

The execution layer should never be *very* complex though. It should work the same way no matter what code you push. And it shouldn't do anything you can't do yourself to test and verify.

When developing IaC, you need a sandbox environment in which to work. In that environment, *you* want to be the executor, not CI/CD. This way you can test that everything is working before it gets pushed to the repo. This is much faster than pushing to a repo, opening a PR, getting approvals, and praying it works. Once you've done it yourself and gone through the iterations, you can push to the repo. And given that your CI/CD pipeline runs the same commands you have been, it stands a much higher chance of working on the first try.

If you modify the execution layer, it should be because you need to make a change common to all of your infrastructure pipelines. For example, your CI/CD might pass some common variables to all stacks, such as a secret used to enable the deployment. Maybe you need to add another such common variable. In that case, go ahead and make the change. Don't change the execution layer to add workarounds for a special snowflake environment that works a little differently from everything else. This is when it starts to break down. I have never seen a case where such workarounds could not be done better in one of the other layers, usually the library layer.

## Library Layer

The library layer is the meat of the IaC stack.

## API Layer

## Resource Layer

An example of the model using Terraform:

**Data** - This is the Terraform "root" module. What most people think of when they write Terraform. I'm just a little stricter about what goes in here.

**Execution** - The execution of the tool. This can be you running `terraform apply` or your CI/CD process doing it. It should work the same either way, so you can iterate in a sandbox without pushing to a remote git server.

**Library** - Reusable, versioned Terraform modules. Everything is parameterized, without references to your environments; all of that lives in the data layer. Most of your development time should be spent here.

**API** - The APIs called by your modules. This can be AWS, Databricks, GCP, or even your own APIs. If you write a custom Terraform provider, you will be working in this layer.

**Resource** - The end result: actual cloud resources. The data layer is the code representation for what lives here.
