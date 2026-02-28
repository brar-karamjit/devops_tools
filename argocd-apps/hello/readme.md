# Hello microservice

Tiny Flask-based service used by the MomentInMotion Django app to demonstrate a second service in the Istio mesh.

Directory structure is automatically picked up by the `all-apps-generator` ApplicationSet in `devops_tools/argocd/applicationset-all.yaml`. Place any Kubernetes manifests here and Argo CD will deploy them into the `hello` namespace.

The included `deployment.yaml` creates a Deployment+Service (and optional ingress). Update the image reference to point at your built `hello-service` container.
