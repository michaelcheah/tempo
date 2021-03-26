# Basic Tempo Example

This notebook will walk you through an end-to-end example deploying a Tempo pipeline, running on its own Conda environment.

## Prerequisites

  * rclone and conda installed.
  * Run this notebook within the `seldon-examples` conda environment. Details to create this can be found [here]().

## Architecture

We will show two Iris dataset prediction models combined with service orchestration that shows some arbitrary python code to control predictions from the two models.

![architecture](architecture.png)


```python
from IPython.core.magic import register_line_cell_magic

@register_line_cell_magic
def writetemplate(line, cell):
    with open(line, 'w') as f:
        f.write(cell.format(**globals()))
```

## Train Iris Models

We will train:

  * A sklearn logistic regression model
  * A xgboost model


```python
from sklearn import datasets
from sklearn.linear_model import LogisticRegression
import joblib
iris = datasets.load_iris()
X = iris.data  # we only take the first two features.
y = iris.target
logreg = LogisticRegression(C=1e5)
logreg.fit(X, y)
logreg.predict_proba(X[0:1])
with open("./artifacts/sklearn/model.joblib","wb") as f:
    joblib.dump(logreg, f)
```


```python
from xgboost import XGBClassifier
clf = XGBClassifier()
clf.fit(X,y)
clf.save_model("./artifacts/xgboost/model.bst")
```

## Defining pipeline

The first step will be to define our custom pipeline.
This pipeline will access 2 models, stored remotely. 


```python
import os
import numpy as np
from typing import Tuple
from tempo.serve.metadata import ModelFramework, KubernetesOptions
from tempo.serve.model import Model
from tempo.seldon.docker import SeldonDockerRuntime
from tempo.kfserving.protocol import KFServingV2Protocol
from tempo.serve.utils import pipeline, predictmethod
from tempo.seldon.k8s import SeldonKubernetesRuntime
from tempo.serve.utils import pipeline

import logging
logging.basicConfig(level=logging.INFO)

SKLEARN_FOLDER = os.getcwd()+"/artifacts/sklearn"
XGBOOST_FOLDER = os.getcwd()+"/artifacts/xgboost"
PIPELINE_ARTIFACTS_FOLDER = os.getcwd()+"/artifacts/classifier"

docker_runtime = SeldonDockerRuntime()

sklearn_model = Model(
        name="test-iris-sklearn",
        runtime=docker_runtime,
        platform=ModelFramework.SKLearn,
        local_folder=SKLEARN_FOLDER,
        uri="s3://tempo/basic/sklearn"
)

xgboost_model = Model(
        name="test-iris-xgboost",
        runtime=docker_runtime,
        platform=ModelFramework.XGBoost,
        local_folder=XGBOOST_FOLDER,
        uri="s3://tempo/basic/xgboost"
)

docker_runtime_v2 = SeldonDockerRuntime(protocol=KFServingV2Protocol())

@pipeline(name="classifier",
          runtime=docker_runtime_v2,
          uri="s3://tempo/basic/pipeline",
          local_folder=PIPELINE_ARTIFACTS_FOLDER,
          models=[sklearn_model, xgboost_model])
def classifier(payload: np.ndarray) -> Tuple[np.ndarray,str]:
    res1 = sklearn_model(payload)

    if res1[0][0] > 0.5:
        return res1,"sklearn prediction"
    else:
        return xgboost_model(payload),"xgboost prediction"
```

## Deploying pipeline to Docker

The next step, will be to deploy our pipeline to Docker.
We will divide this process into 2 steps:

1. Save our artifacts and environment
2. Deploy resources

### Saving artifacts

We provide a conda yaml in out `local_folder` which tempo will use as the runtime environment to save. If this file was not there it would save the current conda environment. One can also provide a named conda environment with `conda_env` in the decorator.


