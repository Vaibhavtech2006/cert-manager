# Design: Per-Certificate Secret Owner Reference

> 🌟 This design document was written by Maël Valais on 20 July 2022 in order to facilitate Denis Romanenko's feature request presented in [#5158](https://github.com/cert-manager/cert-manager/pull/5158).

- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Use-cases](#use-cases)
- [Questions](#questions)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Supported Versions](#supported-versions)
- [Production Readiness](#production-readiness)
  - [How can this feature be enabled / disabled for an existing cert-manager installation?](#how-can-this-feature-be-enabled--disabled-for-an-existing-cert-manager-installation)
  - [Does this feature depend on any specific services running in the cluster?](#does-this-feature-depend-on-any-specific-services-running-in-the-cluster)
  - [Will enabling / using this feature result in new API calls (i.e to Kubernetes apiserver or external services)?](#will-enabling--using-this-feature-result-in-new-api-calls-ie-to-kubernetes-apiserver-or-external-services)
  - [Will enabling / using this feature result in increasing size or count of the existing API objects?](#will-enabling--using-this-feature-result-in-increasing-size-or-count-of-the-existing-api-objects)
  - [Will enabling / using this feature result in significant increase of resource usage? (CPU, RAM...)](#will-enabling--using-this-feature-result-in-significant-increase-of-resource-usage-cpu-ram)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

This checklist contains actions which must be completed before a PR implementing this design can be merged.

- [ ] This design doc has been discussed and approved
- [ ] Test plan has been agreed upon and the tests implemented
- [ ] Feature gate status has been agreed upon (whether the new functionality will be placed behind a feature gate or not)
- [ ] Graduation criteria is in place if required (if the new functionality is placed behind a feature gate, how will it graduate between stages)
- [ ] User-facing documentation has been PR-ed against the release branch in [cert-manager/website]

## Summary

cert-manager has the ability to set the owner reference field in generated Secret resources.
The option is global, and takes the form of the flag `--enable-certificate-owner-ref` set in
the cert-manager controller Deployment resource.

Let us take an example Certificate resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cert-1
  namespace: ns-1
  uid: 1e0adf8
spec:
  secretName: cert-1
```

When `--enable-certificate-owner-ref` is passed to the cert-manager controller, cert-manager,
when issuing the X.509 certificate, will create a Secret resource that looks like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cert-1
  namespace: ns-1
  ownerReferences:
    - controller: true
      blockOwnerDeletion: false
      uid: 1e0adf8
      name: cert-1
      kind: Certificate
      apiVersion: cert-manager.io/v1
data:
  tls.crt: "..."
  tls.key: "..."
  ca.crt: "..."
```

The proposition is to add a new field `cleanupPolicy` to the Certificate resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  secretName: cert-1
  cleanupPolicy: [OnDelete|Never] # ✨ Can be left empty.
```

The new field `cleanupPolicy` has three possible values:

1. When not set, the value set by `--default-secret-cleanup-policy` is inherited.
2. When `OnDelete`, the owner reference is always created on the Secret resource.
3. When `Never`, the owner reference is never created on the Secret resource.

> At first, the proposed field was named `certificateOwnerRef` and was a
> nullable boolean. James Munnelly reminded us that the Kubernetes API
> never uses boolean fields, and instead uses the string type with
> "meaningful values". On top of being more readable, it also makes the
> field extensible.

When changing the value of the field `cleanupPolicy` from `OnDelete` to `Never`,
the associated Secret resource immediately loses its owner reference. The user
doesn't need to wait until the certificate is renewed. Similarly, when `cleanupPolicy`
is changed from `OnDelete` to `Never`, the associated Secret resource loses its
owner reference.

Along with this new field, we propose to deprecate the flag `--enable-certificate-owner-ref`
and introduce the new flag `--default-secret-cleanup-policy`. Its values are as follows:

- When `--default-secret-cleanup-policy` is set to `Never`, the Certificate resources
  that don't have the `cleanupPolicy` field set will have their associated Secret
  resources updated (i.e., the owner reference gets removed) on the next issuance of
  the Certificate.
- When `--default-secret-cleanup-policy` is set to `OnDelete`, the Certificate resources
  that don't have the `cleanupPolicy` field set will have their associated Secret
  resources updated (i.e., the owner reference gets added) on the next issuance of
  the Certificate.
  
The effect of changing `--default-secret-cleanup-policy` from `Never` to `OnDelete`
or from `OnDelete` to `Never` is not immediate: the change requires a re-issuance
of the Certificate resources.

The default value for `--default-secret-cleanup-policy` is `Never`.

When changing the flag from `Never` to `OnDelete`, the existing Certificate resources
that don't have `cleanupPolicy` set are immediately affected, meaning that their
associated Secrets will gain a new owner reference. When changing the flag from
`OnDelete` to `Never`, the Secrets associated to Certificates that have no `cleanupPolicy`
set will see their owner reference immediately removed.

The reason we decided to deprecate `--enable-certificate-owner-ref` is because this
flat behaves differently to how the new `cleanupPolicy` behaves:

- When `--enable-certificate-owner-ref` is not passed (or is set to false), the existing
  Secret resources that have an owner reference are not changed even after a re-issuance.
  With `--default-secret-cleanup-policy` and given that `cleanupPolicy` is not set, the
  behavior is slightly different: unlike with the old flag, the existing Secret resources
  will have their owner references removed.
- When `--enable-certificate-owner-ref` is set to true, the behavior is the same as
  when `--default-secret-cleanup-policy` is set to `OnDelete` and `cleanupPolicy` is not
  set.  

The deprecated flag `--enable-certificate-owner-ref` keeps precendence over the new flag
in order to keep backwards compatibility.

When upgrading to the new flag, users can refer to the following table:

| If... | then they should replace it with... |
|-----|-----|
| `--enable-certificate-owner-ref` not passed to the controller | No change needed |
| `--enable-certificate-owner-ref=false` | Replace with `--default-secret-cleanup-policy=Never` |
| `--enable-certificate-owner-ref=true` | Replace with `--default-secret-cleanup-policy=OnDelete` |

## Use-cases

[Flant](https://flant.com) manages certificates for users, and has hit a Kubernetes apiserver limitation where too many leftover Secret resources were slowing the apiserver down. This issue has happened because Certificate resources are created using auto-generated names, and Certificate resources are often deleted shortly after being created.

## Questions

<!--
This section is important for producing high-quality, user-focused
documentation such as release notes.

A good summary is probably around a paragraph in length.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
the proposed enhancement.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to
demonstrate the interest in this functionality amongst the community.
-->

### Goals

<!--
List specific goals. What is this proposal trying to achieve? How will we
know that this has succeeded?
-->

### Non-Goals

<!--
What is out of scope for this proposal? Listing non-goals helps to focus discussion
and make progress.
-->

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
What is the desired outcome and how do we measure success?
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation- those should go into "Design Details" below.
-->

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this proposal gets implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

#### Story 2

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go into as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes/PKI ecosystem.

-->

## Design Details

cert-manager would have to change in a few places.

**Mutating webhook**

We propose to have no "value defaulting" for `cleanupPolicy` because the
"empty" value has a meaning for us: when `cleanupPolicy` is empty, the
presence or not of the flag `--enable-certificate-owner-ref` takes over.
To give more context, some other resources, such as the Pod resource,
will mutate the object when the value is "empty", for example the
`imagePullPolicy` value will default to `IfNotPresent`.

**PostIssuancePolicyChain**

In ([policies.go#L95](https://github.com/cert-manager/cert-manager/blob/b78af1ef867f8776715cae3dd6a8b83049c4d9b2/internal/controller/certificates/policies/policies.go#L95-L104)), cert-manager does a few sanity checks right after the issuer (either an
internal or an external issue) has filled the CertificateRequest's status
with the signed certificate.

One of the checks is called
[`SecretOwnerReferenceValueMismatch`](https://github.com/cert-manager/cert-manager/blob/b78af1ef867f8776715cae3dd6a8b83049c4d9b2/internal/controller/certificates/policies/checks.go#L511)
and checks that the owner reference on the Secret resource matches the one
on the Certificate resource.

### Test Plan

- Unit tests for the changes in the secret manager controller.


### Graduation Criteria

We propose to release this feature in GA immediately and skip the "beta"
phase that consists of gathering user feedback, since this feature has a
low user-facing surface. We think that we will be able to take a good
decision (e.g., the name of the new field, whether it is a boolean or a
string, and which values the field can take) while developing the feature
in the PR.

We don't think this feature needs to be [feature gated][feature gate].

[feature gate]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md

### Upgrade / Downgrade Strategy

Upgrading from a version without this feature to a version with this
feature won't be breaking.

Downgrading, however, will be a breaking change, since a new field will be
introduced.

### Supported Versions

This feature will be supported in all the versions of Kubernetes that are
supported by cert-manager.

## Production Readiness

<!--
This section should confirm that the feature can be safely operated in production environment and can be disabled or rolled back in case it is found to increase failures.
-->

### How can this feature be enabled / disabled for an existing cert-manager installation?

<!--

Can the feature be disabled after having been enabled?

Consider whether any additional steps will need to be taken to start/stop using this feature, i.e change existing resources that have had new field added for the feature before disabling it.


Do the test cases cover both the feature being enabled and it being disabled (where relevant)?

-->

### Does this feature depend on any specific services running in the cluster?

No.

### Will enabling / using this feature result in new API calls (i.e to Kubernetes apiserver or external services)?

No.

### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

### Will enabling / using this feature result in significant increase of resource usage? (CPU, RAM...)

No.

## Drawbacks

## Alternatives

It is possible to use the
[`csi-driver`](https://github.com/cert-manager/csi-driver) to circumvent
the problem of "too many ephemeral Secret resources stored in etcd". Using
the CSI driver, no Secret resource is created, alleviating the issue.

Since Flant offers its customers the capability to use Certificate resources,
and wants to keep supporting the Certificate type, switching from Certificate
resources to the CSI driver isn't an option.