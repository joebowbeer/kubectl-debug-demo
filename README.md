Demonstrates [kubectl custom debug profiles](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/4292-kubectl-debug-custom-profile/README.md), which are *alpha* in kubectl 1.30.

Demos:
1. Custom [restricted](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/1441-kubectl-debug/README.md#profile-restricted) profile
    * Run an ephemeral debug container in a PSS-restricted pod
    * The custom profile specifies a numeric user to satisfy the pod's `runAsNonRoot` restriction, and
    * Mounts one of the pod's volumes in the debug container
1. Custom [sysadmin](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/1441-kubectl-debug/README.md#profile-sysadmin) profile
    * Run a privileged debug container in a PSS-restricted pod
    * NOTE: This may be blocked if [PSA](https://kubernetes.io/docs/concepts/security/pod-security-admission/) restrictions are enforced on the namespace
    * The custom profile overrides the pod's `runAsNonRoot` restriction, and
    * Mounts one of the pod's volumes in the debug container

The distroless node-express app was adapted from [distroless/examples/nodejs/node-express](https://github.com/GoogleContainerTools/distroless/tree/main/examples/nodejs/node-express)

Instructions:
1. Create a local Kubernetes cluster, such as [Rancher Desktop](https://rancherdesktop.io/)
1. Install [skaffold](https://skaffold.dev/docs/install/)
1. Deploy the `node-express` app to the `default` namespace
    ```shell
    skaffold run
    ```
1. Verify that the `default` namespace complies with [restricted](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) pod security standards
    ```shell
    kubectl label --dry-run=server --overwrite ns default \
    pod-security.kubernetes.io/enforce=restricted
    ```
1. Deploy a debug container using a custom `restricted` profile[^1]
    ```shell
    KUBECTL_DEBUG_CUSTOM_PROFILE=true kubectl debug -it $NODE_EXPRESS_PODNAME \
    --image=nixery.dev/arm64/shell/busybox/curl --target=node-express \
    --profile=restricted --custom=restricted-profile.json
    ```
1. Run `ps` in the debug container to verify that it shares the process namespace with the `node-express` container
1. Run `ls` in the debug container to verify that the `demo-volume` is mounted
1. Verify that `curl` works
    ```shell
    curl localhost:3000
    ```
1. Exit the container and verify that the `default` namespace still complies with the `restricted` pod security standards
    ```shell
    kubectl label --dry-run=server --overwrite ns default \
    pod-security.kubernetes.io/enforce=restricted
    ```
1. Deploy a privileged debug container using a custom `sysadmin` profile
    ```shell
    KUBECTL_DEBUG_CUSTOM_PROFILE=true kubectl debug -it $NODE_EXPRESS_PODNAME \
    --image=busybox --target=node-express \
    --profile=sysadmin --custom=sysadmin-profile.json
    ```
1. Run `ps` and `ls` as above
1. Verify that `wget` (from busybox) works
    ```shell
    wget -qO- localhost:3000
    ```
1. Exit the container and verify that the `default` namespace would now violate the `restricted` pod security standards, because a privileged container is now running in one of its pods
    ```shell
    kubectl label --dry-run=server --overwrite ns default \
    pod-security.kubernetes.io/enforce=restricted
    ```
1. Cleanup
    ```shell
    skaffold delete
    ```

[^1]: Use nixery.dev at your own risk