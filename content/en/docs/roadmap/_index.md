---
title: "Roadmap"
weight: 5
description: >
  The roadmap for future releases
---

## 2023

- Support intelligent machine learning replicas prediction algorithm.
- Support algorithm which detects abnormal traffic or potential capacity risks, and suggests a safe replica count proactively.
- Support custom portrait verification which further controls the rules of the algorithm output to mitigates risks.
- Support automatic risk detection and mitigation during autoscaling.

## Future

- Fully support IHPA extension framework which enable users to custom or extend the behaviors/functions of IHPA without hacking into the project.
- Introduce Kapacity Agent, which supports Pod Standby state switching, Pod health scoring, etc. This can further enhance the capability of multi-stage scaling, and also serve as a basic function of colocation.
- Introduce Kapacity Scheduler, which supports dynamic scheduling based on realtime Pod and Node resource usage to improve resource utilization as well as mitigate hotspot problems, and supports more advanced scheduling strategies.
- Support on-demand batch switching of Pod Online and Standby states to support time-sharing scheduling.
- Support recommendation of Pod resource specifications (CPU, memory, etc.) through intelligent algorithms, and support dynamic adjustment of Pod resource specifications through VPA.
- Introduce console UI, and support multi-dimensional cost and carbon emission calculation.
