# KEP-NNNN: Multi-Path Persistent Storage for Sandbox

<!--
TOC is auto-generated via `make toc-update`.
-->

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Use Case 1: Preserve user-installed packages across hibernate](#use-case-1-preserve-user-installed-packages-across-hibernate)
    - [Use Case 2: Preserve shell / editor configuration](#use-case-2-preserve-shell--editor-configuration)
    - [Use Case 3: Work with existing single-volume pattern](#use-case-3-work-with-existing-single-volume-pattern)
  - [High-Level Design](#high-level-design)
    - [API Changes](#api-changes)
    - [Controller Behavior](#controller-behavior)
    - [First-Boot Bootstrap](#first-boot-bootstrap)
    - [Relationship to `volumeClaimTemplates`](#relationship-to-volumeclaimtemplates)
    - [Relationship to Suspend / Snapshot Provider (#694)](#relationship-to-suspend--snapshot-provider-694)
    - [Safety: Path Validation](#safety-path-validation)
    - [Implementation Guidance](#implementation-guidance)
- [Scalability](#scalability)
- [Alternatives](#alternatives)
<!-- /toc -->

## Summary

This KEP proposes `persistentStorage` — a declarative, single-PVC, multi-mount mechanism for preserving **filesystem state at arbitrary paths** across Sandbox pod restarts (hibernate, PodFailed recovery, node eviction). Unlike the existing `volumeClaimTemplates`, which primarily serves a single user-data directory, `persistentStorage` lets a Sandbox keep `/root`, `/usr/local`, `/home/<user>`, and other system-adjacent paths stateful without forcing users to manage multiple PVCs or write bespoke init containers. First-boot content is seeded from the image's original filesystem via a short-lived init container, so users get "just works" behavior out of the box.

## Motivation

Agents and developer sandboxes frequently modify paths **outside** the designated user-data directory:

- `apt install <pkg>` writes to `/usr/bin/`, `/usr/lib/`, `/var/lib/dpkg/`.
- `pip install --user` writes to `~/.local/lib/python*/site-packages/`.
- `npm install -g` writes to `/usr/local/lib/node_modules/`.
- Shell history, `.bashrc`, `.zshrc`, `.vimrc`, `.ssh/known_hosts` all live in `$HOME`.
- `/etc/hosts`, custom `/etc/ssh/sshd_config`, or a user-added `/etc/cron.d/` entry.

With the current API, a user who hibernates their Sandbox loses all of the above on wake — only content under the explicitly mounted `volumeClaimTemplates` path survives. The UX is surprising ("I installed X yesterday, where did it go?") and pushes users toward workarounds like baking tools into custom images (every user → every team → image-explosion, exactly what agent-sandbox is trying to avoid).

Downstream platforms (including `lpai-api-agent-sandbox`) today work around this by requiring all user work to live under a single well-known path (typically `/workspace`), which is both a documentation burden and an incomplete solution — nothing catches configuration written to `/root/.config/` or packages installed to `/usr/local/`.

This KEP complements the existing `volumeClaimTemplates` field with a higher-level, more declarative surface for the common "I want N paths to survive pod restarts" use case.

### Goals

- Allow a user to declare **multiple paths** in the main container that persist across pod restarts using a **single backing PVC**.
- On **first pod start**, seed each persistent path with the image's original content at that path, so `/root/.bashrc` (shipped in the image) exists immediately.
- On **subsequent starts**, preserve user modifications — never clobber writes made during earlier pod incarnations.
- Remain **backward compatible** with the existing `volumeClaimTemplates` field — users choose one, the other, or both.
- Work with **any CSI driver** (no gVisor, CRIU, or cloud-specific dependency).

### Non-Goals

- **Memory / process state preservation.** Runtime state (open file descriptors, in-memory data, paused processes) is out of scope. That is covered by the GKE PodSnapshot extension (gVisor-only) and the broader Suspend/Snapshot provider discussion in [#694](https://github.com/kubernetes-sigs/agent-sandbox/issues/694).
- **Cross-sandbox data sharing.** PVCs remain RWO; sharing state between Sandboxes is a separate problem.
- **Persisting the entire container rootfs.** Full rootfs persistence requires deep runtime integration (OverlayFS upper-dir redirection, `docker commit`-style flows) that is orders of magnitude more complex to operate and outside the scope of this KEP.

## Proposal

Introduce a `persistentStorage` field on both `SandboxTemplateSpec` and `SandboxSpec` that lets users declare a set of paths to preserve, backed by a single controller-managed PVC.

### User Stories

#### Use Case 1: Preserve user-installed packages across hibernate

> "I `apt install jq` in my sandbox. I hibernate it at end of day. Tomorrow morning I wake it up. `jq` is still there."

The user declares `/usr/bin`, `/usr/lib`, `/var/lib/dpkg` as persistent paths in their template (or workspace create). On first boot, the image's original `/usr/bin` contents are cp'd to the PVC; from then on, every subsequent boot mounts the PVC subpaths in place, so `apt install` writes land on the PVC and persist.

#### Use Case 2: Preserve shell / editor configuration

> "I customized my `~/.bashrc` and installed oh-my-zsh. I don't want to redo that every time."

The user declares `/root` (or `/home/<user>`). The image's default `.bashrc` etc. seed the PVC on first boot, then the user's edits survive restarts.

#### Use Case 3: Work with existing single-volume pattern

> "I already have a `/workspace` PVC via `volumeClaimTemplates`. I want to additionally persist `/root`."

The user keeps their existing `volumeClaimTemplates` and adds a `persistentStorage` entry for `/root`. Both coexist; PVCs remain independent (separate resize, separate lifecycle).

### High-Level Design

#### API Changes

Add to `api/v1alpha1/sandbox_types.go`:

```go
// SandboxSpec adds:

// PersistentStorage, when set, provisions a single PVC whose subpaths
// are mounted at the declared container paths. On first pod start, each
// subpath is seeded from the image's original content at that path.
// +optional
PersistentStorage *PersistentStorageSpec `json:"persistentStorage,omitempty"`
```

```go
type PersistentStorageSpec struct {
    // Size is the total capacity of the backing PVC. The PVC is shared
    // across all Mounts (via subPath), so this is the sum of all paths'
    // expected space. Defaults to 20Gi when omitted.
    // +optional
    Size *resource.Quantity `json:"size,omitempty"`

    // StorageClassName for the backing PVC. nil/empty uses the cluster
    // default StorageClass.
    // +optional
    StorageClassName *string `json:"storageClassName,omitempty"`

    // Mounts is the list of container paths to persist.
    // +kubebuilder:validation:MinItems=1
    Mounts []PersistentMount `json:"mounts"`
}

type PersistentMount struct {
    // Path is the absolute path inside the main container to persist.
    // Must be unique within Mounts and may not be nested inside another
    // Mount's Path (e.g. declaring both /root and /root/.cache is
    // rejected).
    // +kubebuilder:validation:Pattern=`^/.+`
    Path string `json:"path"`

    // BootstrapFromImage controls first-boot seeding.
    //   - true  (default): an init container copies the image's content
    //     at Path into the PVC subpath BEFORE the main container starts.
    //     Subsequent boots skip seeding (the subpath is non-empty).
    //   - false: the subpath starts empty. Suitable for pure user-data
    //     directories (e.g. /workspace).
    // +optional
    BootstrapFromImage *bool `json:"bootstrapFromImage,omitempty"`
}
```

The same field is added to `extensions/api/v1alpha1/sandboxtemplate_types.go` so operators can declare persistent paths at the template level and have every derived Sandbox inherit them.

#### Controller Behavior

The `SandboxReconciler`'s existing reconciliation chain gains a new step `reconcilePersistentStorage`, called alongside `reconcilePVCs` and `reconcileService`:

1. **Ensure backing PVC.** If `persistentStorage.mounts` is non-empty, create (or adopt) a PVC named `<sandbox>-persist` with `Size`, `StorageClassName`, `AccessModes=[ReadWriteOnce]`, controller-ref back to the Sandbox. The same adoption pattern used by `reconcilePVCs` applies.
2. **Inject volume into pod spec.** Add one `Volume{Name: "lpai-persist", PersistentVolumeClaim: {ClaimName: <sandbox>-persist}}` to `pod.Spec.Volumes`.
3. **Inject volumeMounts on main container.** For each `Mount`, append to the main container's `VolumeMounts`:
   ```yaml
   - name: lpai-persist
     mountPath: <Mount.Path>
     subPath: <encode(Mount.Path)>   # e.g. "/root" → "root", "/usr/local" → "usr-local"
   ```
4. **Inject bootstrap init container.** For every Mount with `BootstrapFromImage != false`, add a single shared init container that runs **the same image as the main container** (so it can see the original `Path` content). The container mounts the PVC once at `/mnt/persist` (no subPath at the init stage) and runs:
   ```sh
   for p in <list of paths>; do
     sub=$(echo "$p" | sed 's|^/||; s|/|-|g')
     mkdir -p "/mnt/persist/$sub"
     if [ -z "$(ls -A "/mnt/persist/$sub" 2>/dev/null)" ] && [ -d "$p" ]; then
       cp -a "$p/." "/mnt/persist/$sub/" || true
     fi
   done
   ```
   The init container name is `persistent-storage-bootstrap`; it is a no-op on all boots after the first (empty-check short-circuits the copy).

#### First-Boot Bootstrap

The correctness hinges on the ordering:

1. PVC exists (from step 1) — may be empty (first boot) or populated (later boots).
2. Init container runs with the **un-mounted** image. It sees the original `/root`, `/usr/local`, etc. from the image layer. It writes to `/mnt/persist/<encoded-path>`.
3. Main container starts. For each Mount, kubelet sets up a subPath bind mount. The user sees either the image's content (first boot, just seeded) or their previously written content.

After first boot, image content at a persisted path is **frozen** from the user's perspective — even if the image is later updated, the PVC keeps the old content. This is a deliberate trade-off (consistent user state trumps automatic image upgrade), discussed further under "Alternatives" below.

#### Relationship to `volumeClaimTemplates`

`persistentStorage` and `volumeClaimTemplates` are **independent and composable**:

- `volumeClaimTemplates` remains the lower-level primitive: one entry per PVC, caller-authored PVC spec, caller-authored pod `volumeMounts`. Appropriate when the user wants precise control over AccessModes, VolumeAttributesClass, or a RWX volume shared with a sidecar.
- `persistentStorage` is the higher-level, opinionated primitive: one PVC for many paths, first-boot seeding, sensible defaults. Appropriate for "I just want these paths to persist."

Users may use both simultaneously. The controller creates one PVC per `volumeClaimTemplate` entry **plus** one `<sandbox>-persist` PVC when `persistentStorage` is set. No conflicts arise because the volume names are disjoint.

No existing field is modified or deprecated. Callers of the stable v1alpha1 API see zero behavior change.

#### Relationship to Suspend / Snapshot Provider (#694)

Issue [#694](https://github.com/kubernetes-sigs/agent-sandbox/issues/694) proposes a more general Suspend/Snapshot provider abstraction. `persistentStorage` fits cleanly underneath:

- A "filesystem" provider wraps the logic described in this KEP.
- A "pod-snapshot" provider (GKE gVisor) is an additional provider option users may choose when the underlying runtime supports it.
- A "CRIU" provider could be added later for memory snapshot support without touching this KEP's surface.

In this view, `persistentStorage` is the **universally-available** provider — the floor that every cluster has, which more capable providers can enrich but not replace. This KEP does not itself introduce the provider abstraction; it delivers the filesystem piece as a concrete, usable feature today, making #694 easier to realize later because at least one working implementation exists.

#### Safety: Path Validation

A validating webhook rejects a `persistentStorage` spec that:

- Has two Mounts with identical `Path`.
- Has a Mount whose `Path` is a prefix of another Mount's `Path` (e.g. `/root` and `/root/.cache`) — nested subPath mounts have undefined ordering.
- Uses a path on a hard block list: `/`, `/dev`, `/proc`, `/sys`, `/etc/passwd`, `/etc/shadow`, `/etc/ssh/*_host_key*`.

Sensitive-but-sometimes-legitimate paths (`/etc`, `/etc/ssh`) emit a warning through the API response but are not rejected; an admin may further restrict them with standard Pod Security Standards or OPA/Kyverno policies layered on top.

#### Implementation Guidance

- Reuse the ownership-adoption pattern in `reconcilePVCs` (`controllers/sandbox_controller.go:854`) — same `checkOwnership` / `SetControllerReference` flow, same `sandboxLabel=nameHash` on PVC labels.
- The bootstrap init container should **always** run, not just on first boot. The `ls -A` check inside the script makes it idempotent and cheap (~50ms on a populated PVC).
- Use a single init container regardless of mount count — iterating paths inside one shell is simpler than emitting N init containers.
- PVC resize: this KEP does not prescribe automatic resize. If a user edits `persistentStorage.size`, the controller updates the PVC `spec.resources.requests.storage` only when the new value is larger; shrinks are rejected (standard k8s PVC semantics).
- Deletion: PVC is GC'd via OwnerReference when the Sandbox is deleted. `shutdownPolicy=Retain` preserves the PVC alongside the Sandbox as expected.

## Scalability

- **Per-Sandbox overhead:** one PVC (already ~the same cost as `volumeClaimTemplates`), plus one init container that runs once and exits quickly. After the first boot, the init container's wall-clock impact is dominated by the image pull (which was already going to happen) plus the ~50ms `ls -A` check.
- **Per-cluster overhead:** the controller gains one more reconcile step (`reconcilePersistentStorage`) that is a no-op when the field is absent. No new controller-wide watches beyond the existing PVC watch.
- **Warm pool compatibility:** `persistentStorage` is set on the `SandboxTemplate` and inherited into pre-warmed Sandboxes. The PVC is provisioned as part of the pre-warm, so adoption latency is unchanged. First-boot seeding happens during pool warm-up, off the user-request critical path.

## Alternatives

**Alternative 1: Use multiple `volumeClaimTemplates` entries + hand-authored pod `volumeMounts`.**

Technically possible today. Rejected because:
- Users must manage N PVCs for N paths (N× cost, N× storage management).
- First-boot seeding still requires a user-authored init container — the most common request boils down to "I want the image's `/root/.bashrc` to show up, then persist my edits," and today every user reinvents the same 10-line init container.
- Operators can't offer "just persist these standard paths" as a platform default without making every Sandbox users a CRD expert.

**Alternative 2: Annotation-based declaration (`agents.x-k8s.io/persist-paths: "/root,/usr/local"`).**

Simpler API surface, but:
- No type safety, no per-path configuration (size, bootstrap flag).
- Silently ignored by older controllers, making the feature's presence unobservable in the spec.
- Validating webhooks on annotations are awkward.

Rejected.

**Alternative 3: Automatic image-update propagation.**

The proposal's "frozen after first boot" semantics means an image update does not automatically propagate new `/root/.profile` content to existing Sandboxes. Considered adding a "reset these paths on next start" knob (e.g. `agents.x-k8s.io/reset-persist-paths` annotation), but:
- Deferred to a follow-up KEP to keep this one focused.
- Users who need fresh image content can delete the PVC manually (one kubectl command) and the next pod start will re-seed.

**Alternative 4: Full rootfs persistence via OverlayFS upperdir redirection.**

Approaches like mounting the container's OverlayFS upperdir onto a PVC exist in research projects. Rejected because:
- Requires CRI modifications or a custom RuntimeClass; breaks the "works on any k8s" constraint.
- Couples Sandbox to a specific container runtime.
- Operationally complex: upperdir state can include half-written files, process-specific temp dirs, which then fail to cleanly restart.

The targeted-path approach in this KEP gives ~90% of the user-visible value at a fraction of the implementation and operational cost.
