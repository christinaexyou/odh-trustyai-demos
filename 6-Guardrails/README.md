# Guardrailing LLMs via TrustyAI in ODH

This demo will explore how to use TrustyAI to moderate LLM output content.

This demo leverages KServe RawDeployment for model deployment; for a guide on how to install KServe RawDeployment on Open Data Hub, refer to this installation guide.

## Context
GuardrailsOrchestrator is a service to moderate LLM input and output content. It is based on the upstream open-source project, [FMS Guardrails Orchestrator](https://github.com/foundation-model-stack/fms-guardrails-orchestrator).

## Setup
At the time of this publication, GuardrailsOrchestrator is only available on TrustyAI Service Operator's `dev/guardrails-orch-merge` branch. In order to use it on Open Data Hub, add the following devFlag to your DataScienceCluster (DSC) resource:
```
trustyai:
    devFlags:
        manifests:
            - contextDir: config
        sourcePath: ''
        uri: https://github.com/trustyai-explainability/trustyai-service-operator/tarball/dev/guardrails-orch-merge
    managementState: Managed
```

## Deploy LLM
1. Navigate to the `model-namespace` created in the setup section: `oc project model-namespace`
2. Deploy the LLM's storage container: `oc apply -f resources/llm_storage_container.yaml`
3. Deploy the vLLM serving runtime: `oc apply -f resources/vllm.yaml`
4. Deploy the LLM: `oc apply -f resources/llm.yaml`

## Deploy a GuardrailsOrchestrator instance
1. Deploy the TrustyAI Service configmap: `oc apply -f resources/trustyai_configmap.yaml`
2. Deploy the Guardrails Orchestrator configmap: `oc apply -f guardrails_configmap.yaml`
3. Deploy the vLLM Gateway configmap: `oc apply -f vllm_gateway_configmap.yaml`
4. Deploy the Guardrails Orchestrator custom resource: `oc apply -f guardrails_crd.yaml`
5. From the Openshift Console, navigate to the `model-namespace` project and look at the Workloads -> Pods screen. You should see the following pods:

    Optionally, run `oc get pods`:

6. To sanity-check the health of your LLM and detector services, run the following command to access the orchestrator container within the orchestrator pod:
    ```
    export POD_NAME=gorch-test-55bf5f84d9-dd4vm
    export CONTAINER_NAME=gorch-test
    oc exec -it -n $TEST_NS $POD_NAME -c $CONTAINER_NAME -- /bin/bash
    ```

    a) Query the `/health` endpoint: `curl -v http://localhost:8034/health`

    Expected output:
    ```
    *   Trying ::1:8034...
    * connect to ::1 port 8034 failed: Connection refused
    *   Trying 127.0.0.1:8034...
    * Connected to localhost (127.0.0.1) port 8034 (#0)
    > GET /health HTTP/1.1
    > Host: localhost:8034
    > User-Agent: curl/7.76.1
    > Accept: */*
    >
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    < content-type: application/json
    < content-length: 36
    < date: Fri, 31 Jan 2025 14:04:25 GMT
    <
    * Connection #0 to host localhost left intact
    {"fms-guardrails-orchestr8":"0.1.0"}
    ```

    b) Query the `/info` endpoint: `curl -v http://localhost:8034/info`

    Expected output:
    ```
        *   Trying ::1:8034...
    * connect to ::1 port 8034 failed: Connection refused
    *   Trying 127.0.0.1:8034...
    * Connected to localhost (127.0.0.1) port 8034 (#0)
    > GET /info HTTP/1.1
    > Host: localhost:8034
    > User-Agent: curl/7.76.1
    > Accept: */*
    >
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    < content-type: application/json
    < content-length: 82
    < date: Fri, 31 Jan 2025 14:05:10 GMT
    <
    * Connection #0 to host localhost left intact
    {"services":{"chat_generation":{"status":"HEALTHY"},"regex":{"status":"HEALTHY"}}}
    ```

7. Exit out of the orchestrator container

## Guardrailing LLM Input Prompts
1. Find the route to the GuardrailsOrchestrator instance: `GORCH_ROUTE=https://$(oc get route/ 00 --template={{.spec.host}})`

2.  Query the `/pii` endpoint to perform completions while enabling detections on email and SSN information
    ```
    curl $GORCH_ROUTE/pii \
    -H "Content-Type: application/json" \
    -d '{
        "model": "llm",
        "messages": [
            {
                "role": "user",
                "content": "say hello to me at someemail@somedomain.com"
            },
            {
                "role": "user",
                "content": "btw here is my social 123456789"
            }
        ]
    }'
    ```
The payload is structured as follows:
* `model`: The name of the model to query
* `messages`: Input prompts

This request will return the following output:
```
Object {
        "choices": Array [],
        "created": Number(1738705923),
        "detections": Object {
            "input": Array [
                Object {
                    "message_index": Number(1),
                    "results": Array [
                        Object {
                            "detection": String("SocialSecurity"),
                            "detection_type": String("pii"),
                            "detector_id": String("regex"),
                            "end": Number(31),
                            "score": Number(1.0),
                            "start": Number(22),
                            "text": String("123456789"),
                        },
                    ],
                },
            ],
        },
        "id": String("71b080689abf47099c7fb5424aced478"),
        "model": String("llm"),
        "object": String(""),
        "usage": Object {
            "completion_tokens": Number(0),
            "prompt_tokens": Number(0),
            "total_tokens": Number(0),
        },
        "warnings": Array [
            Object {
                "message": String("Unsuitable input detected. Please check the detected entities on your input and try again with the unsuitable input removed."),
                "type": String("UNSUITABLE_INPUT"),
            },
        ],
    }
```

