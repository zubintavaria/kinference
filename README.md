<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/assets/kinference-logo-dark.svg">
    <img alt="[K]inference" src="docs/assets/kinference-logo-light.svg" width="420">
  </picture>
</p>

<h3 align="center">The inference control plane for Kubernetes and vLLM</h3>

<p align="center">
  <a href="https://github.com/Kinference-AI/kinference-releases/releases"><img alt="Release" src="https://img.shields.io/github/v/tag/Kinference-AI/kinference?label=release&color=6dcc2f"></a>
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/badge/license-Apache_2.0-6dcc2f"></a>
  <a href="https://kinference.ai/docs/"><img alt="Documentation" src="https://img.shields.io/badge/docs-kinference.ai-6dcc2f"></a>
  <a href="https://github.com/Kinference-AI/kinference/discussions"><img alt="Discussions" src="https://img.shields.io/github/discussions/Kinference-AI/kinference?color=6dcc2f"></a>
</p>

<p align="center">
  <a href="https://kinference.ai/docs/getting-started/"><b>Get started</b></a> ·
  <a href="https://kinference.ai/docs/"><b>Docs</b></a> ·
  <a href="ARCHITECTURE.md"><b>Architecture</b></a> ·
  <a href="#benchmarks"><b>Benchmarks</b></a> ·
  <a href="https://github.com/Kinference-AI/kinference/discussions"><b>Discussions</b></a> ·
  <a href="CONTRIBUTING.md"><b>Contributing</b></a>
</p>

# [K]inference

🎈 Welcome to [K]inference! 👋

[K]inference is an inference _control plane_ for Kubernetes and vLLM. The name is pronounced "kay-inference", and can be written as `[K]inference` or `Kinference`.

