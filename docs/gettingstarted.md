# Getting Started

To fork this repository and use it as the base for your new plugin, a few changes need to be made.

## Customizing Your Plugin

By default, this repository will package the contents of the `plugin/` directory into the Docker image it builds.

Therefore, once you add your custom plugin logic (i.e. your Javascript code, static assets, etc.) to this repository, you will just need to ensure that the final generated assets are placed into the `plugin/` directory so that they can be served.

> **Note:** The contents of the `plugin/` directory is expected to be an [npm package](https://docs.npmjs.com/about-packages-and-modules#about-packages), which means that it should contain a file that describes the plugin at `plugin/package.json`.
>
> **The Docker image will fail to build if `plugin/package.json` is not found.**

Once the plugin directory contains the contents you would like to serve, simply run `REGISTRY= ORG=rancher REPO=ui-plugin-server TAG=v0.0.0 make` to build your `rancher/ui-plugin-server:v0.0.0` image (or replace the environment variables accordingly based on what image you would like to build).

Once you have built your image, you can run the following Docker command to serve your plugin at `http://localhost:8080`:

```bash
ORG=rancher
REPO=ui-plugin-server
TAG=v0.0.0

docker run -it --rm -p 8080:8080 ${ORG}/${REPO}:${TAG}
```

Once you have finished testing your changes, run `REGISTRY= ORG=rancher REPO=ui-plugin-server TAG=v0.0.0 docker push ${REGISTRY}${ORG}/${REPO}:${TAG}` to push your built image to a Docker registry of your choice (where `REGISTRY` would be your private Docker registry, if that is something that you are using. 

By default, the registry chosen by Docker when no registry is provided is the public [DockerHub](https://hub.docker.com/)).

## Customizing Your Helm Chart

Once your image has been pushed to a Docker registry, you can use the utility script at `./scripts/patch` by running the command `REGISTRY= ORG=rancher REPO=ui-plugin-server TAG=v0.0.0 make patch` to modify the Helm chart's `Chart.yaml` and `values.yaml` according to your specific plugin server's requirements.

In addition to using this script, you may also want to make the following changes to the Helm chart:

- Modifying general `Chart.yaml` metadata about your chart: common fields that you may want to modify by hand include the `description` of the chart, `annotations`, and `keywords`

- Modifying Rancher-specific annotations in the `Chart.yaml` metadata: the following well-known Rancher annotations may be something that any UI Plugin developer would want to modify:
  - `catalog.cattle.io/certified: rancher`: **should only be added to charts that are managed by Rancher itself, not custom developers**
  - `catalog.cattle.io/rancher-version: '>= 2.7.0-0 < 2.8.0-0'`: should indicate what version of Rancher this plugin is supported on
  - `catalog.cattle.io/kube-version: '>= 1.16.0-0 < 1.25.0-0'`: should indicate what version of Kubernetes this plugin is supported on. This annotation is preferred over using Helm's `kubeVersion` metadata field since enforcement of this annotation will not prevent in-place upgrades on Kubernetes clusters that fall outside of this semver range; however, the Helm `kubeVersion` will fail templating if the cluster's version falls outside the `kubeVersion` specified.

> **Important**:
>
> The following Rancher annotations should not be modified by a Plugin Developer:
>
> - `catalog.cattle.io/scope: management`: we always expect plugins to only be deployed in the management / local cluster
> - `catalog.cattle.io/namespace: cattle-ui-plugin-system`: we always expect plugins to be deployed in this namespace
> - `catalog.cattle.io/os: linux`: the image we build does not currently support deployment onto a Windows host
> - `catalog.cattle.io/permits-os: linux, windows`: by default, the Helm chart we have will **permit/tolerate** the existence of a Windows host in a cluster due to `nodeSelectors` and `tolerations` that are added to the Deployment contained within it
> - `catalog.cattle.io/ui-component: plugins`: ensures that the Plugin Helm Chart shows up under UI Plugins, not Apps & Marketplace
  
- Adding in a custom `NOTES.txt` to `templates/NOTES.txt`: as noted in the [Helm docs](https://helm.sh/docs/chart_template_guide/notes_files/), this will emit specific notes that you can show on a successful `helm install` or `helm upgrade` operation


## Deploying

### In Rancher (via Apps & Marketplace)

1. Navigate to `Apps & Marketplace -> Repositories` in your **local / management** cluster and create a Repository that has a `Target` of `http(s) URL to an index generated by Helm` where the `Git Repo URL` is your Git repository (i.e. `https://github.com/rancher/ui-plugin-server`) and the `Git Branch` is your Git branch (i.e. `main`)

2. Navigate to `Apps & Marketplace -> Charts`; you should see your chart under the new Repository you created

> Note: Starting Rancher 2.7.0, this will be available under Apps & Marketplace -> UI Plugins

3. Install the chart!

### In your local / management Kubernetes cluster (via running Helm 3 locally)

```bash
REPO=ui-plugin-server

PLUGIN_RELASE_NAME=${REPO}
helm upgrade --install --create-namespace -n cattle-ui-plugin-system ${PLUGIN_NAME} ./charts/ui-plugin-server
```

## Creating a Plugin Repository

For an example of how to create a repository that hosts multiple plugins in a single `pkg/` directory and simply copies over the Helm chart from this repository with some changes on an install, see [`scripts/publish` in `rancher/ui-plugins-example`](https://github.com/rancher/ui-plugin-examples/blob/main/scripts/publish). 

This script will allow you to dynamically generate and publish multiple UI Plugin Docker images + Helm charts on running the script.