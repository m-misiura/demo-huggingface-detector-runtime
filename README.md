# Demo: guardrails-detector-huggingface-runtime

The [guardrails-detector-huggingface-runtime](https://github.com/trustyai-explainability/guardrails-detectors/tree/main/detectors/huggingface) provides a way to deploy guardrails detection models from [Hugging Face](https://huggingface.co/models) via the existing model serving stack. This allows models like ibm-granite/granite-guardian-hap-38m to be used within TrustyAI Guardrails Ecosystem. This serving runtime should be compatible with most [AutoModelForSequenceClassification](https://huggingface.co/docs/transformers/model_doc/auto#transformers.AutoModelForSequenceClassification)

This serving runtime exposes `api/v1/text/contents` endpoint of the [Detectors API](https://foundation-model-stack.github.io/fms-guardrails-orchestrator/?urls.primaryName=Detector+API#/Text/text_content_analysis_unary_handler)

## Getting Started

In this demo, [ibm-granite/granite-guardian-hap-38m](https://huggingface.co/ibm-granite/granite-guardian-hap-38m) is served using the aforementioned runtime

1. Create a namespace and navigate to it

```bash
oc create ns ${NAMESPACE}
oc project ${NAMESPACE}
```

where `${NAMESPACE}` is the name of the namespace you want to use. You can also use `oc new-project ${NAMESPACE}` to create and switch to the new namespace in one command.

2. Deploy the model's storage container:

```bash
oc apply -f resources/model_storage_container.yaml
```

3. Deploy the model's serving runtime:

```bash
oc apply -f resources/serving_runtime.yaml
```

4. Create an inference service:

```bash
oc apply -f resources/isvc.yaml
```

Within the namespace, you should see [the following pods](images/hap_pods.png): 

1. One pod for the model storage container
2. One pod for the model's inference service

5. Create a route to access the inference service:

```bash
oc apply -f resources/route.yaml
```

6. Test the inference service by hitting the `/health` endpoint:

```bash
HAP_ROUTE=$(oc get routes hap-route -o jsonpath='{.spec.host}')
curl -s http://$HAP_ROUTE/health | jq
```

which should yield `"ok"`

7. Test the inference service by hitting the `api/v1/text/contents` endpoint with sample inputs:

```bash
curl -s -X POST \
  "http://$HAP_ROUTE/api/v1/text/contents" \
  -H 'accept: application/json' \
  -H 'detector-id: hap' \
  -H 'Content-Type: application/json' \
  -d '{
    "contents": ["You dotard, I really hate this stuff", "I simply love this stuff"],
    "detector_params": {}
  }' | jq
```

which should return

```
[
  [
    {
      "start": 0,
      "end": 36,
      "detection": "sequence_classifier",
      "detection_type": "sequence_classification",
      "score": 0.9634237885475159,
      "sequence_classification": "LABEL_1",
      "sequence_probability": 0.9634237885475159,
      "token_classifications": null,
      "token_probabilities": null,
      "text": "You dotard, I really hate this stuff",
      "evidences": []
    }
  ],
  [
    {
      "start": 0,
      "end": 24,
      "detection": "sequence_classifier",
      "detection_type": "sequence_classification",
      "score": 0.00016678044630680233,
      "sequence_classification": "LABEL_0",
      "sequence_probability": 0.00016678044630680233,
      "token_classifications": null,
      "token_probabilities": null,
      "text": "I simply love this stuff",
      "evidences": []
    }
  ]
]
```

The first input is classified as `LABEL_1` with a score of `0.9634237885475159`, indicating that the model detected hate speech. The second input is classified as `LABEL_0` with a score of `0.00016678044630680233`, indicating that the model did not detect hate speech.

## Changing the model

If you would like to try this serving runtime with a different model, the following steps would be required 
- Download a different model from Hugging Face, for example see this configuration file to [download a different model](https://github.com/trustyai-explainability/reference/blob/main/guardrails/end-to-end/text_contents/llm-vllm/http/suicide_model_container.yaml)
- In the `resources/isvc.yaml`: 
    - change `key` and `path` in the `storage` section

## Pre-requisites

The following operators are available on the cluster:

- Authorino (0.16.0)
- NVIDIA GPU (24.9.2)
- Node Feature Discovery Operator (4.16.0)
- Open Data Hub (2.21.0)
- Red Hat Opneshift Serverless (1.35.0)
- Red Hat Service Mesh (2.6.5-0)

The Open Data Hub, can be initialised with the default DSC Initialization and Data Science Cluster; the cluster had one `g4dn-xlarge`.

