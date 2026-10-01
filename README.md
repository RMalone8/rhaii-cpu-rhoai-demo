# RHAII CPU model on a shared PVC

The download Job writes `Qwen/Qwen2.5-1.5B-Instruct` to the `model-cache` PVC. The regular Deployment loads `/models/Qwen2.5-1.5B-Instruct`. The KServe InferenceService reads the same directory through `pvc://model-cache/Qwen2.5-1.5B-Instruct`.

Apply the namespace manifest first. The download Job needs access to Hugging Face and the `redhat-registry-pull` secret in the namespace. Apply the PVC, wait for the download, and apply either or both serving manifests:

```sh
oc apply -f preflight/namespace.yaml
oc apply -f preflight/model-cache-pvc.yaml
oc apply -f preflight/model-download-job.yaml
oc -n ai-inference-cpu-demo wait --for=condition=complete job/model-download-job --timeout=60m
oc apply -f deployment/cpu-deployment.yaml
oc apply -f kserve/cpu-inferenceservice.yaml
```

The Job uses the same RHAII image as the regular Deployment. It uses `hf-secret` if present; the public model does not require it. `hf download` keeps metadata in the model directory and reuses files that are already current. The shorter Job no longer writes a completion marker or stages the download before making the directory visible. Wait for the Job to complete before starting either serving manifest. To download a different revision, use a new model directory and update both serving manifests.

The KServe manifest sets `VLLM_CPU_KVCACHE_SPACE=8`, matching the regular Deployment's 8 GiB CPU KV cache. Apply the InferenceService manifest again after changing it and check its status with `oc -n ai-inference-cpu-demo describe inferenceservice qwen25-instruct`.

If the Job already exists and needs a Pod template change, delete the Job and apply the updated manifest again:

```sh
oc -n ai-inference-cpu-demo delete job model-download-job
oc apply -f preflight/model-download-job.yaml
```

The PVC currently uses `ReadWriteOnce`. Both serving pods can mount it together only when scheduled on the same node. Use a `ReadWriteMany` storage class if they must run on different nodes at the same time.
