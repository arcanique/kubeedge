# Usage

## Prerequisites

+ [Install docker](https://docs.docker.com/install/)

+ [Install kubeadm/kubectl](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

+ [Creating cluster with kubeadm](<https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/>)

+ KubeEdge supports https connection to Kubernetes apiserver. 

  Enter the path to kubeconfig file in controller.yaml
  ```yaml
  controller:
    kube:
      ...
      kubeconfig: "path_to_kubeconfig_file" #Enter path to kubeconfig file to enable https connection to k8s apiserver
  ```
  
+ (Optional) KubeEdge also supports insecure http connection to Kubernetes apiserver for testing, debugging cases.
  Please follow below steps to enable http port in Kubernetes apiserver.

    ```shell
    vi /etc/kubernetes/manifests/kube-apiserver.yaml
    # Add the following flags in spec: containers: -command section
    - --insecure-port=8080
    - --insecure-bind-address=0.0.0.0
    ```
  
  Enter the master address in controller.yaml
    ```yaml
    controller:
      kube:
        ...
        master: "http://127.0.0.1:8080" #Note if master and kubeconfig are both set, master will override any value in kubeconfig.
    ```

### Clone KubeEdge

```shell
git clone https://github.com/kubeedge/kubeedge.git $GOPATH/src/github.com/kubeedge/kubeedge
cd $GOPATH/src/github.com/kubeedge/kubeedge
```

### Configuring MQTT mode

The Edge part of KubeEdge uses MQTT for communication between deviceTwin and devices. KubeEdge supports 3 MQTT modes:
1) internalMqttMode: internal mqtt broker is enabled.
2) bothMqttMode: internal as well as external broker are enabled.
3) externalMqttMode: only external broker is enabled.

