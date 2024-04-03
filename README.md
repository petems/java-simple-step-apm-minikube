# java-simple-step-apm-minikube

A basic setup showing how Simple Step APM works for Java (aka. Automatic Tracer Injection)

The setup is as follows:

* Minikube 
* Datadog Agent 
* A Simple Java App without the Datadog tracer in it natively - [petems/datadog-java-apm-demo](https://github.com/petems/datadog-java-apm-demo)

**NB: We're assuming you're running with docker engine on an OSX `arm64` Macbook Pro, but should work on amd64 as I pushed a multi-build version of the image**

Automagically, the [Single Step APM](https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/single-step-apm/?tab=linuxhostorvm) setup will automatically inject the Datadog Java Tracer into the existing image

## Start minikube

```
minikube start
```

## Deploy datadog agent

```
$ kubectl create secret generic datadog-secret --from-literal api-key=YOURKEYHERE
$ helm install datadog-agent -f datadog-values.yaml datadog/datadog
```

## Wait for Datadog agent to confirm healthy

```
$ kubectl get pods -w
NAME                                           READY   STATUS    RESTARTS      AGE
datadog-agent-2bgmt                            3/3     Running   0             89s
datadog-agent-cluster-agent-6f446955b4-zjl55   0/1     Running   3 (47s ago)   89s
datadog-agent-cluster-agent-6f446955b4-zjl55   1/1     Running   3 (48s ago)   90s
```

## Run the Java app

```
kubectl apply -f java-app.yaml
```

## Wait for Java app to confirm healthy

```
kubectl get pods datadog-java-apm-demo
NAME                    READY   STATUS    RESTARTS   AGE
datadog-java-apm-demo   1/1     Running   0          18m
```

## Deploy it as a service

```
$ kubectl expose deployment java-app --type=LoadBalancer --port=8080
service/java-app exposed
$ kubectl get services
NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
datadog-agent                                      ClusterIP      10.99.167.139    <none>        8125/UDP,8126/TCP   105m
datadog-agent-cluster-agent                        ClusterIP      10.101.255.109   <none>        5005/TCP            105m
datadog-agent-cluster-agent-admission-controller   ClusterIP      10.109.153.38    <none>        443/TCP             105m
java-app                                           LoadBalancer   10.98.20.110     <pending>     8080:30800/TCP      4s
kubernetes                                         ClusterIP      10.96.0.1        <none>        443/TCP             106m
```

Run it as a minikube service so we can open it in the browser:

```
$ minikube service java-app
|-----------|----------|-------------|---------------------------|
| NAMESPACE |   NAME   | TARGET PORT |            URL            |
|-----------|----------|-------------|---------------------------|
| default   | java-app |        8080 | http://192.168.49.2:30800 |
|-----------|----------|-------------|---------------------------|
🏃  Starting tunnel for service java-app.
|-----------|----------|-------------|------------------------|
| NAMESPACE |   NAME   | TARGET PORT |          URL           |
|-----------|----------|-------------|------------------------|
| default   | java-app |             | http://127.0.0.1:52838 |
|-----------|----------|-------------|------------------------|
🎉  Opening service default/java-app in default browser...
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

## Observe the Single Step APM Settings

Do some pod describes so you can see the automatic injection working:

```
$ kubectl describe pod datadog-java-apm-demo | grep Volumes -C 14
      DD_ENTITY_ID:                      (v1:metadata.uid)
      DD_DOGSTATSD_URL:                 unix:///var/run/datadog/dsd.socket
      DD_TRACE_AGENT_URL:               unix:///var/run/datadog/apm.socket
      JAVA_TOOL_OPTIONS:                 -javaagent:/datadog-lib/dd-java-agent.jar -XX:OnError=/datadog-lib/continuousprofiler/tmp/dd_crash_uploader.sh -XX:ErrorFile=/datadog-lib/continuousprofiler/tmp/hs_err_pid_%p.log
    Mounts:
      /datadog-lib from datadog-auto-instrumentation (rw)
      /var/run/datadog from datadog (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w82b6 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-w82b6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  datadog:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/datadog
    HostPathType:  DirectoryOrCreate
  datadog-auto-instrumentation:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:   <unset>
```

You can see the `datadog-auto-instrumentation` volume set as an `EmptyDir` that shares the pods lifetime

We can also see the environment variables that add the `JAVA_TOOL_OPTIONS ` has worked as well:

```
$ kubectl describe pod datadog-java-apm-demo | grep Environment -C 9
      Finished:     Wed, 03 Apr 2024 13:53:40 +0100
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     50m
      memory:  20971520
    Requests:
      cpu:     50m
      memory:  20971520
    Environment:
      DD_INSTRUMENTATION_INSTALL_ID:    c75eace2-f3dd-4484-a5a8-f03056ccc739
      DD_INSTRUMENTATION_INSTALL_TIME:  1712147422
      DD_ENTITY_ID:                      (v1:metadata.uid)
      DD_DOGSTATSD_URL:                 unix:///var/run/datadog/dsd.socket
      DD_TRACE_AGENT_URL:               unix:///var/run/datadog/apm.socket
    Mounts:
      /datadog-lib from datadog-auto-instrumentation (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w82b6 (ro)
Containers:
--
    Container ID:   docker://703cff397f2736d41048e0949e69427fb1825457b24852968bea192344e5d562
    Image:          petems/datadog-java-apm-demo
    Image ID:       docker-pullable://petems/datadog-java-apm-demo@sha256:7ad25e7c0aa4c0104f3d98a8cb46bad8ec31619e8d7e471c3fe4eed25d67fc2a
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 03 Apr 2024 13:53:49 +0100
    Ready:          True
    Restart Count:  0
    Environment:
      DD_INSTRUMENTATION_INSTALL_TYPE:  k8s_lib_injection
      DD_INSTRUMENTATION_INSTALL_ID:    c75eace2-f3dd-4484-a5a8-f03056ccc739
      DD_INSTRUMENTATION_INSTALL_TIME:  1712147422
      DD_ENTITY_ID:                      (v1:metadata.uid)
      DD_DOGSTATSD_URL:                 unix:///var/run/datadog/dsd.socket
      DD_TRACE_AGENT_URL:               unix:///var/run/datadog/apm.socket
      JAVA_TOOL_OPTIONS:                 -javaagent:/datadog-lib/dd-java-agent.jar -XX:OnError=/datadog-lib/continuousprofiler/tmp/dd_crash_uploader.sh -XX:ErrorFile=/datadog-lib/continuousprofiler/tmp/hs_err_pid_%p.log
    Mounts:
      /datadog-lib from datadog-auto-instrumentation (rw)
```

We can also look at the logs for the app to see the Datadog tracer is running:

```
$ kubectl logs datadog-java-apm-demo | grep dd-java-agent.jar
Defaulted container "java-service" out of: java-service, datadog-lib-java-init (init)
Picked up JAVA_TOOL_OPTIONS:  -javaagent:/datadog-lib/dd-java-agent.jar -XX:OnError=/datadog-lib/continuousprofiler/tmp/dd_crash_uploader.sh -XX:ErrorFile=/datadog-lib/continuousprofiler/tmp/hs_err_pid_%p.log
```

## Jump into the Java app to create some traces

Now we know the tracer has been injected, we can create some traces:

```
$ kubectl exec -it datadog-java-apm-demo /bin/sh
# apk add curl
(1/5) Installing ca-certificates (20240226-r0)
(2/5) Installing brotli-libs (1.0.9-r6)
(3/5) Installing nghttp2-libs (1.47.0-r2)
(4/5) Installing libcurl (8.5.0-r0)
(5/5) Installing curl (8.5.0-r0)
Executing busybox-1.35.0-r17.trigger
Executing ca-certificates-20240226-r0.trigger
OK: 325 MiB in 22 packages
/ # curl localhost:8080/success
/ # curl localhost:8080/failure
{"timestamp":"2024-04-03T12:57:52.478+00:00","status":500,"error":"Internal Server Error","path":"/failure"}/ # exit
```

## Confirm Traces are showing in Datadog

* Go to the APM page and see the traces

![image](https://github.com/petems/java-simple-step-apm-minikube/assets/1064715/749a60f8-172e-48a1-bff0-49b19f9945b4)

## TODO

* Add curl into the image 
* Get the logs sending as well
* Better annotations and labelling

## Attributions

Built heavily on the work by @nilushancosta - https://medium.com/@nilushancosta/how-to-automatically-add-datadog-apm-libraries-for-java-into-containers-on-kubernetes-8ab0db2606bb