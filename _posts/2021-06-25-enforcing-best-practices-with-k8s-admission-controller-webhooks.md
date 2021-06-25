---
layout: post
title:  "Enforcing best practices automatically – short tale about Kubernetes Admission Validation Webhooks"
date:   2021-06-25 16:12:11 +0200
author: "Paweł Bojanowski"
categories: kubernetes devops automation golang
---


# Enforcing best practices automatically – short tale about Kubernetes Admission Validation Webhooks


Administering Kubernetes cluster may become a tricky task over time. Imagine that your engineering team is growing rapidly, and not every new engineer has significant Kubernetes expertise. Your team was asked to ensure that everyone in org are following best practices for configuring and managing their Kubernetes workloads.

I have good news for you: you can do this automatically without writing a lot of boilerplate code. Kubernetes provides a concept of Admission Webhooks. What are admission webhooks? Let’s quote official documentation:

> Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, validating admission webhook and mutating admission webhook. Mutating admission webhooks are invoked first, and can modify objects sent to the API server to enforce custom defaults. After all object modifications are complete, and after the incoming object is validated by the API server, validating admission webhooks are invoked and can reject requests to enforce custom policies.


In this article I will focus on the validation admission webhooks, which can effectively prevent cluster users from not following standards that we established. For showing purposes, I decided to create a validating webhook which checks whether [recommended set of labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/).
is set on Pod objects.

## Let’s get started.

Validating function should process v1.Pod object, and return boolean answer for “is this pod spec valid one?” question. Short description of what’s missing would be nice to have as well.
Here’s a function:
```go
func hasRecommendedLabels(pod *v1.Pod) (bool, string) {
	recommendedLabels := getRecommendedLabels()
	var msg string
	for _, labelKey := range recommendedLabels {
		_, ok := pod.Labels[labelKey]
		if !ok {
			msg += fmt.Sprintf("%s,", labelKey)
		}
	}
	if msg != "" {
		return false, "Missing labels: " + msg
	}
	return true, ""
}
```


As you probably suspect, we will need to set up HTTP server which has an endpoint accepting and responding in JSON. This endpoint will receive AdmisionReview object in JSON, so our responsibility is to unmarshal it into a struct, check whether all needed labels are set and respond with correct boolean under allowed key.

For reference, [here’s kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#webhook-request-and-response) about expected request and response payloads. 
To avoid writing a lot of boilerplate code, we are going to use excellent [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/webhook) package, which would do all heavy lifting for us.

Our responsibility would be to write only our validating logic. We are going to implement Handler interface which looks like this:
```go
// Handler can handle an AdmissionRequest.
type Handler interface {
	// Handle yields a response to an AdmissionRequest.
	//
	// The supplied context is extracted from the received http.Request, allowing wrapping
	// http.Handlers to inject values into and control cancelation of downstream request processing.
	Handle(context.Context, Request) Response
}
```

Our Handler implementation is really short and concise:
```go
type ValidationHandler struct{}

func (mh *ValidationHandler) Handle(ctx context.Context, req admission.Request) admission.Response {
	var pod v1.Pod
	if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
		glog.Errorf("failed to unmarshal raw pod object: %v", err)
		return admission.Response{
			AdmissionResponse: v1beta1.AdmissionResponse{
				Result: &metav1.Status{
					Message: err.Error(),
				},
			},
		}
	}
	if isAllowed, msg := hasRecommendedLabels(&pod); !isAllowed {
		return admission.Response{
			AdmissionResponse: v1beta1.AdmissionResponse{
				Allowed: false,
				Result: &metav1.Status{
					Message: msg,
				},
			},
		}
	}
	return admission.Response{
		AdmissionResponse: v1beta1.AdmissionResponse{
			Allowed: true,
		},
	}
}
```

# Adding webhook configuration

Once we have our binary ready, we need to run it inside our Kubernetes cluster. Typically, webhook server is running as Deployment:
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-labels-validation-webhook
  namespace: cert-manager
  labels:
    app: k8s-labels-validation-webhook
spec:
  selector:
    matchLabels:
      app: k8s-labels-validation-webhook
  template:
    metadata:
      labels:
        app: k8s-labels-validation-webhook
    spec:
      containers:
        - name: k8s-labels-validation-webhook
          image: hidalgopl/k8s-labels-validation-webhook:latest
          args:
            - -cert-dir=/etc/webhook/certs
            - -port=8443
            - -v=5
          volumeMounts:
            - name: webhook-certs
              mountPath: /etc/webhook/certs
              readOnly: true
      volumes:
        - name: webhook-certs
          secret:
            secretName: k8s-labels-validation-webhook-certs

```

as there isn't anything unusual in the deployment definition, let's move on. Once the deployment is running, we will need Kubernetes Service:
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: k8s-labels-validation-webhook
  namespace: cert-manager
  labels:
    app: k8s-labels-validation-webhook
spec:
  ports:
    - port: 8443
      targetPort: 8443
  selector:
    app: k8s-labels-validation-webhook
```

now, to inform Kubernetes Control plane about webhook server, we need to create object: `ValidatingWebhookConfiguration`
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: k8s-labels-validation-webhook
  namespace: cert-manager
  annotations:
    cert-manager.io/inject-ca-from: cert-manager/selfsigned-cert # this asks cert-manager for CA injection
webhooks:
  - clientConfig:
      caBundle: Cg==
      service:
        name: k8s-labels-validation-webhook
        path: /validate # this is server endpoint we have defined in webhook server app
        port: 8443 # port matches the one we specified in Service
        namespace: cert-manager
    sideEffects: None
    admissionReviewVersions: ["v1beta1"]
    failurePolicy: Fail
    name: recommendedlabels.elotl.io 
    rules: # those rules specify when webhook should be called
      - operations: [ "CREATE" ]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
```
After this object is created, we can test it, simply by creating a pod. As we specified above, after Pod is created, kubernetes will send Pod object to `k8s-labels-validation-webhook:8443/validate`. Then, our app with verify whether the pod has needed labels.

Applying pod without set labels should result with similar output:
```
$ kubectl apply -f pod_test.yaml
Error from server: error when creating "pod_test.yaml": admission webhook "recommendedlabels.elotl.io" denied the request: Missing labels: app.kubernetes.io/name,app.kubernetes.io/instance,app.kubernetes.io/version,app.kubernetes.io/component,app.kubernetes.io/part-of,app.kubernetes.io/managed-by,app.kubernetes.io/created-by,
```


## Summary
Admission controller webhooks are simple yet powerful way to maintain your cluster workloads and enhance organization-wide practices. You can use it to ensure that workloads are labeled properly, have resource requests and limits set, etc. 
There is also another class of admission controller webhooks, mutating ones, which allow you to modify objects Spec. Example use case of those could be injecting sidecar containers to every Pod.
What I really like about admission controller webhooks is that they don't require much effort to create, but they can provide a lot of value for your engineering teams.

Code used in this article can be seen in [my github](https://github.com/hidalgopl/k8s-labels-validation-webhook).