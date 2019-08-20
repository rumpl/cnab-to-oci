# How does cnab-to-oci work

This document describes how docker app (and cnab-to-oci) pushes a cnab to a registry.

The process is as follows:
* convert a docker app to a cnab bundle
* convert the bundle into an [oci index](https://github.com/opencontainers/image-spec/blob/master/image-index.md) (docker calls this a [manifest list](https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list), there are some minor differencies between the two)
* push the configuration
* copy/mount the invocation image and the service images
* push the index

## CNAB

[Cloud Native Application Bundles](https://cnab.io/) are a package format specification escribes a technology for bundling, installing, and managing distributed applications.

A CNAB bundle contains (among other things):

* Invocation images (`[]Image`)
* Component images (`map[string]Image`)
* Parameters
* Metadata
* Actions
* Creds
* Outputs
* Custom extension

Here is a sample bundle.json that we will be using during this document.

```json
{
	"schemaVersion": "v1.0.0-WD",
	"name": "helloworld",
	"version": "0.1.0",
	"description": "Simple docker app with a frontend and a backend",
	"maintainers": [
		{
			"name": "Djordje Lukic",
			"email": "djordje.lukic@docker.com"
		}
	],
	"invocationImages": [
		{
			"imageType": "docker",
			"image": "helloworld:0.1.0-invoc"
		}
	],
	"images": {
		"backend": {
			"imageType": "docker",
			"image": "docker/backend:1.0.0",
			"description": "docker/backend:1.0.0"
		},
		"frontend": {
			"imageType": "docker",
			"image": "docker/frontend:1.0.0",
			"description": "docker/frontend:1.0.0"
		}
	},
	"parameters": {
		"fields": {
			"backend_port": {
				"definition": "backend_port",
				"destination": {
					"env": "docker_param1"
				}
			}
		}
	},
	"definitions": {
		"backend_port": {
			"default": "80",
			"type": "string"
		}
	}
}
```

This bundle defines an application with:
* an invocation image (`technosophos/helloworld:0.1.0`), an invocation image is the main entry point for the application, this image must have an executable in `/var/cnab/run` this executable must define three actions: `install`, `upgrade` and `uninstall`. The invocation image is the entry point for managing the application.
* two services (`backend` and `frontend`), these represent the parts your application is built with.
* a parameter for the backend (`backend_port`), parameters represnet information about the application configuration

This example is kept simple and contain only the bare minimal definitions, if you wish to know more you can read the [full spec](https://github.com/deislabs/cnab-spec).

<details><summary>Click here to see the whole bundle.json</summary>

```json
{
	"schemaVersion": "v1.0.0-WD",
	"name": "helloworld",
	"version": "0.1.0",
	"description": "Simple docker app with a frontend and a backend",
	"maintainers": [
		{
			"name": "Djordje Lukic",
			"email": "djordje.lukic@docker.com"
		}
	],
	"invocationImages": [
		{
			"imageType": "docker",
			"image": "helloworld:0.1.0-invoc"
		}
	],
	"images": {
		"backend": {
			"imageType": "docker",
			"image": "docker/backend:1.0.0",
			"description": "docker/backend:1.0.0"
		},
		"frontend": {
			"imageType": "docker",
			"image": "docker/frontend:1.0.0",
			"description": "docker/frontend:1.0.0"
		}
	},
	"actions": {
		"com.docker.app.inspect": {
			"stateless": true
		},
		"com.docker.app.render": {
			"stateless": true
		},
		"io.cnab.status": {}
	},
	"parameters": {
		"fields": {
			"backend_port": {
				"definition": "backend_port",
				"destination": {
					"env": "docker_param1"
				}
			},
			"com.docker.app.kubernetes-namespace": {
				"definition": "com.docker.app.kubernetes-namespace",
				"applyTo": [
					"install",
					"upgrade",
					"uninstall",
					"io.cnab.status"
				],
				"destination": {
					"env": "DOCKER_KUBERNETES_NAMESPACE"
				}
			},
			"com.docker.app.orchestrator": {
				"definition": "com.docker.app.orchestrator",
				"applyTo": [
					"install",
					"upgrade",
					"uninstall",
					"io.cnab.status"
				],
				"destination": {
					"env": "DOCKER_STACK_ORCHESTRATOR"
				}
			},
			"com.docker.app.render-format": {
				"definition": "com.docker.app.render-format",
				"applyTo": [
					"com.docker.app.render"
				],
				"destination": {
					"env": "DOCKER_RENDER_FORMAT"
				}
			},
			"com.docker.app.share-registry-creds": {
				"definition": "com.docker.app.share-registry-creds",
				"destination": {
					"env": "DOCKER_SHARE_REGISTRY_CREDS"
				}
			}
		}
	},
	"credentials": {
		"com.docker.app.registry-creds": {
			"path": "/cnab/app/registry-creds.json"
		},
		"docker.context": {
			"path": "/cnab/app/context.dockercontext"
		}
	},
	"definitions": {
		"backend_port": {
			"default": "80",
			"type": "string"
		},
		"com.docker.app.kubernetes-namespace": {
			"default": "",
			"description": "Namespace in which to deploy",
			"title": "Namespace",
			"type": "string"
		},
		"com.docker.app.orchestrator": {
			"default": "",
			"description": "Orchestrator on which to deploy",
			"enum": [
				"",
				"swarm",
				"kubernetes"
			],
			"title": "Orchestrator",
			"type": "string"
		},
		"com.docker.app.render-format": {
			"default": "yaml",
			"description": "Output format for the render command",
			"enum": [
				"yaml",
				"json"
			],
			"title": "Render format",
			"type": "string"
		},
		"com.docker.app.share-registry-creds": {
			"default": false,
			"description": "Share registry credentials with the invocation image",
			"title": "Share registry credentials",
			"type": "boolean"
		}
	}
}
```
</details>

## Pushing

Pushing a bundle is performed in two phases, fixup and index push. The fixup phase checks that all the references are present in the repository, if they are not present it will mount then on that repository. The bundle is patched at this phase, adding digested references to the invocation and service images.

Once the bundle is fixed-up it will look something like this:

```json
{
	"schemaVersion": "v1.0.0-WD",
	"name": "helloworld",
	"version": "0.1.0",
	"description": "Simple docker app with a frontend and a backend",
	"maintainers": [
		{
			"name": "Djordje Lukic",
			"email": "djordje.lukic@docker.com"
		}
	],
	"invocationImages": [
		{
			"imageType": "docker",
			"image": "helloworld@sha256:6a92cd1fcdc8d8cdec60f33dda4db2cb1fcdcacf3410a8e05b3741f44a9b5998"
		}
	],
	"images": {
		"backend": {
			"imageType": "docker",
      "image": "docker/backend@sha256:a59a4e74d9cc89e4e75dfb2cc7ea5c108e4236ba6231b53081a9e2506d1197b6",
			"description": "docker/backend:1.0.0"
		},
		"frontend": {
			"imageType": "docker",
      "image": "docker/frontend@sha256:57334c50959f26ce1ee025d08f136c2292c128f84e7b229d1b0da5dac89e9866",
			"description": "docker/frontend:1.0.0"
		}
	},
	"parameters": {
		"fields": {
			"backend_port": {
				"definition": "backend_port",
				"destination": {
					"env": "docker_param1"
				}
			}
		}
	},
	"definitions": {
		"backend_port": {
			"default": "80",
			"type": "string"
		}
	}
}
```

Note how the references to the images have changed and now contain digests (`@sha256:...`). You can see the result of the fixup phase by running `cnab-to-oci fixup <BUNDLE_JSON>`

Once the bundle is patched we can push the OCI index (manifest list) with links to everything that was pushed. The OCI index has annotations, we use them to know what is the type it's pointing to (service image or config). Application name and version are put in the root annotations of the OCI index.

TODO: explain more how the whole thing works, use the info below, things left to talk about:

* config push, what's inside, how and why we push the partial bundle
* index push

## OCI

myregistry/user/repo:tag

cnab-to-oci will push an oci index (manifest list) to `myregistry/user/repo:tag`

The invocation image and the services images are also pushed to the repo but:

* `myregistry/user/repo:tag-invoc` < invocation image

* `myregistry/user/repo@sha256:xxxasjdaldsjflasd` < service 1
* `myregistry/user/repo@sha256:yyyasjdaldsjflasd` < service 2
* `myregistry/user/repo@sha256:zzzazzzaldsjflasd` < config

CNAB-TO-OCI does:

* push the bundle config as blob (bundle without metadata, invocation images or component images)
* copy/mount the service images
* copy/mount the invocation image

We need to push these to get the digests of the images/blobs.

Once we have all the digests we can push the oci index (manifest list) with links to everything that was pushed.
The oci index has annotations, we use them to know what is the type it's pointing to (service image or config).
Application name and version are put in the root annotations of the oci index.
