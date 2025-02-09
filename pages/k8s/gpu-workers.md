---
wrapper_template: "templates/docs/markdown.html"
markdown_includes:
  nav: "kubernetes/docs/shared/_side-navigation.md"
context:
  title: "Using GPU workers"
  description: How to run workloads with GPU support.
keywords: gpu, nvidia, cuda
tags: [operating]
sidebar: k8smain-sidebar
permalink: gpu-workers.html
layout: [base, ubuntu-com]
toc: False
---

**Charmed Kubernetes** supports GPU-enabled
instances for applications which can use them. The kubernetes-worker charm will
automatically detect NVIDIA hardware and enable the appropriate support.
However, the implementation of GPU-enabled instances differs greatly between
public clouds. This page outlines the recommended methods for running GPU
enabled hardware for different public clouds.

### Deploying Charmed Kubernetes with GPU workers on AWS

If you are installing Charmed Kubernetes using a bundle, you can use constraints to specify
that the worker units are deployed on GPU-enabled machines. Because GPU support
varies considerably depending on the underlying cloud, this requires specifying
a particular instance type.

This can be done with a YAML overlay file. For example, when deploying Charmed
Kubernetes on AWS, you may decide you wish to use AWS's 'p3.2xlarge' instances
(you can check the AWS instance definitions on the
[AWS website][aws-instance]). NVIDIA also updates its list of supported GPUs
frequently, so be sure to look at [NVIDIA GPU support docs][nvidia-gpu-support] 
before installing on a specific AWS instance.


A YAML overlay file can be constructed like this:

```yaml
#gpu-overlay.yaml
applications:
  kubernetes-worker:
    constraints: instance-type=p3.2xlarge
```

And then deployed with Charmed Kubernetes like this:

```bash
juju deploy charmed-kubernetes --overlay ~/path/aws-overlay.yaml --overlay ~/path/gpu-overlay.yaml
```

As demonstrated here, you can use multiple overlay files when deploying, so you
can combine GPU support with an integrator charm or other custom configuration.

You may then want to [test a GPU workload](#test)

### Adding GPU workers with AWS

It isn't necessary for all the worker units to have GPU support. You can simply
add GPU-enabled workers to a running cluster. The recommended way to do this is
to first create a new constraint for the kubernetes-worker:

```bash
juju set-constraints kubernetes-worker instance-type=p3.2xlarge
```

Then you can add as many new worker units as required. For example, to add two
new units.

```bash
juju add-unit kubernetes-worker -n2
```

### Adding GPU workers with GCP

Google supports GPUs slightly differently to most clouds. There are no GPUs
included in any of the default instance templates, and therefore they have
to be added manually.

To begin, add a new machine with Juju. Include any desired constraints for
memory,cores,etc :

```bash
juju add-machine --constraints cores=2
```

The command will return, telling you the number of the machine that was
created - keep a note of this number.

Next you will need to use the gcloud tool or the GCP console to stop the
instance, edit its configuration and then restart the machine.

Once it is up and running, you can then add it as a worker:

```bash
juju add-unit kubernetes-worker --to 10
```

...replacing '10' in the above with the number of the machine you created.

As the charm installs, the GPU will be detected and the relevant drivers will
also be installed.
<a  id="test"> </a>

## Testing

As GPU instances can be costly, it is useful to test that they can actually be
used. A simple test job can be created to run NVIDIA's hardware reporting tool.
Please note that you may need to replace the image tag in the following
YAML with [the latest supported one][nvidia-supported-tags].

This can also be [downloaded here][asset-nvidia].

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nvidia-smi
spec:
  template:
    metadata:
      name: nvidia-smi
    spec:
      restartPolicy: Never
      containers:
      - image: nvidia/cuda:11.6.0-base-ubuntu20.04
        name: nvidia-smi
        args:
          - nvidia-smi
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            nvidia.com/gpu: 1
        volumeMounts:
        - mountPath: /usr/bin/
          name: binaries
        - mountPath: /usr/lib/x86_64-linux-gnu
          name: libraries
      volumes:
      - name: binaries
        hostPath:
          path: /usr/bin/
      - name: libraries
        hostPath:
          path: /usr/lib/x86_64-linux-gnu

```

Download the file and run it with:

```bash
kubectl create -f nvidia-test.yaml
```

You can inspect the logs to find the hardware report.

```bash
kubectl logs job.batch/nvidia-smi

Thu Mar  3 14:52:26 2022       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 510.47.03    Driver Version: 510.47.03    CUDA Version: 11.6     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla V100-SXM2...  On   | 00000000:00:1E.0 Off |                    0 |
| N/A   39C    P0    24W / 300W |      0MiB / 16384MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

<!-- LINKS -->
[asset-nvidia]: https://raw.githubusercontent.com/juju-solutions/kubernetes-docs/main/assets/nvidia-test.yaml
[nvidia-supported-tags]: https://gitlab.com/nvidia/container-images/cuda/blob/master/doc/README.md#supported-tags
[quickstart]: /kubernetes/docs/quickstart
[aws-instance]: https://aws.amazon.com/ec2/instance-types/
[azure-instance]: https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes-gpu
[nvidia-gpu-support]: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/platform-support.html#supported-nvidia-gpus-systems

<!-- FEEDBACK -->
<div class="p-notification--information">
  <div class="p-notification__content">
    <p class="p-notification__message">We appreciate your feedback on the documentation. You can
    <a href="https://github.com/charmed-kubernetes/kubernetes-docs/edit/main/pages/k8s/gpu-workers.md" >edit this page</a>
    or
    <a href="https://github.com/charmed-kubernetes/kubernetes-docs/issues/new" >file a bug here</a>.</p>
  </div>
</div>

