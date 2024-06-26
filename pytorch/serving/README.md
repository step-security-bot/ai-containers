# TorchServe

[TorchServe](https://pytorch.org/serve/) is a performant, flexible and easy to use tool for serving PyTorch models in production.

## Configuration

Setting up TorchServe for your production application may require additional steps depending on the type of model you are serving and how that model is served.

### Archive Model

The [Torchserve Model Archiver](https://github.com/pytorch/serve/blob/master/model-archiver/README.md) is a command-line tool found in the torchserve container as well as on [pypi](https://pypi.org/project/torch-model-archiver/). This process is very similar for the [TorchServe Workflow](https://github.com/pytorch/serve/tree/master/workflow-archiver).

Follow the instructions found in the link above depending on whether you are intending to archive a model or a workflow. Use the provided container rather than installing the archiver with the example command below:

```bash
curl -O https://download.pytorch.org/models/squeezenet1_1-b8a52dc0.pth
docker run --rm -it \
           -v $PWD:/home/model-server \
           intel/intel-optimized-pytorch:2.2.0-serving-cpu \
           torch-model-archiver --model-name squeezenet \
            --version 1.0 \
            --model-file model-archive/model.py \
            --serialized-file squeezenet1_1-b8a52dc0.pth \
            --handler image_classifier \
            --export-path /home/model-server
```

### Test Model

Test Torchserve with the new archived model. The example below is for the squeezenet model.

```bash
# Assuming that the above pre-archived model is in the current working directory
docker run -d --rm --name server \
          -v $PWD:/home/model-server/model-store \
          --net=host \
          intel/intel-optimized-pytorch:2.2.0-serving-cpu
# Verify that the container has launched successfully
docker logs server
# Attempt to register the model and make an inference request
curl -X POST "http://localhost:8081/models?initial_workers=1&synchronous=true&url=squeezenet1_1.mar&model_name=squeezenet"
curl -O https://raw.githubusercontent.com/pytorch/serve/master/docs/images/kitten_small.jpg
curl -X POST http://localhost:8080/v2/models/squeezenet/infer -T kitten_small.jpg
# Stop the container
docker container stop server
```

### Modify TorchServe Config File

As demonstrated in the above example, models must be registered before they can be used for predictions. The best way to ensure models are pre-registered with ideal settings is to modify the included [config file](./config.properties) for the torchserve server.

1. Add your model to the config file

    ```properties
    ...
    cpu_launcher_enable=true
    cpu_launcher_args=--use_logical_core

    models={\
      "squeezenet": {\
        "1.0": {\
            "defaultVersion": true,\
            "marName": "squeezenet1_1.mar",\
            "minWorkers": 1,\
            "maxWorkers": 1,\
            "batchSize": 1,\
            "maxBatchDelay": 1\
        }\
      }\
    }
    ```

    > [!NOTE]
    > Further customization options can be found in the [TorchServe Documentation](https://pytorch.org/serve/configuration.html#config-model).

2. Test Config File

    ```bash
    # Assuming that the above pre-archived model is in the current working directory
    docker run -d --rm --name server \
              -v $PWD:/home/model-server/model-store \
              -v $PWD/config.properties:/home/model-server/config.properties \
              --net=host \
              intel/intel-optimized-pytorch:2.2.0-serving-cpu
    # Verify that the container has launched successfully
    docker logs server
    # Check the models list
    curl -X GET "http://localhost:8081/models"
    # Stop the container
    docker container stop server
    ```

    Expected Output:

    ```json
    {
      "models": [
        {
          "modelName": "squeezenet",
          "modelUrl": "squeezenet1_1.mar"
        }
      ]
    }
    ```

### Simple MaaS on K8s

Using the provided [helm chart](../charts/inference) your model can scale to multiple nodes in Kubernetes (K8s). Once you have set your `KUBECONFIG` environment variable and can access your cluster, use the below instructions to deploy your model as a service.

1. Install [Helm](https://helm.sh/docs/intro/install/)

    ```bash
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 && \
    chmod 700 get_helm.sh && \
    ./get_helm.sh
    ```

2. (Optional) Push TorchServe Image to a Private Registry

    If you added layers to an existing torchserve container image in a [previous step](#test-model), use `docker push` to add that image to a private registry that your cluster can access.

3. Set up Model Storage

    Your model archive file will no longer be accessible from your local environment, so it needs to be added to a [PVC](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) using a network storage solution like [NFS](https://kubernetes.io/docs/concepts/storage/volumes/#nfs).

4. Install TorchServe Chart

    Using the provided [Chart README](../charts/inference/README.md) set the variables found in the table to match the expected model storage, cluster type, and model configuration for your service. The example below assumes that a PVC has been created with the squeezenet model found in the root directory of the volume.

    ```bash
    helm install \
        --namespace=<namespace> \
        --set deploy.image=intel/intel-optimized-pytorch:2.2.0-serving-cpu \
        --set deploy.models='squeezenet=squeezenet1_1.mar' \
        --set deploy.storage.pvc.enable=true \
        --set deploy.storage.pvc.claimName=squeezenet \
        ipex-serving \
        ../charts/inference
    ```

5. Test Service

    By default the service is a `NodePort` service, and is accessible from the ip address of any node in your cluster. Find a node ip with `kubectl get node -o wide` and attempt to communicate with service using the command below:

    ```bash
    curl -X GET http://<your-node-ip>:30000/ping
    curl -X GET http://<your-node-ip>:30001/models
    ```

> [!NOTE]
> If you are under a network proxy, you may need to unset your `http_proxy` and `no_proxy` to communicate with the nodes in your cluster with `curl`.

#### Next Steps

There are some additional steps that can be taken to prepare your service for your users:

- Enable [Autoscaling](https://github.com/pytorch/serve/blob/master/kubernetes/autoscale.md#autoscaler) via Prometheus
- Enable [Intel GPU](https://github.com/intel/intel-device-plugins-for-kubernetes/blob/main/cmd/gpu_plugin/README.md#install-to-nodes-with-intel-gpus-with-fractional-resources)
- Enable [Metrics](https://pytorch.org/serve/metrics.html) and [Metrics API](https://pytorch.org/serve/metrics_api.html).
- Enable [Profiling](https://github.com/pytorch/serve/blob/master/docs/performance_guide.md#profiling).
- Export an [INT8 Model for IPEX](https://github.com/pytorch/serve/blob/f7ae6f8281ac6e26404a6ae4d210535c9dc96d9a/examples/intel_extension_for_pytorch/README.md#creating-and-exporting-int8-model-for-intel-extension-for-pytorch)
- Integrate an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) to your service to serve to a hostname rather than an ip address.
- Integrate [MLFlow](https://github.com/mlflow/mlflow-torchserve).
- Integrate an [SSL Certificate](https://pytorch.org/serve/configuration.html#enable-ssl) in your model config file to serve models securely.

### KServe

Apply Intel Optimizations to KServe by patching the serving runtimes to use Intel Optimized Serving Containers with `kubectl apply -f patch.yaml`

> [!NOTE]
> You can modify this `patch.yaml` file to change the serving runtime pod configuration.

#### Create an Endpoint

1. Create a volume with the follow file configuration:

    ```text
    my-volume
    ├── config
    │   └── config.properties
    └── model-store
        └── my-model.mar
    ```

2. Modify your TorchServe Server Configuration with a model snapshot like the following:

    ```text
    ...
    enable_metrics_api=true
    metrics_mode=prometheus
    model_store=/mnt/models/model-store
    model_snapshot={"name":"startup.cfg","modelCount":1,"models":{"mnist":{"1.0":{"defaultVersion":true,"marName":"mnist.mar","minWorkers":1,"maxWorkers":5,"batchSize":1,"responseTimeout":120}}}}
    ```

   > The model snapshot **MUST** contain the keys `defaultVersion`, `marName`, `minWorkers`, `maxWorkers`, `batchSize`, and `responseTimeout`. Even if your model `.mar` includes those keys.

3. Create a new endpoint

    ```yaml
    apiVersion: "serving.kserve.io/v1beta1"
    kind: "InferenceService"
    metadata:
      name: "ipex-torchserve-sample"
    spec:
      predictor:
        model:
          modelFormat:
            name: pytorch
          protocolVersion: v2
          storageUri: pvc://my-volume
    ```

4. Test the endpoint

    ```bash
    curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models
    ```

5. Make a Prediction
   Use this python script to convert your input to a bytes format:

   ```python
   import base64
   import json
   import argparse
   import uuid

   parser = argparse.ArgumentParser()
   parser.add_argument("filename", help="converts image to bytes array", type=str)
   args = parser.parse_args()

   image = open(args.filename, "rb")  # open binary file in read mode
   image_read = image.read()
   image_64_encode = base64.b64encode(image_read)
   bytes_array = image_64_encode.decode("utf-8")
   request = {
       "inputs": [
            {
                "name": str(uuid.uuid4()),
                "shape": [-1],
                "datatype": "BYTES",
                "data": [bytes_array],
            }
       ]
    }

    result_file = "{filename}.{ext}".format(
        filename=str(args.filename).split(".")[0], ext="json"
    )
    with open(result_file, "w") as outfile:
        json.dump(request, outfile, indent=4, sort_keys=True)
    ```

   Using the script will produce a json file to use as a prediction payload:

   ```bash
   curl -v -H "Host: ${SERVICE_HOSTNAME}" -X POST \
   http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/${MODELNAME}/infer \
   -d @./${PAYLOAD}.json
   ```

> [!TIP]
> You can find your `SERVICE_HOSTNAME` in the KubeFlow UI with the copy button and removing the `http://` from the url.

> [!TIP]
> You can find your ingress information with `kubectl get svc -n istio-system | grep istio-ingressgateway` and using the external IP and port mapped to `80`.
