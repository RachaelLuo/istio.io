---
title: Announcing Istio 1.14.5
linktitle: 1.14.5
subtitle: Patch Release
description: Istio 1.14.5 patch release.
publishdate: 2022-10-11
release: 1.14.5
---

This release includes security fixes in Go 1.18.7 (released 2022-10-04) for the `archive/tar`, `net/http/httputil`, and `regexp` packages.
This release also includes fixes to improve robustness.
This release note describes what is different between Istio 1.14.4 and Istio 1.14.5.

{{< relnote >}}

## Changes

- **Fixed** some `ServiceEntry` host names can cause non-deterministic Envoy routes.
  ([Issue #38678](https://github.com/istio/istio/issues/38678))

- **Fixed** `kube-inject` crashes when the pod annotation `proxy.istio.io/config` is set.

- **Fixed** an issue where the user can not delete the Istio Operator resource with revision if istiod is not running.  ([Issue #40796](https://github.com/istio/istio/issues/40796))

- **Fixed** an issue that the default `idleTimeout` for the passthrough cluster was changed to `0s` in 1.14.0, disabling the timeout. Restored the previous behavior to using Envoy's default value of 1 hour.  ([Issue #41114](https://github.com/istio/istio/issues/41114))

- **Fixed** a bug where the return dynamically generated by `jwks` was not base64 encoded, causing Envoy to fail to parse it.

- **Fixed** an issue where adding a `ServiceEntry` could affect an existing `ServiceEntry` with the same host name.
  ([Issue #40166](https://github.com/istio/istio/issues/40166))

- **Fixed** an issue where a root namespace `Sidecar` config would be ignored.

- **Fixed** the gateway API integration to not fail when the `v1alpha2` version is removed.
