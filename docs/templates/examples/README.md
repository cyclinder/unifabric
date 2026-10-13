# Template Validation Examples

The documents in this directory test the templates in
[docs/templates/](../README.md) by result. Each example rewrites real Unifabric
content to follow one template exactly. Maintainers can review an example next to its
template and judge whether the template produces good documents.

| Example | Template | Rewritten from |
| --- | --- | --- |
| [project-readme.md](./project-readme.md) | [project-readme.md](../project-readme.md) | Root [README.md](../../../README.md) |
| [concepts-architecture.md](./concepts-architecture.md) | [concepts.md](../concepts.md) | Root README, [docs/README.md](../../README.md), [topology-discovery.md](../../topology-discovery.md) |
| [getting-started-sonic-roce.md](./getting-started-sonic-roce.md) | [getting-started.md](../getting-started.md) | [docs/getting-started-sonic-roce.md](../../getting-started-sonic-roce.md) |
| [usage-topology-visualization.md](./usage-topology-visualization.md) | [usage.md](../usage.md) | [docs/usage/topology-visualization.md](../../usage/topology-visualization.md) |
| [reference-fabricnode.md](./reference-fabricnode.md) | [reference.md](../reference.md) | [docs/reference/fabricnode.md](../../reference/fabricnode.md), verified against `pkg/api/v1beta1/fabricnode_types.go` |
| [design-topology-crd.md](./design-topology-crd.md) | [design.md](../design.md) | [docs/design/topology-crd.md](../../design/topology-crd.md) |

Notes for reviewers:

- The placeholder `<!-- TODO: capture output from a real environment -->` marks an
  expected-output block that must come from a real environment. The common constraints
  forbid fabricated outputs, and no environment was available during writing.
- All prose follows the simple-english (ASD-STE100) rules named in the common
  constraints.
- These files are review artifacts. They are not linked from the published
  documentation tree.
