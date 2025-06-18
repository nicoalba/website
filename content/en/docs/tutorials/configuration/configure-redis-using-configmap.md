---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

This tutorial provides a real-world example of how to configure [Redis](https://redis.io/docs/latest/) using a [ConfigMap](../../concepts/configuration/configmap.md) and builds on the [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task. 

## {{% heading "objectives" %}}

* Create a ConfigMap with Redis configuration values.
* Create a Redis Pod that mounts and uses the created ConfigMap.
* Verify that the configuration was correctly applied.

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* The example on this page works with `kubectl` 1.14 and above.
* Understand [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).

<!-- lessoncontent -->

## Real World Example: Configuring Redis using a ConfigMap

Follow the steps below to configure a Redis cache using data stored in a ConfigMap.

1. Create a ConfigMap with an empty configuration block:

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

2. Apply the ConfigMap created above, along with a Redis pod manifest:

    ```shell
    kubectl apply -f example-redis-config.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

3. Examine the contents of the Redis pod manifest and note the following:
    
    * A volume named `config` is created by `spec.volumes[1]`.
    * The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
    * The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

    This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config` ConfigMap above as `/redis-master/redis.conf` inside the Pod:

    {{% code_sample file="pods/config/redis-pod.yaml" %}}

4. Examine the created objects:

    ```shell
    kubectl get pod/redis configmap/example-redis-config 
    ```

    You should see the following output:

    ```
    NAME        READY   STATUS    RESTARTS   AGE
    pod/redis   1/1     Running   0          8s

    NAME                             DATA   AGE
    configmap/example-redis-config   1      14s
    ```

    Recall that we left `redis-config` key in the `example-redis-config` ConfigMap blank:

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

5. Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

6. Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should show the default value of 0:

    ```shell
    1) "maxmemory"
    2) "0"
    ```

7. Similarly, check `maxmemory-policy`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    Which should also yield its default value of `noeviction`:

    ```shell
    1) "maxmemory-policy"
    2) "noeviction"
    ```

8. Add configuration values to the `example-redis-config` ConfigMap:

    {{% code_sample file="pods/config/example-redis-config.yaml" %}}

9. Apply the updated ConfigMap:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

10. Confirm that the ConfigMap was updated:

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

11. Check the Redis Pod again using `redis-cli` via `kubectl exec` to see if the configuration was applied:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

12. Check `maxmemory`:
   
    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It remains at the default value of 0:

    ```shell
    1) "maxmemory"
    2) "0"
    ```

13. Similarly, check `maxmemory-policy` which remains at the `noeviction` default setting:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    Returns:

    ```shell
    1) "maxmemory-policy"
    2) "noeviction"
    ```

    The configuration values have not changed because the Pod needs to be restarted to grab updated values from associated ConfigMaps. 
    
14. Delete and recreate the Pod:

    ```shell
    kubectl delete pod redis
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

15. Re-check the configuration values one last time:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

16. Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should now return the updated value of 2097152:

    ```shell
    1) "maxmemory"
    2) "2097152"
    ```

17. Similarly, check that `maxmemory-policy` has also been updated:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    It now reflects the desired value of `allkeys-lru`:

    ```shell
    1) "maxmemory-policy"
    2) "allkeys-lru"
    ```

18. Clean up your work by deleting the created resources:

    ```shell
    kubectl delete pod/redis configmap/example-redis-config
    ```

## {{% heading "whatsnext" %}}

* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
