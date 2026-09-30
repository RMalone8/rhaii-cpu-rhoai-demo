# RHAII CPU model on a shared PVC

The download Job writes `Qwen/Qwen2.5-1.5B-Instruct` to the `rhaii-cache` PVC. The regular Deployment loads `/models/Qwen2.5-1.5B-Instruct` in offline mode. The KServe InferenceService reads the same directory through `pvc://rhaii-cache/Qwen2.5-1.5B-Instruct`.

Apply the namespace manifest first. The download Job needs access to Hugging Face and the `redhat-registry-pull` secret in the namespace. Apply the PVC, wait for the download, and apply either or both serving manifests:

```sh
oc apply -f preflight/namespace.yaml
oc apply -f deployment/rhaii-cpu-cache-pvc.yaml
oc apply -f preflight/rhaii-cpu-model-download-job.yaml
oc -n ai-inference-cpu-demo wait --for=condition=complete job/rhaii-cpu-model-download --timeout=60m
oc apply -f preflight/rhaii-cpu-deployment.yaml
oc apply -f kserve/rhaii-cpu-inferenceservice.yaml
```

The Job uses the same RHAII image as the regular Deployment. It uses `hf-secret` if present; the public model does not require it. It leaves a completion marker in the model directory and skips an already completed download. To download a different revision, use a new model directory and update both serving manifests.

The PVC currently uses `ReadWriteOnce`. Both serving pods can mount it together only when scheduled on the same node. Use a `ReadWriteMany` storage class if they must run on different nodes at the same time.
