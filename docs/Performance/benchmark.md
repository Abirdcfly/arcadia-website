---
sidebar_position: 1
title: Benchmark
sidebar_label: Benchmark
---

## Test the performance of the model

### Prepare data

Downloading the ShareGPT dataset
```
wget https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json
```

### Start the model service
Start the model service to be benchmarked, and get the model id.

### Run benchmark script
Get the benchmark script from vllm github below:
```
https://github.com/vllm-project/vllm/tree/main/benchmarks
```
Run benchmark_serving script to do benchmark.
* fastchat will generate a random model id when deploy the model and it's not the same as the --model parameter here. We have to use local model path for benchmark_serving's --model, and when call the API, wen should use the id of fastchat's model service. To achieve this, we can update the ```backend_request_func.py``` script in line 222, change ```request_func_input.prompt``` to the generated one by fastchat. Or we can add a new parameter in benchmark_serving to specify this model id.

```shell
# we use vllm as the backend, host and port will be the base url of the model service
# benchmark script will run against its /v1/completions endpoint
python benchmark_serving.py --backend vllm --model /ray/models/qwen1.5-7b-chat --dataset ShareGPT_V3_unfiltered_cleaned_split.json --num-prompts 100 --request-rate 10 --host 10.96.201.197 --port 8000
```
Here is a sample output
```
Namespace(backend='vllm', base_url=None, host='10.96.201.197', port=8000, endpoint='/v1/completions', dataset='ShareGPT_V3_unfiltered_cleaned_split.json', dataset_name='sharegpt', dataset_path=None, model='/ray/models/qwen1.5-7b-chat', tokenizer=None, best_of=1, use_beam_search=False, num_prompts=1000, sonnet_input_len=550, sonnet_output_len=150, sonnet_prefix_len=200, request_rate=20.0, seed=0, trust_remote_code=False, disable_tqdm=False, save_result=False, metadata=None, result_dir=None)
Special tokens have been added in the vocabulary, make sure the associated word embeddings are fine-tuned or trained.
/home/ray/benchmark_serving.py:590: UserWarning: The '--dataset' argument will be deprecated in the next release. Please use '--dataset-name' and '--dataset-path' in the future runs.
  main(args)
Traffic request rate: 20.0
100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1000/1000 [03:01<00:00,  5.50it/s]
============ Serving Benchmark Result ============
Successful requests:                     863       
Benchmark duration (s):                  181.98    
Total input tokens:                      186901    
Total generated tokens:                  75378     
Request throughput (req/s):              4.74      
Input token throughput (tok/s):          1027.06   
Output token throughput (tok/s):         414.22    
---------------Time to First Token----------------
Mean TTFT (ms):                          21203.99  
Median TTFT (ms):                        0.00      
P99 TTFT (ms):                           103100.89 
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          143.82    
Median TPOT (ms):                        97.03     
P99 TPOT (ms):                           601.67    
==================================================
```

You can use ```python benchmark_serving.py --help``` to get all paramaeters that can be used to run the benchmark script.
