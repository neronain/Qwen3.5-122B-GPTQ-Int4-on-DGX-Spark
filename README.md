# 🚀 Deploying Qwen3.5-122B (GPTQ-Int4) on DGX Spark (GB10) for VS Code & Cline

Repository นี้รวบรวมขั้นตอนการตั้งค่าและ Deploy โมเดล **Qwen3.5-122B-A10B-GPTQ-Int4** (รองรับ Context Window ขนาดใหญ่) บนเครื่อง **NVIDIA DGX Spark (สถาปัตยกรรม Blackwell GB10 - 128GB Unified Memory)** โดยใช้ **vLLM** ผ่าน Docker Compose เพื่อนำมาใช้งานเป็น AI Coding Assistant ส่วนตัวคู่กับ Extension **Cline** บน VS Code

## 📋 Prerequisites (สิ่งที่ต้องมี)

1. เครื่อง **DGX Spark (GB10)** หรือเซิร์ฟเวอร์ที่มี VRAM อย่างน้อย 128GB
2. ติดตั้ง **Docker** และ **NVIDIA Container Toolkit** เรียบร้อยแล้ว
3. VS Code พร้อมติดตั้ง Extension **Cline**
4. (Optional) Hugging Face Token หากใช้โมเดลที่ต้องการการยืนยันสิทธิ์

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