Use mode field in [edge.yaml](https://github.com/kubeedge/kubeedge/blob/master/edge/conf/edge.yaml#L4) to select the desired mode.

To use KubeEdge in double mqtt or external mode, you need to make sure that [mosquitto](https://mosquitto.org/) or [emqx edge](https://www.emqx.io/downloads/edge) is installed on the edge node as an MQTT Broker.

### Generate Certificates

RootCA certificate and a cert/key pair is required to have a setup for KubeEdge. Same cert/key pair can be used in both cloud and edge.

```bash
# $GOPATH/src/github.com/kubeedge/kubeedge/build/tools/certgen.sh genCertAndKey edge
```

The cert/key will be generated in the `/etc/kubeedge/ca` and `/etc/kubeedge/certs` respectively.

## Run KubeEdge

### Run Cloud

#### Run as a binary

+ Build Cloud and edge

    ```shell
    cd $GOPATH/src/github.com/kubeedge/kubeedge
     make 
    ```

+ Build Cloud

    ```shell
    cd $GOPATH/src/github.com/kubeedge/kubeedge
     make all WHAT=cloud
    ```

+ The path to the generated certificates should be updated in `$GOPATH/src/github.com/kubeedge/kubeedge/cloud/conf/controller.yaml`. Please update the correct paths for the following :
    + cloudhub.ca
    + cloudhub.cert
    + cloudhub.key

+ Create device model and device CRDs.
    ```shell
    cd $GOPATH/src/github.com/kubeedge/kubeedge/build/crds/devices
    kubectl create -f devices_v1alpha1_devicemodel.yaml
    kubectl create -f devices_v1alpha1_device.yaml
    ```

+ Run cloud
    ```shell
    cd $GOPATH/src/github.com/kubeedge/kubeedge/cloud
    # run edge controller
    # `conf/` should be in the same directory as the cloned KubeEdge repository
    # verify the configurations before running cloud(cloudcore)
    ./cloudcore
    ```

+ (**Optional**)Run `admission`, this feature is still being evaluated.
    please read the docs in [install the admission webhook](../../build/admission/README.md)


#### [Run as Kubernetes deployment](../../build/cloud/README.md)

### Run Edge

#### Deploy the Edge node
We have provided a sample node.json to add a node in kubernetes. Please make sure edge-node is added in kubernetes. Run below steps to add edge-node.

+ Modify the `$GOPATH/src/github.com/kubeedge/kubeedge/build/node.json` file and change `metadata.name` to the name of the edge node
+ Deploy node
    ```shell
    kubectl apply -f $GOPATH/src/github.com/kubeedge/kubeedge/build/node.json
    ```
+ Transfer the certificate file to the edge node

#### Run Edge

##### Run as a binary
+ Build Edge

    ```shell
    cd $GOPATH/src/github.com/kubeedge/kubeedge
    make all WHAT=edge
    ```

    KubeEdge can also be cross compiled to run on ARM based processors.
    Please follow the instructions given below or click [Cross Compilation](../setup/cross-compilation.md) for detailed instructions.

    ```shell
    cd $GOPATH/src/github.com/kubeedge/kubeedge/edge
    make edge_cross_build
    ```

    KubeEdge can also be compiled with a small binary size. Please follow the below steps to build a binary of lesser size:

    ```shell
    apt-get install upx-ucl
    cd $GOPATH/src/github.com/kubeedge/kubeedge/edge
    make edge_small_build
    ```

    **Note:** If you are using the smaller version of the binary, it is compressed using upx, therefore the possible side effects of using upx compressed binaries like more RAM usage, 
    lower performance, whole code of program being loaded instead of it being on-demand, not allowing sharing of memory which may cause the code to be loaded to memory 
    more than once etc. are applicable here as well. There are options in upx-ucl like --best, but it has a drawback of taking too long for big files, we are using --9 since it takes less time.
    Please refer to upx-ucl --help for more information.

+ Modify the `$GOPATH/src/github.com/kubeedge/kubeedge/edge/conf/edge.yaml` configuration file
    + Replace `edgehub.websocket.certfile` and `edgehub.websocket.keyfile` with your own certificate path
    + Update the IP address of the master in the `websocket.url` field. 
    + replace `edge-node` with edge node name in edge.yaml for the below fields :
        + `websocket:URL`
        + `controller:node-id`
        + `edged:hostname-override`

+ Run edge

    ```shell
    # run mosquitto
    mosquitto -d -p 1883
    # or run emqx edge
    # emqx start
    
    # run edgecore
    # `conf/` should be in the same directory as the cloned KubeEdge repository
    # verify the configurations before running edge(edgecore)
    ./edgecore
    # or
    nohup ./edgecore > edgecore.log 2>&1 &
    ```

    **Note:** Please run edge using the users who have root permission.

##### [Run as container](../../build/edge/README.md)

#### [Run as Kubernetes deployment](../../build/edge/kubernetes/README.md)

#### Check status

After the Cloud and Edge parts have started, you can use below command to check the edge node status.

```shell
kubectl get nodes
```

Please make sure the status of edge node you created is **ready**.

If you are using HuaweiCloud IEF, then the edge node you created should be running (check it in the IEF console page).

## Deploy Application

Try out a sample application deployment by following below steps.

```shell
kubectl apply -f $GOPATH/src/github.com/kubeedge/kubeedge/build/deployment.yaml
```

**Note:** Currently, for applications running on edge nodes, we don't support `kubectl logs` and `kubectl exec` commands(will support in future release), support pod to pod communication running on **edge nodes in same subnet** using edgemesh.

Then you can use below command to check if the application is normally running.

```shell
kubectl get pods
```

## Run Tests

### Run Edge Unit Tests

 ```shell
 make edge_test
 ```

 To run unit tests of a package individually.

 ```shell
 export GOARCHAIUS_CONFIG_PATH=$GOPATH/src/github.com/kubeedge/kubeedge/edge
 cd <path to package to be tested>
 go test -v
 ```

### Run Edge Integration Tests

```shell
make edge_integration_test
```

### Details and use cases of integration test framework

Please find the [link](https://github.com/kubeedge/kubeedge/tree/master/edge/test/integration) to use cases of intergration test framework for KubeEdge.
