# Template: getting started

Common constraints and prose rules: see [.agent/skills/unifabric-docs/SKILL.md](../../.agent/skills/unifabric-docs/SKILL.md).

**Position**: for new users with no unifabric knowledge. One single document
covers the installation for every environment (InfiniBand / Spectrum-X /
SONiC RoCE / ...): environments differ only in a few Helm values, so they are
sections of one document, not separate documents. Goal: the reader goes from
an empty cluster to "I can see unifabric work" in 30 minutes.

## Skeleton

```markdown
# Deploy unifabric               <!-- The title has no "Getting Started:"
                                      prefix. The document location
                                      already shows the type. -->

## Introduction and Deployment Goals   <!-- 2-3 sentences: what this document
                                           completes, and what the reader sees
                                           at the end. -->
## Prerequisites        <!-- A checklist: hardware (switch and NIC models),
                             Kubernetes version, dependent components,
                             permissions. The reader can check each item. -->
## Installation         <!-- helm install and similar steps. Each step shows
                             the command and the expected output. -->
## Configure Topology Discovery  <!-- One subsection per physical network.
                                      Start with a table that tells the reader
                                      which subsection to pick. Each subsection
                                      contains only the environment-specific
                                      prerequisites and Helm values. Deeper,
                                      post-install procedures (for example
                                      switch onboarding) live in a usage
                                      document; link to it. -->
## Verification         <!-- The smallest verifiable example: run
                             kubectl get fabricnodes and see the topology,
                             or check the Node labels.
                             Attach real output or a screenshot. -->
## Troubleshooting      <!-- Common problems in the deployment stage. -->
## Uninstall
## Next Steps           <!-- Links: usage documents for each feature,
                             reference documents. -->
```

## Constraints

1. **Each command is followed by its expected output.** The output must be a
   real capture. See common constraint 3 in the skill.
2. Do not assume any unifabric knowledge. When the document needs a concept,
   explain it in one sentence and link to the concepts document. Do not
   explain internals.
3. **All environments stay in this one document.** Give the criteria for
   choosing an environment at the start of the topology-discovery section.
   Inside each environment subsection, the reader follows one straight line to
   the end. Do not branch again.
4. Each prerequisite must be concrete and checkable.
   (Correct: "NVIDIA Spectrum-X switch, SONiC >= 202305."
   Wrong: "a compatible switch.")
5. An environment subsection contains only the values that are specific to
   that environment. Multi-step procedures that are not required to finish the
   installation (for example switch-agent onboarding or label semantics) live
   in a usage document; the subsection links to it. When a common step would
   repeat, extract it into one linked location, so that the copies do not
   become stale.
6. Section names are identical across the whole series. All documents use
   "Introduction and Deployment Goals". Do not use "Deployment Goal" in one
   document and "Choose a Scenario" in another.
