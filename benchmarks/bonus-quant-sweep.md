# Bonus — Quantization sweep

Tier: `Qwen2.5-1.5B-Instruct`  ·  threads: `4`  ·  n_gpu_layers: `99`

| quant | size (MB) | tg128 (tok/s) |
|:--|--:|--:|
| Q2_K | 718.0 | 106.0 |
| Q4_K_M | 1065.6 | 105.5 |
| Q5_K_M | 1225.9 | 87.2 |
| Q6_K | 1396.3 | 76.6 |
| Q8_0 | 1806.8 | 72.8 |

Smaller quantization = smaller file + faster decode (memory-bandwidth-bound) but lower output quality. Q4_K_M is the production sweet spot. Q8_0 is almost-lossless but ~4× the bytes per weight; useful only when you have RAM to spare. Q2_K is for *truly* tight RAM — quality drops noticeably.
