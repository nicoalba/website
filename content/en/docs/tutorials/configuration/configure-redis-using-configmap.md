---
reviewers:
- eparis
- pmorie
title: Configure Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

This tutorial provides a real-world example of how to configure [Redis](https://redis.io/docs/latest/) using a [ConfigMap](../../concepts/configuration/configmap.md).

## {{% heading "objectives" %}}

* Create a ConfigMap with Redis configuration values.
* Create a Redis Pod that mounts and uses the created ConfigMap.
* Verify that the configuration was correctly applied.

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* The example on this page works with `kubectl` 1.14 and above.
* Ensure you understand how to [Configure a Pod to Use a ConfigMap](../../tasks/configure-pod-container/configure-pod-configmap.md).

<!-- lessoncontent -->

## Real world example: Configure Redis using a ConfigMap

Follow these steps to configure a Redis cache using data stored in a ConfigMap:

1. Run this command to create a ConfigMap with an empty configuration block:

    ```shell
    cat <<EOF >./example-redis-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-redis-config
    data:
      redis-config: ""
    EOF
    ```

    A file named `example-redis-config.yaml` is created in your current directory.

2. Run these commands to apply the ConfigMap created above, along with a Redis pod manifest:

    ```shell
    kubectl apply -f example-redis-config.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

3. Run this command to examine the contents of the Redis pod manifest YAML file:

    ```
    curl https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

    Check for these:
    
    * A volume named `config` is created by `spec.volumes[1]`.
    * The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
    * The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

    This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config` ConfigMap above as `/redis-master/redis.conf` inside the Pod:

    {{% code_sample file="pods/config/redis-pod.yaml" %}}

4. Run this command to examine the created objects:

    ```shell
    kubectl get pod/redis configmap/example-redis-config 
    ```

    You should see this output:

    ```
    NAME        READY   STATUS    RESTARTS   AGE
    pod/redis   1/1     Running   0          8s

    NAME                             DATA   AGE
    configmap/example-redis-config   1      14s
    ```

5. Run this command to verify that the `redis-config` key in the `example-redis-config` ConfigMap is still blank:

    ```shell
    kubectl describe configmap/example-redis-config
    ```

    You should see an empty `redis-config` key:

    ```shell
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ```

6. Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

    If successful, the command drops you into the `redis-cli` prompt:

    ```shell
    127.0.0.1:6379>
    ```

    This verifies that the Redis Pod is running and accessible.

7. Check `maxmemory`:

    **Note**: Don't include the `127.0.0.1:6379>` part of the code block when copying and pasting the commands.

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should show the default value of 0:

    ```shell
    1) "maxmemory"
    2) "0"
    ```

8. Check `maxmemory-policy`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    This should also yield its default value of `noeviction`:

    ```shell
    1) "maxmemory-policy"
    2) "noeviction"
    ```

9. Add the highlighted configuration values to the `example-redis-config.yaml` ConfigMap:

    **Note**: Don't forget to include the `|` operator.

    {{% code_sample file="pods/config/example-redis-config.yaml" %}}

10. Apply the updated ConfigMap:

    Exit the `redis-cli` interactive session:

    ```shell
    EXIT
    ```

    Then run:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

11. Confirm that the ConfigMap was updated:

    ```shell
    kubectl describe configmap/example-redis-config
    ```

    You should see the configuration values we just added:

    ```shell
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ----
    maxmemory 2mb
    maxmemory-policy allkeys-lru
    ```

12. Use `kubectl exec` to enter the pod and run the `redis-cli` tool again to see if the configuration was applied:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

    Repeat steps 7 and 8 to check `maxmemory` and `maxmemory-policy`.
  
    The configuration values should *not* have changed because the Pod needs to be restarted to grab updated values from associated ConfigMaps. 
    
13. Delete and recreate the Pod:

    Exit the `redis-cli` interactive session:

    ```shell
    EXIT
    ```

    Then run:

    ```shell
    kubectl delete pod redis
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

14. Re-enter the `redis-cli` interactive session to check the configuration values one last time:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

15. Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should now return the updated value of `2097152`:

    ```shell
    1) "maxmemory"
    2) "2097152"
    ```

16. Check that `maxmemory-policy` has also been updated:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    It should now reflect the desired value of `allkeys-lru`:

    ```shell
    1) "maxmemory-policy"
    2) "allkeys-lru"
    ```

17. Clean up your work by deleting the created resources:

    Exit the `redis-cli` interactive session:

    ```shell
    EXIT
    ```

    Then run:

    ```shell
    kubectl delete pod/redis configmap/example-redis-config
    ```

## {{% heading "whatsnext" %}}

* Learn more about [ConfigMaps](../../tasks/configure-pod-container/configure-pod-configmap.md).
* Follow an example of [Updating configuration via a ConfigMap](updating-configuration-via-a-configmap.md).
