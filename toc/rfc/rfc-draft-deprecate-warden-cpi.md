# Meta
[meta]: #meta
- Name: Deprecate the Warden CPI
- Start Date: 2026-07-02
- Author(s): @mkocher
- Status: Draft
- RFC Pull Request: (fill in after submission)
- Related RFCs: [rfc-draft-resolute-raccoon-os](rfc-draft-resolute-raccoon-os.md)
- Affected Component(s): bosh-warden-cpi-release, bosh-docker-cpi-release, bosh-linux-stemcell-builder

## Summary

The BOSH ecosystem maintains two container-based CPIs: the Warden CPI and the Docker CPI. Both are used to run a local BOSH director and BOSH deployments inside containers for local development and CI pipelines. This RFC proposes deprecating the Warden CPI in favour of the Docker CPI, and publishing container image stemcells for Resolute and later under an `oci` name rather than `warden-boshlite`.

## Problem

We have two largely redundant solutions to local bosh development - the Warden CPI and the Docker CPI. Both of them require ongoing maintenance, such as the recent effort to make both of them work correctly under cgroups v2.

The practical concerns:

- The Warden CPI uses many privileged operations in its source code to work around Garden API limitations, eg creating loopback devices to use as volumes. As bosh moves to running CPIs inside of BPM these operations are no longer possible. The right fix would be a large investment in both the CPI and Garden to expand the surface area of the API to encompass all of our needs.

- AI agents inherently know more about how to work with Docker. Agents are more adept when validating or troubleshooting with docker, making them faster and consuming fewer tokens. (eg: using the docker cli vs nsenter or [gaol](https://github.com/contraband/gaol))

- Local development on a Mac is easier with Docker than with Garden. There are any number of pre-made solutions which provide a Docker API locally on a Mac. 

## Proposal

### Deprecate the Warden CPI

- **At RFC Approval** — Mark `bosh-warden-cpi-release` as deprecated. Add a deprecation notice to the repository README and to [bosh.io](https://bosh.io).

- **Within 3 months** — Pipelines in both the Foundational Infrastructure WG and the App Runtime Deployments WG that use the Warden CPI are migrated to the Docker CPI.

- **End of deprecation period** — Stop cutting new releases of `bosh-warden-cpi-release`. Remove its CI pipelines and archive the repository.

### Support the docker-cpi container image outside the foundational infrastructure working group

The BOSH team produces the [bosh/docker-cpi](https://ghcr.io/cloudfoundry/bosh/docker-cpi) container image which it uses in CI which contains docker, the releases necessary for launching a bosh director, and a script which launches docker and runs create-env.

This has proven very useful in many pipelines. An example of a task which uses it can be found in the [bosh-package-nginx-release test task](https://github.com/cloudfoundry/bosh-package-nginx-release/blob/main/ci/tasks/test.sh).

While likely not the right solution for testing cf-deployment, we believe this will prove useful to many teams producing and testing bosh releases.

Support will entail publishing timely updates, avoiding breaking changes if possible, and announcing any deprecation with plenty of advanced notice.

### Stemcell rename: `warden-boshlite` → `oci`

Starting with Resolute, container image stemcells will be published under an `oci` family name. Existing `warden-boshlite` stemcells for Noble and earlier are not affected.

| Old name (Noble and earlier) | New name (Resolute and later) |
|---|---|
| `bosh-warden-boshlite-ubuntu-noble` | `bosh-oci-ubuntu-resolute` (new family, Resolute onwards) |

## Workstreams

### Foundational Infrastructure WG

- Mark `bosh-warden-cpi-release` as deprecated on GitHub and [bosh.io](https://bosh.io)
- Publish Resolute container image stemcells under the `oci` name in `bosh-linux-stemcell-builder`
- Migrate all BOSH team pipelines from Warden CPI to Docker CPI
- Archive `bosh-warden-cpi-release` at the end of the deprecation period
- Remove the warden-cpi related ops files from bosh-deployment

### App Runtime Deployments WG

- Migrate any pipelines that use the Warden CPI to the Docker CPI
- Update any references to `warden-boshlite` stemcell names to the `oci` equivalents