```python
%%writefile artifacts/classifier/conda.yaml
name: tempo
channels:
  - defaults
dependencies:
  - _libgcc_mutex=0.1=main
  - ca-certificates=2021.1.19=h06a4308_0
  - certifi=2020.12.5=py37h06a4308_0
  - ld_impl_linux-64=2.33.1=h53a641e_7
  - libedit=3.1.20191231=h14c3975_1
  - libffi=3.3=he6710b0_2
  - libgcc-ng=9.1.0=hdf63c60_0
  - libstdcxx-ng=9.1.0=hdf63c60_0
  - ncurses=6.2=he6710b0_1
  - openssl=1.1.1j=h27cfd23_0
  - pip=21.0.1=py37h06a4308_0
  - python=3.7.9=h7579374_0
  - readline=8.1=h27cfd23_0
  - setuptools=52.0.0=py37h06a4308_0
  - sqlite=3.33.0=h62c20be_0
  - tk=8.6.10=hbc83047_0
  - wheel=0.36.2=pyhd3eb1b0_0
  - xz=5.2.5=h7b6447c_0
  - zlib=1.2.11=h7b6447c_3
  - pip:
    - mlops-tempo
    - mlserver==0.3.1.dev5
    - mlserver-tempo==0.3.1.dev5
```


```python
classifier.save(save_env=True)
```

### Deploying pipeline


```python
classifier.deploy()
classifier.wait_ready()
```

### Sending requests

We can send requests to the deployed components with either the python code for the classifier running locally or remotely. 

First we test calling the classifer locally. It will call out to the remote models running in Docker.


```python
classifier(payload=np.array([[1, 2, 3, 4]]))
```

Now we can use the `.remote` method to call to the remote classifier running in Docker.


```python
classifier.remote(payload=np.array([[1, 2, 3, 4]]))
```


```python
classifier.remote(payload=np.array([[5.964,4.006,2.081,1.031]]))
```

### Undeploy pipeline


```python
classifier.undeploy()
```

## Deploying pipeline to K8s

The next step, will be to deploy our pipeline to Kubernetes.
We will divide this process into 3 sub-steps:

1. Save our artifacts and environment
2. Upload to remote storage
3. Deploy resources

### Setup Namespace


```python
!kubectl create namespace production
```


```python
!kubectl apply -f ../../../k8s/tempo-pipeline-rbac.yaml -n production
```


```python
%%writefile minio-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: minio-secret
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: minioadmin
  AWS_SECRET_ACCESS_KEY: minioadmin
  AWS_ENDPOINT_URL: http://minio.minio-system.svc.cluster.local:9000
  USE_SSL: "false"
```


```python
!kubectl apply -f minio-secret.yaml -n production
```

### Change runtime


```python
k8s_options = KubernetesOptions(namespace="production",authSecretName="minio-secret")

k8s_runtime = SeldonKubernetesRuntime(k8s_options=k8s_options)
sklearn_model.set_runtime(k8s_runtime)
xgboost_model.set_runtime(k8s_runtime)

k8s_runtime_v2 = SeldonKubernetesRuntime(k8s_options=k8s_options, protocol=KFServingV2Protocol())
classifier.set_runtime(k8s_runtime_v2)
```

### Saving artifacts

Just save the artfacts and not the environment as well.


```python
classifier.save(save_env=False)
```

### Uploading artifacts


```python
MINIO_IP=!kubectl get svc minio -n minio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
MINIO_IP=MINIO_IP[0]
```


```python
%%writetemplate rclone.conf
[s3]
type = s3
provider = minio
env_auth = false
access_key_id = minioadmin
secret_access_key = minioadmin
endpoint = http://{MINIO_IP}:9000

```


```python
import os
from tempo.conf import settings
settings.rclone_cfg = os.getcwd() + "/rclone.conf"
```


```python
sklearn_model.upload()
xgboost_model.upload()
classifier.upload()
```

### Deploy


```python
classifier.deploy()
classifier.wait_ready()
```

### Sending requests

Lastly, we can now send requests to our deployed pipeline.
For this, we will leverage the `remote()` method, which will interact without our deployed pipeline (as opposed to executing our pipeline's code locally).


```python
classifier.remote(payload=np.array([[1, 2, 3, 4]]))
```

### Undeploy pipeline


```python
classifier.undeploy()
```


```python

```