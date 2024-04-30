---
sidebar_position: 1
title: Deployment Approach
sidebar_label: Deployment Approach
---

For distributed deployment of LLM inference, there are two approaches normally:

1. Start multiple inference instances that each is running on isolated GPU, and use LB like Nginx for load balancing between these instances.

2. Start a single instance and use tensor_parallel_size>1, and distribute the model weights across multiple GPUs in the same group

Here is the guidance: If the model is too large to fit in a single GPU, we have to use tensor_parallel_size and use multiple GPUs to load the model. If not, start multiple instances on isolated GPUs should have better performance. And using tensor_parallel_size>1 requires communication between GPUs which is quite expensive and complicated, and probably has bad performance.
