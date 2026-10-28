# Template: getting started

Common constraints and prose rules: see [.agent/skills/unifabric-docs/SKILL.md](../../.agent/skills/unifabric-docs/SKILL.md).

**Position**: for new users with no unifabric knowledge. One single document
covers the installation for every environment (InfiniBand / Spectrum-X /
SONiC RoCE / ...): the main flow installs the default environment, and the
other environments differ only in a few Helm values, so they are subsections
of one document, not separate documents. Goal: the reader goes from
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
## Installation         <!-- helm install and similar steps for the default
                             environment, including its topology-discovery
                             defaults. Each step shows the command and the
                             expected output. -->
## Other Network Environments    <!-- One subsection per non-default physical
                                      network. Each subsection contains only
                                      the environment-specific prerequisites
                                      and Helm values that override the
                                      default installation. Deeper,
                                      post-install procedures (for example
                                      switch onboarding) live in a usage
                                      document; link to it. -->
## Verification         <!-- The smallest verifiable example that proves the
                             deployment itself works: Pods run and
                             kubectl get fabricnodes matches the nodes.
                             Feature results (topology labels, metrics) are
                             verified in their usage documents.
                             Attach real output or a screenshot. -->
## Next Steps           <!-- Links: usage documents for each feature,
                             reference documents. -->
## Troubleshooting      <!-- Common problems in the deployment stage. -->
## Uninstall
```

## Constraints

1. **Each command is followed by its expected output.** The output must be a
   real capture. See common constraint 3 in the skill.
2. Do not assume any unifabric knowledge. When the document needs a concept,
   explain it in one sentence and link to the concepts document. Do not
   explain internals.
3. **All environments stay in this one document.** The main flow installs the
   default environment from start to end. Each other environment gets one
   subsection that lists only the Helm values to change. Inside a subsection,
   the reader follows one straight line to the end. Do not branch again.
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
