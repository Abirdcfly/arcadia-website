---
sidebar_position: 3
title: Model APIs
sidebar_label: Model APIs
---
You can use APIs to communicate with local model service.

```
curl -v http://fastchat.172.40.20.125.nip.io/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"<model-id>","messages":[{"role":"user","content":"hello world."}],"temperature":0.1,"top_p":1,"max_tokens":2048,"stream":false,"safe_prompt":false,"random_seed":null}'
```

