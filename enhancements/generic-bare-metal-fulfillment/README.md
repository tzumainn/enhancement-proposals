---
title: generic-bare-metal-fulfillment
authors:
  - Tzu-Mainn Chen
creation-date: 2026-02-10
last-updated: 2026-02-10
tracking-link:
  - N/A
see-also:
  - N/A
replaces:
  - "/enhancements/bare-metal-fulfillment"
superseded-by:
  - N/A
---

# Generic Bare Metal Fulfillment

## Summary

Recent discussions regarding cluster fulfillment have focused on narrowing its scope; it has been criticized as providing "Coffee-as-a-Service" rather than "Cluster-as-a-Service". This proposal argues that the generic nature of cluster fulfillment is a feature, and not a bug; and that - with a few refinements - we can adopt cluster fulfillment into "Bare-Metal-Cluster-as-a-Service", easing future development if/when additional bare metal workflows are requested.

## Motivation

The current trajectory of bare metal fulfillment within O-SAC is as follows:

* We have a cluster fulfillment workflow that provisions an OpenShift hosted cluster upon bare metal pre-provisioned as OpenShift agents. This workflow uses a template (corresponding to an Ansible role) that both acquires and configures the bare metal, and manages the OpenShift resources needed by a hosted cluster.
* We have an in-progress bare metal fulfillment workflow whose purpose is to simply acquire and configure bare metal infrastructure; this is intended to eventually be used as the basis of host configuration in cluster fulfillment, as well as other future bare metal fulfillment workflows.

Discussions around refinding cluster fulfillment have emphasized a desire to move away from the open-ended nature of the workflow: the templates can theoretically do anything (“Coffee-as-a-Service”), and user-inputs include a dictionary of template parameters that can contain arbitrary key/value pairs. Instead, we could put more of the required operations (such as creating NodePool and HostedCluster CRDs) directly into the operator and expose specific user inputs for the operator to consume. The end result is a tightly focused OpenShift cluster fulfillment workflow.

There are issues with this approach, however:

* It assumes all customers want the same cluster fulfillment workflow, with little to no variation.
* If we want to implement another bare metal fulfillment workflow with similar scope - for example, Slurm on bare metal -  we will have to make changes at every level of the architecture

Instead, we can create a generic bare metal fulfillment workflow. At a high level, it executes two steps:

* Acquire and configure hardware into a bare metal cluster
* Use a template to perform additional arbitrary steps to configure the bare metal cluster

For OpenShift clusters, Step 1 consists of acquiring the requested hardware from the underlying inventory (only selecting hosts that are already provisioned as Agents), and connecting them to an isolated network. Step 2 consists of creating the requisite NodePool and HostedCluster resources, as well as configuring components such as ingress and DNS and load balancing. An alternative bare metal fulfillment workflow might simply acquire hardware and put them on an existing provider network for Step 1; and do nothing for Step 2.

Adding another bare metal fulfillment workflow becomes far simpler: all you have to do is implement a new template. A generic bare metal CLI immediately allows a user to select the template and run the bare metal workflow to completion. Note that the CLI can be extended to accommodate a new bare metal fulfillment workflow if a simpler user experience is desired.

### User Stories

* As an O-SAC developer, I want a flexible bare metal fulfillment architecture so I can implement additional bare metal workflows with a minimum of fuss.
* As a cloud service provider, I want O-SAC to provide a path for additional bare metal workflows without requiring long development times and disruptive upgrades.

### Goals

* O-SAC supports a single generic bare metal fulfillment workflow.
* The workflow is used for both OpenShift clusters and simple bare metal clusters.
* The workflow supports easy expansion through the use of templates.

### Non-Goals

* This proposal does not discuss networking. That topic is discussed in a separate proposal; we expect the bare metal fulfillment workflow to eventually specify network parameters as well.
* This proposal does not go into detail regarding a low-level bare metal API; that is considered an implementation detail that is needed independent of this proposal.
* This proposal does not go into detail regarding refinement of the cluster fulfillment workflow to separate bare metal configuration and cluster configuration; that is considered an implementation detail that is needed independent of this proposal.

## Proposal

The bulk of this proposal is actually already implemented in O-SAC (or approved for implementation):

* The cluster fulfillment workflow accepts an arbitrary template
* The accepted bare metal fulfillment proposal details the intention to replace the bare metal configuration in cluster fulfillment with a low level bare metal fulfillment workflow

The modifications are as follows:

* Remove the concept of a `HostPool` representing a generic request for hosts; instead use the `ClusterOrder` for that purpose
* Do not remove the ability for cluster fulfillment to specify arbitrary templates

In order to support generic cluster fulfillment workflows, we would extend the `ClusterOrder` object with two additional fields:

* `profile`: a string used by O-SAC to place additional constraints upon host selection; for example, an `openshift` profile would only return hosts that are already provisioned as Agents
* `template_output`: a dictionary where templates can set arbitrary key/value pairs that can later be returned to the user

It is important to note that there have been concerns raised in regards to supporting arbitrary templates; those concerns are worth discussing here.

####  Template Parameter Validation

A template architecture makes it difficult to validate input parameters. In order to remediate this issue, our update template system can include a parameter specification which the cluster fulfillment operator explicitly checks as an initial step.

#### User Experience

A generic fulfillment workflow makes it difficult to create a targeted "push button" user experience, as additional parameters must be specified for the generic fulfillment API. For example, a sample request for an OpenShift cluster might be as follows:

```
  profile: openshift
  host_sets:
    fc430: 2
    h100: 1
  template: osac.templates.ocp_4_17_small
  template_input:
    pull_secret: some_secret
```

However, we can always extend user tools such as CLI. For example, an OpenShift cluster fulfillment CLI can call the exact same fulfillment APIs while supplying certain pre-sets (`profile: openshift`) and explicitly requiring specific input parameters intended for the template (`pull_secret`). This feature also allows for the creation of custom user experiences for customers who may have wildly different requirements in this area.

### Workflow Description

**O-SAC developer** is a human user responsible for developing a new bare metal workflow.

1. The O-SAC developer receives requirements for a new bare metal fulfillment workflow.
2. The O-SAC developer codes a new template that fulfills the desired bare metal workflow.
3. The O-SAC developer tests the bare metal workflow using the bare metal fulfillment CLI.
4. The O-SAC developer optionally extends the CLI to create a custom experience for the new bare metal workflow.

### API Extensions

Implementation would use the cluster fulfillment API and CRDs, and possibly involve renaming cluster fulfillment to bare metal fulfillment if a distinction is needed.

### Implementation Details/Notes/Constraints

This proposal does not radically alter cluster fulfillment; in fact, if we simply update cluster fulfillment to explicitly separate bare metal infrastructure configuration and cluster configuration, then we have essentially implemented the bulk of this idea. All other features can be built incrementally off that spine, and most (such as adding a Host API and CRD) have been discussed in separate enhancement proposals.

### Risks and Mitigations

N/A

### Drawbacks

N/A

## Alternatives (Not Implemented)

The alternative is to continue down our current path of having distinct cluster and bare metal fulfillment workflows, and understanding that implementing a new bare metal workflow will require substantial effort and changes to the code base.

## Open Questions [optional]

N/A

## Test Plan

TBD

## Graduation Criteria

N/A

### Removing a deprecated feature

N/A

## Upgrade / Downgrade Strategy

N/A

## Support Procedures

N/A

## Infrastructure Needed [optional]

N/A