At its core, [K]inference enables routing, load balancing, prompt classification and other actions for serving inference, through declarative policies. [K]inference is designed to be paired with a proxy. For example, it pairs with the [Buoyant Gateway](https://github.com/BuoyantIO/gateway/).

You can deploy any type of model with vLLM, on GPUs and server infrastructure that you have access to and can provision. [K]inference puts one OpenAI- and Anthropic-compatible endpoint in front of the vLLM workers. For each request, [K]inference picks the worker that it predicts will answer fastest, and it applies the routing policies that you declare.

## What you can do with [K]inference

[K]inference serves platform teams that run open-weight and proprietary models on their own GPUs. Application teams point their clients at one endpoint and change no code. The platform team decides, in policy, how [K]inference serves each request.

- **Get more work from every GPU in the pool.** Routing that cannot see each worker's cache and queue recomputes prompts or overloads busy workers. [K]inference sends each request to the worker that it predicts will return the first token soonest. When the pool has spare capacity, that worker is usually the one that already holds the prompt's prefix in its KV cache. As that worker gets busy, [K]inference sends requests to a less busy worker that computes the prompt from scratch but still answers sooner.
- **Measure every app, model and tenant.** vLLM's metrics do not say which app a request came from. [K]inference records the time to first token, the inter-token latency and the end-to-end latency of every request, by model, QoS class and tenant. The chart ships a Grafana dashboard for those metrics, so a team can track an inference Service Level Objective (SLO) as it tracks any other production SLO.
- **Burst to a hosted provider when the GPUs are full.** At peak load, requests wait in a worker's queue and may time out. A policy can exclude the workers that are too busy and send the overflow to a provider outside the cluster. [K]inference routes every request on its own, so new requests return to the local workers as soon as a worker's queue drains.
- **Serve coding agents on your own GPUs.** Coding agents resend the same files on almost every request, so those requests share long prefixes. Cache-aware load balancing helps most on requests that share prefixes. [K]inference accepts both OpenAI chat-completions requests and Anthropic Messages requests, so an agent that speaks either API needs only a new endpoint.
- **Route by what the prompt needs.** A policy can widen the set of models for a request and let the lowest predicted time to first token decide. For example, a policy can let a smaller model serve the prompts that a classifier tags as low complexity. The semantic classifiers that produce those tags are in development.

## What [K]inference does

| Capability | What it does | Status |
|------------|--------------|--------|
| **KV-cache-aware load balancing** | Predicts the time to first token of each request on each worker, from the worker's cached prefixes and its queue. Sends the request to the worker with the lowest prediction. | Available |
| **Inference policies** | An `InferencePolicy` matches requests by model, tenant, QoS class and token counts. Its rules narrow and rank the candidate endpoints. | Available |
| **Model aliases** | A worker set declares short aliases for a model, such as `gemma-4` for `google/gemma-4-12B-it`. | Available |
| **External providers** | An `InferenceProvider` declares a destination outside the cluster. A policy can prefer local workers and fall back to a provider. | Available |
| **OpenAI and Anthropic APIs** | Handles OpenAI chat completions and Anthropic Messages requests. | Available |
| **Inference metrics** | Exports Prometheus metrics for TTFT, ITL, request duration and prompt, cached and completion tokens, labeled by model, QoS class and tenant. | Available |
| **Offline policy checks** | `kinfctl check` finds mistakes in policy manifests without a cluster. | Available |
| **Token rate limiting** | Per-tenant token limits. | In development |
| **Semantic classifiers** | Tags on requests, such as prompt complexity or PII, that a policy can match. | In development |
| **Multiple replicas** | More than one [K]inference replica behind the gateway. | In development |

## Benchmarks

Buoyant compared [K]inference with a plain Kubernetes Service in front of the same vLLM deployment. Each arm had three vLLM pods, each on one L40S GPU, serving Qwen-3.8-27B-FP8. GuideLLM generated the load.

| Result | Workload | Kubernetes Service | [K]inference |
|--------|----------|--------------------|------------|
| **95% lower** median TTFT | Shared prefix, light load | 6.88 s | 0.31 s |
| **65% lower** median TTFT | Multi-turn, light load | 1.70 s | 0.59 s |
| **7.4× higher** peak goodput | Shared prefix, under load | 14.0 req/min | 103.0 req/min |
| **3.5× higher** peak goodput | Multi-turn, under load | 23.8 req/min | 82.7 req/min |
| **No added latency** at the median | No shared prefix, light load | 6.92 s | 6.83 s |

Goodput is the number of requests per minute that met the latency objective. The absolute numbers depend on the model and the hardware, so read the two arms against each other. The [detailed benchmark write-up, on the Buoyant blog](https://www.buoyant.io/blog/distributed-inference-benchmarks-kinference-vs-plain-kubernetes) has more information on the benchmarking setup, methodology, the confidence intervals and the full results.

## How it works

<p align="center">
  <img alt="Tenants send requests to the gateway. The gateway asks [K]inference for an endpoint and forwards each request to a vLLM worker or to an external provider. [K]inference reads policies from Kubernetes CRDs and KV-cache events and metrics from the workers." src="docs/assets/kinference-overview.svg" width="100%">
</p>

A gateway streams the request body to [K]inference over gRPC and asks for a pick. The body is an inference request, for example in OpenAI chat-completions or Anthropic Messages format. [K]inference returns the endpoint for that request, as well as the appropriately mutated body.

To make the pick, [K]inference monitors vLLM KV-cache events and engine metrics. [K]inference tokenizes the prompt and matches the prompt's prefix against the blocks that each worker has cached. [K]inference then predicts a wait time for each worker and returns the winner. [K]inference currently uses NVIDIA's [dynamo-kv-router](https://github.com/ai-dynamo/dynamo) crate to handle cache events.

[K]inference also monitors and sets the status on a set of Kubernetes CRDs that configure its behavior. If the request named an `InferencePool`, the pool's `InferencePolicy` first narrows and ranks the candidates, and the winner may be a provider outside the cluster.

[ARCHITECTURE.md](ARCHITECTURE.md) is the map of the code, and [docs/request-flow.md](docs/request-flow.md) follows one request step by step.

## Example policy

This `InferencePolicy` serves each request from the local workers when they have room, and sends the request to a provider when they do not.

```yaml
apiVersion: kinference.buoyant.io/v1alpha1
kind: InferencePolicy
metadata: { name: burst-policy, namespace: inference }
spec:
  rules:
  - name: burst-to-cloud
    route:
      exclude:
      - name: drop-overloaded-workers
        drop:
          backend:       { type: Local }
          queuedPrompts: { greaterThan: 2 }
      select:
      - { type: Local,    minimize: PredictedTTFT }
      - { type: Provider, order: [bedrock, databricks] }
```

- `exclude` drops every local worker that averaged more than 2 queued prompts over the last 30 seconds.
- The first `select` tier picks the remaining local worker with the lowest predicted time to first token.
- The second `select` tier applies when no local worker remains. It sends the request to the `InferenceProvider` named `bedrock`, and to the one named `databricks` when `bedrock` is not available.

See [the [K]inference CRD reference docs](https://kinference.ai/docs/reference/crds/) for every field.

## Quick start

You need a Kubernetes cluster, `kubectl` and `helm`. These commands install the Gateway API CRDs, the Buoyant Gateway and [K]inference into the `default` namespace.

```sh
GWAPI_VERSION=1.5.1
kubectl apply --server-side --force-conflicts -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v$GWAPI_VERSION/standard-install.yaml

helm upgrade -i gateway-crds oci://ghcr.io/buoyantio/charts/gateway-crds --devel
helm upgrade -i gateway oci://ghcr.io/buoyantio/charts/gateway --devel --wait

helm upgrade -i kinference-crds oci://cr.kinference.ai/kinference-ai/charts/kinference-crds --devel
helm upgrade -i kinference oci://cr.kinference.ai/kinference-ai/charts/kinference \
  --set extension.enabled=true \
  --devel --wait
```

`kubectl get po` then shows a `gateway-cp` pod and a `kinference` pod in the `Running` state.

[The [K]inference getting started guide](https://kinference.ai/docs/getting-started/) has the full steps, including how to install the CLI and how to serve your first model.

## CRDs

[K]inference primarily uses Kubernetes Custom Resource Definitions (CRDs) for configuration. The CRD schemas are in [charts/kinference-crds](charts/kinference-crds/templates). See [the [K]inference CRD reference docs](https://kinference.ai/docs/reference/crds/) for documentation.

| CRD | What it declares |
|-----|------------------|
| `InferenceWorkerSet` | A group of vLLM workers, the models that they serve and the aliases of those models. |
| `InferenceProvider` | A destination outside the cluster. |
| `InferencePool` | The destinations that a route may reach, and the policy that chooses among them. |
| `InferencePolicy` | The rules that match requests and select an endpoint. |

## kinfctl

This repo includes a `kinfctl` CLI tool. This CLI is designed to make it easy for you (or your agents) to validate policies before sending them to the server. See [the [K]inference kinfctl reference](https://kinference.ai/docs/reference/kinfctl/) for more.

To build the CLI from source, run `just install-cli`.

## Known limitations

[K]inference is alpha software. It has not seen sustained production exposure. Run at your own risk.

Several features that are in the full scope of the project are still in active development phases, including:

* Rate limiting
* Semantic classifiers
* Support for multiple replicas

These features, and others are coming soon.

## Related projects

[K]inference builds on several projects and shares questions with others. [docs/prior-art.md](docs/prior-art.md) lists them all and says where each one's design decisions are written down.

- [vLLM](https://github.com/vllm-project/vllm) is the engine that publishes the KV events and the metrics that [K]inference reads.
- [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) provides the `dynamo-kv-router` crate that [K]inference depends on for the radix tree.
- The [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension) defines an `InferencePool` API and the ext_proc protocol between a proxy and an endpoint picker.

## Documentation

- [Getting started](https://kinference.ai/docs/getting-started/): How to install [K]inference on a cluster.
- [Architecture](ARCHITECTURE.md): Overall project architecture.
- [Request flow](docs/request-flow.md): How requests flow through the system as a whole.
- [Local development](docs/local-development.md): How to build, install, and run locally.
- [Contributing](CONTRIBUTING.md): How to contribute to the project.

## Community

Please use [GitHub discussions](https://github.com/Kinference-AI/kinference/discussions)!

## Copyright and license

[K]inference is copyright 2026 Buoyant, Inc.

[K]inference is licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for the full text.
