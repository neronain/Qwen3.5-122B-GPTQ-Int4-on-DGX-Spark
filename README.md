# 🚀 Deploying Qwen3.5-122B (GPTQ-Int4) on DGX Spark (GB10) for VS Code & Cline

Repository นี้รวบรวมขั้นตอนการตั้งค่าและ Deploy โมเดล **Qwen3.5-122B-A10B-GPTQ-Int4** (รองรับ Context Window ขนาดใหญ่) บนเครื่อง **NVIDIA DGX Spark (สถาปัตยกรรม Blackwell GB10 - 128GB Unified Memory)** โดยใช้ **vLLM** ผ่าน Docker Compose เพื่อนำมาใช้งานเป็น AI Coding Assistant ส่วนตัวคู่กับ Extension **Cline** บน VS Code

## 📋 Prerequisites (สิ่งที่ต้องมี)

1. เครื่อง **DGX Spark (GB10)** หรือเซิร์ฟเวอร์ที่มี VRAM อย่างน้อย 128GB
2. ติดตั้ง **Docker** และ **NVIDIA Container Toolkit** เรียบร้อยแล้ว
3. VS Code พร้อมติดตั้ง Extension **Cline**
4. (Optional) Hugging Face Token หากใช้โมเดลที่ต้องการการยืนยันสิทธิ์

https://huggingface.co/Qwen/Qwen3.5-122B-A10B-GPTQ-Int4

## 🛠️ Step 1: Deployment (การรัน Server)

เราจะใช้ `vLLM` ในการทำตัวเป็น API Server (OpenAI Compatible) เพื่อให้แอปพลิเคชันอื่นเรียกใช้งานได้ง่าย

1. สร้างไฟล์ `docker-compose.yaml` และใส่โค้ดด้านล่างนี้:

```yaml
services:
  vllm:
    image: vllm/vllm-openai:nightly
    container_name: qwen-122b-serving
    runtime: nvidia
    ipc: host
    ports:
      - "8000:8000"
    volumes:
      # Mount โฟลเดอร์โมเดลหรือ HF Cache ของคุณ
      - ~/.cache/huggingface:/root/.cache/huggingface 
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    command: >
      --model Qwen/Qwen3.5-122B-A10B-GPTQ-Int4
      --served-model-name qwen-122b
      --port 8000
      --tensor-parallel-size 1
      --max-model-len 32768
      --quantization gptq
      --reasoning-parser qwen3
      --enable-auto-tool-choice
      --tool-call-parser qwen3_coder
      --gpu-memory-utilization 0.85
      --trust-remote-code
```

*หมายเหตุ: ปรับ `--gpu-memory-utilization` ให้อยู่ที่ประมาณ `0.85` เพื่อป้องกันปัญหา Out of Memory (OOM) ระหว่างการเริ่มต้นระบบ*

2. รันคำสั่งเพื่อเปิดเซิร์ฟเวอร์:

```bash
docker compose up -d

```

3. ตรวจสอบสถานะการโหลดโมเดล:

```bash
docker logs -f qwen-122b-serving

```

*(รอจนกว่าจะขึ้นข้อความ `Uvicorn running on http://0.0.0.0:8000`)*

## 💻 Step 2: VS Code + Cline Integration

เมื่อ Server รันเรียบร้อยแล้ว ให้ตั้งค่าใน Extension **Cline** บน VS Code ดังนี้:

1. เปิดการตั้งค่า (Settings) ของ Cline
2. ไปที่หัวข้อ **API Configuration**
3. กำหนดค่าตามนี้:
* **API Provider:** `OpenAI Compatible`
* **Base URL:** `http://<IP_ของเครื่อง_DGX>:8000/v1` *(อย่าลืมใส่ `/v1` ต่อท้าย)*
* **API Key:** `EMPTY` *(หรือพิมพ์อักษรใดๆ ก็ได้ ห้ามปล่อยว่าง)*
* **Model ID:** `qwen-122b` *(ต้องตรงกับ `--served-model-name` ใน Docker)*



## 🎛️ Step 3: Recommended Sampling Parameters

สถาปัตยกรรมของ Qwen จะทำงานได้ดีที่สุดเมื่อปรับค่า Parameter ให้ตรงกับลักษณะงาน (อ้างอิงจาก Official Recommendation) คุณสามารถนำค่าเหล่านี้ไปปรับใช้ใน Code ของคุณ หรือใส่ผ่าน Custom Headers/Settings ของ Client ที่รองรับ:

| Mode | Tasks | Temperature | Top P | Top K | Min P | Presence Penalty | Repetition Penalty |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Thinking** | General Tasks | `1.0` | `0.95` | `20` | `0.0` | `1.5` | `1.0` |
| **Thinking** | Precise Coding (e.g., WebDev) | `0.6` | `0.95` | `20` | `0.0` | `0.0` | `1.0` |
| **Instruct** | General Tasks | `0.7` | `0.8` | `20` | `0.0` | `1.5` | `1.0` |
| **Instruct** | Reasoning Tasks | `1.0` | `1.0` | `40` | `0.0` | `2.0` | `1.0` |

*Note: Support for specific sampling parameters (like `top_k`, `min_p`) may vary depending on the exact inference framework and client implementation.*

---

**Troubleshooting Tips:**

* หาก Client (เช่น Cline) เชื่อมต่อไม่ได้ ให้ลองรันคำสั่ง `curl http://<IP>:8000/v1/models` เพื่อเช็คว่าพอร์ตถูกบล็อกด้วย Firewall หรือไม่ และตรวจสอบชื่อ Model ID ที่ระบบกำลังให้บริการอยู่

```
