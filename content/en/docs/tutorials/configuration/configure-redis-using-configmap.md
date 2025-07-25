---
reviewers:
- eparis
- pmorie
title: Configure a Redis pod using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

A ConfigMap is a Kubernetes object that stores configuration data in key-value pairs. ConfigMaps can separate Redis settings from your container images, so you can update settings without rebuilding containers. You can also use different Redis configurations in development and production environments. Using ConfigMaps also makes it easier to manage version control for your Redis configuration data.

In this tutorial, you will learn:

* How to create a ConfigMap containing Redis configuration values
* How to apply the ConfigMap to a Redis pod in your Kubernetes cluster
* How to modify properties in the ConfigMap and reapply them to the pod
* How to verify that the configuration in your ConfigMap has been correctly applied

**Tip:**
If this is your first time using a ConfigMap to configure a pod, we suggest you start by following the more general [Configure a Pod to use a ConfigMap](http://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap) tutorial.

## {{% heading "prerequisites" %}}

Before you begin, you need access to a Kubernetes cluster, and you must have version 1.14 or later of the [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) command line tool installed on your local machine to interact with the cluster.

If you prefer not to use a real cluster, you can run this tutorial using [minikube](https://minikube.sigs.k8s.io/docs/) to simulate a full Kubernetes environment on your local machine. Alternatively, you can use one of the following Kubernetes playgrounds:

* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)
* [Play with Kubernetes](https://labs.play-with-k8s.com/)


<!-- lessoncontent -->

## Step 1 - Create an empty ConfigMap

Start by creating a ConfigMap with an empty configuration block using a YAML configuration file.

1. To specify your empty ConfigMap, create a manifest file named `example-redis-config.yaml` and paste in the following.

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: example-redis-config
    data:
    redis-config: ""
    ```

2. Create the ConfigMap in your cluster by running the following `kubectl` command.
   
    ```bash
    kubectl apply -f example-redis-config.yaml
    ```

3. To verify that the ConfigMap was created, run the following command.

    ```bash
    kubectl get configmap/example-redis-config
    ```

    You should see the following output.

    ```text
    NAME                             DATA   AGE
    configmap/example-redis-config   1      14s
    ```

## Step 2 - Create a Redis Pod using the ConfigMap configuration

Next create a Redis Pod in your cluster using the ConfigMap you created in the first step.

1. Run the following `kubectl` command to create the Pod.

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

    The manifest file contains the following configuration. The `volumeMounts` and `configMap` sections expose the data from the `example-redis-config` ConfigMap you created in the first step as `/redis-master/redis.conf` inside the Pod.

    {{% code_sample file="pods/config/redis-pod.yaml" %}}
   
2. To verify that your Pod has been successfully created, run the following command.

    ```bash
    kubectl get pod/redis
    ```

    You should see the following output.

    ```text
    NAME        READY   STATUS    RESTARTS   AGE
    pod/redis   1/1     Running   0          8s
    ```

3. The ConfigMap you created and applied to this Pod doesn't specify any values. You can confirm that your Pod has default values for `maxmemory` and `maxmemory-policy` by starting the Redis command-line client inside your Pod and using the `CONFIG GET` command.

    To do this, run the following commands.

    ```bash
    kubectl exec -it redis -- redis-cli
    127.0.0.1:6379> CONFIG GET maxmemory maxmemory-policy
    ```

    You should see the following output.

    ```text
    1) "maxmemory"
    2) "0"
    3) "maxmemory-policy"
    4) "noeviction"
    ```

4. To return out of the Redis command line to your terminal, enter `exit`.

## Step 3 - Update the ConfigMap

1. To update the ConfigMap, open the `example-redis-config.yaml` file you created in the first step and add `maxmemory` and `maxmemory-policy` values so that your file looks like the following:

   {{% code_sample file="pods/config/example-redis-config.yaml" %}}

2. Apply the updated ConfigMap by running the following command.

    ```bash
    kubectl apply -f example-redis-config.yaml
    ```

3. Run the following command to confirm that the ConfigMap has been updated.

    ```bash
    kubectl describe configmap/example-redis-config
    ```

    You should see the following output. Note that `maxmemory` and `maxmemory-policy` are now displaying the values in your updated configuration file

    ```text
    Name:          example-redis-config
    Namespace:     default
    Labels:        <none>
    Annotations:   <none>

    Data
    ====
    redis-config:
    ----
    maxmemory 2mb
    maxmemory-policy allkeys-lru
    ```

## Step 4 - Update the Redis Pod to use the new ConfigMap

Although the ConfigMap now contains the updated values you specified, those changes aren't yet reflected in your Redis Pod. This is because ConfigMaps are mounted at Pod startup time only. When you update the ConfigMap after your Redis Pod is already running, the Redis process inside the Pod continues to use the configuration that was available when the Pod first started.

You can confirm that this is the case by starting the Redis command-line client inside your Pod and using the `CONFIG GET` command as you did in Step 2:

```bash
kubectl exec -it redis -- redis-cli
127.0.0.1:6379> CONFIG GET maxmemory maxmemory-policy
```

You should still see the following output.

```text
1) "maxmemory"
2) "0"
3) "maxmemory-policy"
4) "noeviction"
```

Enter `exit` to return to your terminal.

To use your new configuration values, delete and recreate your Pod.

1. Run the following `kubectl` command to delete the Pod.

    ```bash
    kubectl delete Pod redis
    ```

2. Recreate the Pod using the same manifest file you used originally. When the Pod is created this time, your updated ConfigMap will be mounted.

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

3. Verify that your Pod has been successfully recreated by running the following command.

    ```bash
    kubectl get pod/redis
    ```

    You should see the following output.

    ```text
    NAME        READY     STATUS     RESTARTS   AGE
    pod/redis   1/1       Running    0          23s
    ```

4. Confirm that the new Pod is using the configuration values in your ConfigMap.

    ```bash
    kubectl exec -it redis -- redis-cli
    127.0.0.1:6379> CONFIG GET maxmemory maxmemory-policy
    ```

    This time, you should see the following output.

    ```text
    1) "maxmemory"
    2) "2mb"
    3) "maxmemory-policy"
    4) "allkeys-lru"
    ```

## Step 5 - clean up your resources

To delete the resources you created in this tutorial, run the following command.

```bash
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
