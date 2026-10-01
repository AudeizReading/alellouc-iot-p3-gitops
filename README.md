# Release Beacon · GitOps

Kubernetes manifests for **Release Beacon**, the application used in Inception of Things P3 to demonstrate a Git-driven release change. The application reports its version and Pod hostname in both HTML and JSON: checking the HTTP response confirms which release is actually serving traffic, rather than only which image is declared in Git.

## Repository contract

The GitHub repository `alellouc-iot-p3-gitops` is intended to contain `README.md` and `release-beacon.yaml` at its root. Argo CD must follow branch `main`, source path `.`, and destination namespace `dev`; the manifest does not set a namespace, so the destination is essential.

| Resource | Name | Purpose |
| --- | --- | --- |
| Deployment | `release-beacon` | Runs one replica of `audeizreading/release-beacon:v1` |
| Service | `release-beacon` | Exposes the application through NodePort `30080` |
| External ConfigMap | `release-beacon-env` | Supplies runtime configuration; created by the cluster bootstrap, not stored here |

The public image is available on [Docker Hub](https://hub.docker.com/r/audeizreading/release-beacon).

Give it a try:

```bash
docker pull audeizreading/release-beacon:v1
```

The current manifest selects `v1`; `v2` must be built, tested and published before changing that reference. Image publication alone does not establish that an Argo CD deployment has succeeded.

## Prepare the deployment

The Linux work VM needs a running Docker Engine, a K3d cluster, Argo CD installed in namespace `argocd`, and namespace `dev`. Argo CD's Application must enable automatic synchronization; no manual application of `release-beacon.yaml` should be needed for the v1-to-v2 demonstration.

The bootstrap must create `release-beacon-env` in `dev` before the application starts. A ConfigMap of the same name in `default` will not satisfy the Deployment's reference. The environment supplied by mise on the host is not automatically inherited by Kubernetes Pods.

| Variable | Required value or role |
| --- | --- |
| `APP_NAME` | `release-beacon` |
| `PART` | `p3` |
| `PORT` | `8080`, matching the named container port `http` |
| `IOT_LEADER_LOGIN` | Maintainer login supplied by the bootstrap environment |
| `MAINTENER_MAIL` | Maintainer email supplied by the bootstrap environment |

These values remain outside Git but are stored in Kubernetes; a ConfigMap is not a secret store. Updating values consumed through `envFrom` requires replacement of the affected Pods, and Argo CD cannot restore this externally managed ConfigMap if it is deleted.

The displayed version comes from the application's `pyproject.toml` bundled inside the image. Do not add `APP_VERSION` to the ConfigMap to simulate an upgrade: the HTTP version must identify the deployed image's content, not a separately editable environment value.

## Reach the application without port-forward

When creating the K3d cluster, publish the first server node's NodePort to the Linux host with `--port "8888:30080@server:0"`. Changing the Service manifest cannot create this Docker port mapping after the fact; the cluster setup must provide it.

```text
Linux host :8888 → K3d server node :30080 → Service → Pod :8080
```

From the Linux host, open `http://localhost:8888/` for HTML or explicitly request JSON with the command below. From another machine, use the Linux VM's reachable IP instead of `localhost`; VM networking and firewall rules must allow the published port.

```bash
curl -fsS -H 'Accept: application/json' http://localhost:8888/
```

An illustrative v1 response is shown below. The Pod hostname changes when Kubernetes replaces an instance; the `version` field is the release check.

```json
{"app":"release-beacon","pod":"release-beacon-<pod-suffix>","version":"v1"}
```

## Demonstrate a release change

Run Kubernetes checks against the intended K3d context. A successful rollout confirms availability, while the HTTP response confirms the running application version; both are needed to distinguish a healthy old deployment from a successful upgrade.

```bash
kubectl config current-context
kubectl -n dev rollout status deployment/release-beacon --timeout=120s
kubectl -n dev get deployment,pods,service
curl -fsS -H 'Accept: application/json' http://localhost:8888/
```

1. Confirm the application reports `v1` and Argo CD shows the Application as `Synced` and `Healthy`.
2. Publish the visibly different v2 image as `audeizreading/release-beacon:v2`, built from application sources that report `v2`.
3. In `release-beacon.yaml`, change only the Deployment's image reference from `:v1` to `:v2`.
4. Commit and push that manifest change to `main`; allow Argo CD to detect and synchronize the new revision.
5. After synchronization, check the rollout, the browser presentation and the JSON `version` again. Do not use a manual `kubectl apply` to perform this upgrade.

Keep published release tags unchanged: the manifest uses `imagePullPolicy: IfNotPresent`, so overwriting an existing tag can leave cached code on a node. To return to v1, revert the image-reference change in Git and let Argo CD reconcile it; a direct cluster-only rollback would conflict with Git's desired state.

## Diagnose the failing layer

Use Pod events for missing configuration, image-pull or probe failures, and application logs for Python startup errors. Both probes request `/` through the named port `http`, so a runtime `PORT` different from `8080` breaks readiness, liveness and Service traffic together.

```bash
kubectl -n dev describe pods -l app=release-beacon
kubectl -n dev logs deployment/release-beacon
kubectl -n dev get endpointslices \
  -l kubernetes.io/service-name=release-beacon
```

If the Pod is ready but the host URL is unreachable, check the K3d/Docker port publication and host network path before changing the Deployment. If the image cannot start on the Linux VM, verify that its published architecture matches the Docker host; a build performed on an ARM Mac does not establish AMD64 compatibility.
