# Supplemental Tools Build and Release Runbook

## Overview

The **supplemental-tools** group is a collection of container images that are built and released outside the main OpenShift payload. These images are not part of the OCP release payload (`for_payload: false`) and are not included in the standard release (`for_release: false`). Instead, they are standalone tools delivered to `registry.redhat.io` for customer use.

### Current Images

| Image | Delivery Repo | Source |
|-------|--------------|--------|
| block-copyfail | `openshift4/ose-block-copyfail-rhel9` | [openshift/block-copyfail](https://github.com/openshift/block-copyfail) |

### Key Properties

- **Group name:** `supplemental-tools`
- **Product:** `supplemental-tools`
- **Version:** `1.0.0`
- **Architectures:** `x86_64`, `aarch64`, `s390x`, `ppc64le`
- **Build system:** Konflux (hermetic builds with cachi2 enabled)
- **Konflux image repo:** `quay.io/redhat-user-workloads/ocp-art-tenant/supplemental-tools`
- **ocp-build-data branch:** `supplemental-tools`
- **freeze_automation:** `true` (automation does not run automatically; all operations are manual)

---

## Prerequisites

### Tools

| Tool | Purpose |
|------|---------|
| **doozer** | Image metadata management and build orchestration |
| **elliott** | Release/advisory management and Konflux snapshot operations |
| **oc** | OpenShift CLI for applying Release CRs to the Konflux cluster |
| **podman** / **skopeo** | Verifying image availability on registries |

### Access Requirements

- Access to the [build/layered-products](https://saml.buildvm.hosts.prod.psi.bos.redhat.com:8888/job/aos-cd-builds/job/build%2Flayered-products/) Jenkins job
- Permissions to run `elliott` and `doozer` commands against the `supplemental-tools` group
- `oc` access to the Konflux cluster where Release CRs are applied
- Access to [Brew](https://brewhub.engineering.redhat.com/brewhub) for build inspection
- Access to [Quay.io ocp-art-tenant](https://quay.io/organization/redhat-user-workloads) for Konflux image verification

---

## Build Process

Supplemental-tools images are built using the **layered-products** Jenkins job. This is a manual process since `freeze_automation` is set to `true`.

### Step-by-Step

1. **Navigate to the Jenkins job:**

   Go to the [build/layered-products](https://saml.buildvm.hosts.prod.psi.bos.redhat.com:8888/job/aos-cd-builds/job/build%2Flayered-products/) Jenkins job.

2. **Configure the build parameters:**

   - Set `--group=supplemental-tools` as the group parameter.
   - Select the image(s) to build (e.g., `block-copyfail`), or leave empty to build all images in the group.
   - Review any additional parameters as needed.

3. **Trigger the build:**

   Click **Build** to start the job. The Jenkins job will:
   - Rebase the source repository using doozer
   - Submit a Konflux build request
   - Build the image hermetically (network isolation with cachi2 dependency pre-fetching)
   - Build across all configured architectures (`x86_64`, `aarch64`, `s390x`, `ppc64le`)

4. **Monitor the build:**

   - Watch the Jenkins console output for progress.
   - Check the Konflux build pipeline in the `ocp-art-tenant` workspace for per-architecture build status.
   - On success, the built image will be available in the Konflux image repo:
     `quay.io/redhat-user-workloads/ocp-art-tenant/supplemental-tools`

### Using Doozer Directly (Alternative)

If needed, you can invoke doozer directly:

```bash
doozer --group=supplemental-tools images:konflux:build
```

---

## Snapshot Creation

After a successful build, create a Konflux snapshot to capture the built image references for release.

### Step-by-Step

1. **Create and apply a new snapshot:**

   ```bash
   elliott -g supplemental-tools snapshot new --apply
   ```

   This command will:
   - Query Konflux for the latest built images in the group
   - Create a new Snapshot custom resource
   - Apply it to the Konflux cluster

2. **Verify the snapshot:**

   Check that the snapshot was created successfully:

   ```bash
   elliott -g supplemental-tools snapshot list
   ```

   Confirm the new snapshot appears and references the correct image builds.

---

## Release Process

Once a snapshot exists, create a Release custom resource (CR) in the Konflux cluster to trigger the release pipeline.

### Step-by-Step

1. **Identify the snapshot name:**

   From the previous step, note the snapshot name (e.g., from `elliott snapshot list` output).

2. **Prepare the Release CR:**

   Create a Release CR YAML that references the snapshot and the appropriate release plan. Example:

   ```yaml
   apiVersion: appstudio.redhat.com/v1alpha1
   kind: Release
   metadata:
     name: supplemental-tools-<date>
     namespace: ocp-art-tenant
   spec:
     snapshot: <snapshot-name>
     releasePlan: <release-plan-name>
   ```

   Replace:
   - `<date>` with the current date (e.g., `2026-06-07`)
   - `<snapshot-name>` with the snapshot created in the previous step
   - `<release-plan-name>` with the release plan configured for supplemental-tools

3. **Apply the Release CR:**

   ```bash
   oc apply -f release.yaml
   ```

4. **Monitor the release pipeline:**

   ```bash
   oc get releases -n ocp-art-tenant
   ```

   Watch for the release to transition through its stages. The release pipeline will:
   - Push the image to the staging registry
   - Run any required tests and checks
   - Publish the image to `registry.redhat.io`

---

## Verification

After the release pipeline completes, verify that the image is available on `registry.redhat.io`.

### Step-by-Step

1. **Check image availability with skopeo:**

   ```bash
   skopeo inspect docker://registry.redhat.io/openshift4/ose-block-copyfail-rhel9:latest
   ```

   Verify:
   - The image digest matches the build output
   - All expected architectures are present (`x86_64`, `aarch64`, `s390x`, `ppc64le`)
   - The image labels and metadata are correct

2. **Check the manifest list for multi-arch:**

   ```bash
   skopeo inspect --raw docker://registry.redhat.io/openshift4/ose-block-copyfail-rhel9:latest | jq .
   ```

   Confirm the manifest list includes entries for all four architectures.

3. **Pull and test (optional):**

   ```bash
   podman pull registry.redhat.io/openshift4/ose-block-copyfail-rhel9:latest
   podman run --rm registry.redhat.io/openshift4/ose-block-copyfail-rhel9:latest --help
   ```

---

## Onboarding a New Image

To add a new image to the supplemental-tools group, follow these steps.

### 1. Prepare the Source Repository

- Ensure the source repository exists under the [openshift](https://github.com/openshift) GitHub org.
- A corresponding private mirror should exist under [openshift-priv](https://github.com/openshift-priv).
- The repository must contain a `Dockerfile` at the root (or specify a custom path).
- Add the `public_upstreams` mapping in `group.yml` if the source has a private/public pair.

### 2. Create the Image Metadata File

Create a new YAML file under `images/` on the `supplemental-tools` branch of `ocp-build-data`. Use `images/block-copyfail.yml` as a template:

```yaml
content:
  source:
    path: .
    dockerfile: Dockerfile
    git:
      allow_unprotected_branch: true
      branch:
        target: main
      url: git@github.com:openshift-priv/<repo-name>.git
      web: https://github.com/openshift/<repo-name>
distgit:
  branch: rhaos-{MAJOR}.{MINOR}-rhel-9
  component: <component-name>-container
delivery:
  repo_name: <component-name>
  delivery_repo_names:
  - openshift4/ose-<component-name>-rhel9
for_payload: false
for_release: false
enabled_repos:
- rhel-9-baseos-rpms
- rhel-9-appstream-rpms
from:
  builder:
  - stream: rhel9-custom
  stream: openshift-enterprise-base-rhel9
name: openshift4/ose-<component-name>-rhel9
owners:
- <team-email>@redhat.com
konflux:
  image_repo: quay.io/redhat-user-workloads/ocp-art-tenant/supplemental-tools
  cachi2:
    lockfile:
      backend: rpm-lockfile-prototype
```

Key fields to set:
- **`content.source.git.url`**: The private source repo URL
- **`content.source.git.web`**: The public source repo URL
- **`distgit.component`**: The Brew component name (must be registered)
- **`delivery.repo_name`**: The delivery repository name
- **`delivery.delivery_repo_names`**: The `registry.redhat.io` path(s) for the image
- **`name`**: The full image name
- **`owners`**: The team email for ownership
- **`from`**: The base image stream(s) -- see `streams.yml` for available streams

### 3. Update group.yml (if needed)

If the new image has a private/public upstream pair, add it to the `public_upstreams` list in `group.yml`:

```yaml
public_upstreams:
- private: "https://github.com/openshift-priv/<repo-name>"
  public:  "https://github.com/openshift/<repo-name>"
```

### 4. Set Up Konflux Resources

Coordinate with the ART team to ensure:
- The Konflux application includes the new component
- A release plan exists that covers the new image
- The delivery repo is registered in the Red Hat container catalog

### 5. Test the Build

Run a test build via the layered-products Jenkins job:

```bash
# Via Jenkins: set --group=supplemental-tools and select the new image
# Or via doozer directly:
doozer --group=supplemental-tools --images=<image-name> images:konflux:build
```

### 6. Commit and Merge

Once the test build succeeds:
1. Commit the new image metadata to the `supplemental-tools` branch
2. Get the PR reviewed and merged
3. Perform a full build/snapshot/release cycle as documented above

---

## Troubleshooting

### Build Failures

| Symptom | Possible Cause | Resolution |
|---------|---------------|------------|
| Hermetic build fails with network errors | cachi2 pre-fetch did not cache all dependencies | Check the cachi2 lockfile in the source repo. Ensure `rpm-lockfile-prototype` backend is generating a complete lockfile. Rebuild after fixing. |
| Architecture-specific build failure | Platform-specific code or dependency issue | Check the Konflux pipeline logs for the failing architecture. The build runs independently per-arch. |
| "freeze_automation is true" warning | Expected behavior | This is intentional. All supplemental-tools builds are manual. |
| Jenkins job fails early | Group configuration error | Verify the `supplemental-tools` branch of `ocp-build-data` has valid `group.yml` and image configs. Run `doozer --group=supplemental-tools images:print-name` to validate. |

### Snapshot Issues

| Symptom | Possible Cause | Resolution |
|---------|---------------|------------|
| `snapshot new` finds no images | No successful builds exist | Run a build first via the layered-products job. |
| Snapshot references stale builds | A newer build was expected | Re-run the build and create a new snapshot. |

### Release Issues

| Symptom | Possible Cause | Resolution |
|---------|---------------|------------|
| Release CR stays in pending state | Release plan not found or misconfigured | Verify the `releasePlan` name in the CR matches what is configured in the Konflux workspace. |
| Image not appearing on `registry.redhat.io` | Release pipeline still in progress or failed | Check the Release CR status: `oc get release <name> -n ocp-art-tenant -o yaml`. Look at the `status.conditions` for error details. |
| Multi-arch manifest missing an architecture | Architecture build failed silently | Check that all four architecture builds completed successfully before creating the snapshot. |

### Verification Issues

| Symptom | Possible Cause | Resolution |
|---------|---------------|------------|
| `skopeo inspect` returns 404 | Image not yet published or wrong tag | Wait for the release pipeline to complete. Check the delivery repo name matches what is in the image config. |
| Image digest mismatch | Stale cache or CDN propagation delay | Wait a few minutes and retry. Use `--no-cache` flags if available. |

---

## References

- **ocp-build-data branch:** `supplemental-tools` branch of [ocp-build-data](https://github.com/openshift-eng/ocp-build-data/tree/supplemental-tools)
- **Source repo:** [openshift/block-copyfail](https://github.com/openshift/block-copyfail)
- **Konflux workspace:** `ocp-art-tenant` on Quay.io
- **Jenkins job:** [build/layered-products](https://saml.buildvm.hosts.prod.psi.bos.redhat.com:8888/job/aos-cd-builds/job/build%2Flayered-products/)
- **Jira epic:** ART-18775
- **Post-mortem reference:** ART-18321 (block-copyfail)
- **Available streams** (from `streams.yml`):
  - `rhel9-custom` -- `registry.redhat.io/ubi9:latest`
  - `rhel9-minimal-custom` -- `registry.access.redhat.com/ubi9/ubi-minimal:latest`
  - `openshift-enterprise-base-rhel9` -- `registry.redhat.io/openshift/art-images-base:ose-4-21-openshift-enterprise-base-rhel9`
