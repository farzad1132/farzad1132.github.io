---
title: Roshanfer
excerpt: "Avoiding SLO violations with proactive overload control in microservices"
image: /images/projects/roshanfer.png
image_alt: Roshanfer
github: https://github.com/roshanfer-project
# pdf: /files/roshanfer.pdf
---

***Accepted to EuroSys'27***

Dynamic and unpredictable load affects the performance resilience of cloud services. Frequent and sporadic overloads cause performance degradation that manifests as increased latency, SLO violations, and reduced goodput. Despite the plethora of mechanisms to deal with overload, ranging from auto-scaling and service degradation to careful admission control and load shedding, microservice-based deployments remain vulnerable to such problems. Existing overload control frameworks depend on slow, reactive designs, and fail to cater to the needs of modern microservices with dynamic call graphs and changing workloads.
We present Roshanfer, a proactive overload control system for microservice deployments. With Roshanfer, microservices are overload-proof by design because Roshanfer proactively controls and strictly bounds the number of admitted requests. To do so, it depends on a novel mechanism that creates closed systems between microservice pairs. This design allows fast backpressure and request queue buildup only at the service gateway, which is the centralized point for admission control and policy enforcement. To our knowledge, Roshanfer is the first overload control mechanism to deal with dynamic call graphs at a request granularity, and it is also supported by a robust TLA+ model that provides further correctness guarantees.