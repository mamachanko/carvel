---
aliases: [/kapp-controller/docs/latest/kctrl-faq]
title: "`kctrl` FAQ"
---

### How can we add information to PackageMetadata generated by `kctrl package release`?
Any changes to the PackageMetadata reesource in the `package-resources.yml` file will be copied over to the released artifact.

### How can I build images while releasing a package using `kctrl`?
`kctrl` builds and pushes images using `kbld` if a `kbld` config is specified in the config being bundled into the package.

For example,
```yaml
# config/config-release.yml
apiVersion: kbld.k14s.io/v1alpha1
kind: Config
sources:
- image: simple-app
  path: .
destinations:
- image: simple-app
  newImage: 100mik/simple-app
```
Here, we are expressing that the image referred to as `simple-app` needs to be built from the root of the project (the path being `.`).
And pushed to OCI registry `100mik/simple-app`.

This file needs to reside in the paths passed to `ytt` in the AppSpec described in the PackageBuild resource. And also be included 
in the list of paths assigned to key `includePaths`.

### How can we add ytt overlays or additional configuration to upstream release artifacts?
Overlays can be created in a separate folder in the project directory. `kctrl` can be made aware of any additional folders by updating `package-build.yml` manually,
If the project directory looks something like this,
```bash
.
├── package-build.yml
├── package-resources.yml
├── upstream
│   └── cert-manager.yaml
├── overlays
│   └── overlay.yaml
│   └── values-schema.yaml
├── vendir.lock.yml
└── vendir.yml
```
Where, the directory overlays containing `ytt` files is created by the user. The fields `includePaths` and the `template` section of the App `spec` needs to be updated in `package-build.yml` like this,
```
apiVersion: kctrl.carvel.dev/v1alpha1
kind: PackageBuild
metadata:
name: certmanager.carvel.dev
spec:
release:
- resource: {}
template:
    spec:
    app:
        spec:
        deploy:
        - kapp: {}
        template:
        - ytt:
            paths:
            - upstream
            - overlays   # <= addition to package template
        - kbld: {}
    export:
    - imgpkgBundle:
        image: 100mik/certman-carvel-package
        useKbldImagesLock: true
        includePaths:
        - upstream
        - overlays    # <= ensure additional files are included in imgpkg bundle
```
This is to ensure that the package is aware of the additional files, while `includePaths` ensures that the folder is a part of the `imgpkg` bundle created by `kctrl`.

The template section in `package-resources.yml` should be updated in a similar fashion to ensure that `kctrl dev` yields accurate results.

`kctrl` generates the OpenAPI schema for a package if a values schema is provided.

### How can we go about updating a package dependent on an upstream release?
This can be done by running `kctrl package init` again and using the new tag/version. Alternatively, the value can be updated in the `vendir.yml` file and the new changes can be fetched by running `vendir sync`.

WARNING: This will overwrite any changes to the `/upstream` directory. It is recommended that any additional configuration or overlays should reside in a separate folder.

### Can `kctrl` be used to publish packages in a CI pipeline?
Yes! `kctrl` remembers the answers to questions that have been answered.
The `--yes` flag can be used to run the `release` command while using previously supplied
values if `package-resources.yml` and `package-build.yml` are committed to a repository with the source code.

### Can we provide our own `ImagesLock` resource instead of it being generated when we run the `pkg release` command?
Yes, the path containing the file will need to be added [here](/kapp-controller/docs/develop/kctrl-faq/#how-can-we-add-ytt-overlays-and-values-schema-for-upstream-release-artifacts), before updating `package-build.yml` and setting the key `useKbldImagesLock` in the `export` section to false.

### Can I build images from source while using `kctrl dev`?
Yes, if the `--kbld-build` command is supplied to the `dev` command, it will build images if a `kbld` config (similar to the one seen [here](https://carvel.dev/kapp-controller/docs/latest/kctrl-faq/#how-can-i-build-images-while-releasing-a-package-using-kctrl)) defines it.

NOTE: If your cluster points to the same `docker` daemon using used by `kbld` while building images. The images need not be pushed to a registry (only `sources` need to be defined).
This is useful in some development environments, for example, if you are using a `minikube` cluster, and your local enironment points to the 
`docker` daemon inside the `minikube` cluster, this can be set up by doing something like (`eval $(minikube docker-env)`)
