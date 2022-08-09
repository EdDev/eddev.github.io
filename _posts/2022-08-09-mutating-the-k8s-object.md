---
title: "Mutating the Kubernetes object"
date: 2022-08-09
categories:
  - blog
tags:
  - kubernetes
  - go
---

## Overview

[Kubernetes](https://kubernetes.io/) exposes API/s to modify its objects,
each with special behavior and characteristics. 

The existence of multiple methods that mutate an object is sometimes
confusing, potentially causing suboptimal usage.

We will start by covering the existing methods and continue focusing on
usages that fit them.

## Operations on Objects

The K8S API server supports several operations which can be used on
an object.

These operations are exposed at different layers of the stack, from the
low level HTTP, through the clientset and up to the application (e.g. kubectl).

The following diagram illustrates the relation.

![k8s-oper-on-object](/assets/images/k8s-oper-on-object.svg) 


### Mutable Operations

This section focuses on mutable operations.

> **_Note:_** The Create and Delete operations are excluded.

There are 3 mutable operations that can be performed on an object:
- Update
- Patch
- [Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/) 

> **_Note:_** `Apply` has also a client-side implementation, which uses
the server-side `Patch`. It is a pattern used by `kubectl` and can be
mimic by other application (e.g. controllers).

#### Update

The update operation replaces an existing object content with the new
content. In case some fields have not been included in the update,
they would be removed on the server-side.

The operation is using the opportunistic lock method using the `resourceVersion` field.

#### Patch

The patch operation has 3 variations:
- [JSON Patch](https://datatracker.ietf.org/doc/html/rfc6902/)
- [Merge Patch](https://datatracker.ietf.org/doc/html/rfc7386)
- [Strategic Merge Patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/#notes-on-the-strategic-merge-patch)

#### Server Side Apply

[SSA](https://kubernetes.io/docs/reference/using-api/server-side-apply/) is the most recent
method introduced by Kubernetes to modify and own specific fields in the object manifest.

## Usages

There are various scenarios and cases to modify an object.

The following list is probably incomplete, but still should provide
enough usage context to allow evaluating which method fits better.

- Client that needs to update multiple fields based on other fields content.
  - When the client is the owner.
  - When the client is not the owner.
- Multiple clients, that each own specific fields.
- Client/s need to update individual fields:
  - When the new values dependent on existing fields content.
  - When the new values do not depend on other fields.
- Client/s are interested in modifying the object in a declarative manner.

