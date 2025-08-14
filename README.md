# 导师检测系统
导师雷达，让你在实验室里稳稳“摸鱼”不翻车！
源码都在tag


# Galaxy FaceRecog Dashboard

> 一个基于 **Flask + InsightFace + ONNXRuntime(OpenVINO EP)** 的实时人脸识别预警系统。支持 **CPU 友好** 的实时推理、**网页端可视化**、**目标人物告警**（浏览器通知 + 声音）、**识别记录 CSV** 与 **FPS/检测次数** 展示。

---

## ✨ 特性

* **实时识别（CPU 友好）**：InsightFace `FaceAnalysis('buffalo_l')` + ONNXRuntime(OpenVINO EP)，无需独显也能流畅运行
* **网页可视化**：后端以 **MJPEG** 推流，前端 `<img>` 即可实时观看叠框视频
* **事件流同步**：通过 **SSE**（/events）推送“通知/声音/日志/FPS”，低延迟
* **目标告警（带冷却）**：命中 `POPUP_TARGET` 弹通知，命中 `SOUND_TARGET` 播放声音（前端电脑响）
* **识别记录**：非 Unknown 人员按 `RECORD_INTERVAL` 写入 `recognition_log.csv`
* **性能平衡**：**跳帧检测 + CSRT 追踪**，在算力与稳定性之间折中
* **前端体验**：Tailwind UI、深色模式、统计卡片（FPS/检测次数）、识别记录表

---

## 🧠 工作原理

* 摄像头抓帧 → 子进程 **Worker** 做“人脸检测 + 识别 + 追踪”，在帧上叠加框与标签
* 处理后的帧放入 `result_q`，由 Flask 通过 **/video\_feed** 以 **MJPEG** 推送到浏览器
* 一旦识别到目标人，Worker 按冷却时间向 `event_q` 投递事件（声音/通知/日志/FPS）
* Flask 通过 **/events** 将事件以 **SSE** 实时推给前端；前端播放声音、弹通知、更新表格与统计

```mermaid
flowchart LR
    Cam[Camera] -->|Frames| Worker
    Worker -->|Processed Frames| RQ[(result_q)]
    Worker -->|Events| EQ[(event_q)]
    RQ --> Flask
    EQ --> Flask
    Flask -->|/video_feed (MJPEG)| Browser
    Flask -->|/events (SSE)| Browser
    Browser -->|Play| Audio[(<audio>/sound/man.wav)]
    Browser -->|Notify/Log/FPS| UI[Dashboard UI]
    Worker -->|CSV append| CSV[(recognition_log.csv)]
```

---

## 📁 目录结构

```
project/
├─ app.py               # 后端主程序（Flask + 推理 + 流/事件）
├─ index.html           # 前端单页（Tailwind + SSE + MJPEG）
├─ image/
│  └─ face_data/
│      ├─ junliang.jpg  # 已知人脸样本（文件名=识别名，不含扩展名）
│      └─ alice.jpg
├─ sound/
│  └─ man.wav           # 前端播放的告警音频（由后端 /sound 提供）
└─ recognition_log.csv  # 识别日志（首次运行无则自动创建）
```

---

## 🚀 快速上手

### 1) 环境

* Python 3.9+（建议：虚拟环境）

```bash
pip install opencv-contrib-python onnxruntime-openvino insightface flask numba numpy
# 若 OpenVINO 不可用，可退而求其次：
# pip install onnxruntime
```

### 2) 放好资源

* 把 **已知人脸** 图片放到 `image/face_data/`（文件名即识别名，如 `junliang.jpg` → “junliang”）
* 把 **声音文件** 放到 `sound/man.wav`（前端 `<audio src="/sound/man.wav">` 会来这里取）

### 3) 运行

```bash
python app.py
```

浏览器访问：`http://<服务器IP>:5000` → 点击“点击开始” → 授权通知 → 即可观看与联动告警

---

## ⚙️ 关键配置（app.py 顶部）

```python
COS_THRESHOLD   = 0.4      # 相似度阈值（高：保守易漏；低：敏感易误）
DETECT_INTERVAL = 8        # 跳帧检测间隔（大：省算力但漂移风险升；小：更稳但耗算力）
POPUP_TARGET    = "junliang"   # 弹窗目标（与 face_data 文件名一致、无后缀）
SOUND_TARGET    = "junliang"   # 声音目标（同上）
POPUP_COOLDOWN  = 30       # 通知冷却（秒）
SOUND_COOLDOWN  = 30       # 声音冷却（秒）
RECORD_INTERVAL = 600      # 同一人识别日志最短间隔（秒）
```

> **命名规则**：`KNOWN_FACES_DIR` 内图片的\*\*文件名（不带扩展名）\*\*就是识别名。需与 `SOUND_TARGET/POPUP_TARGET` 完全一致（区分大小写）。

---

## 📌 使用小贴士

* **声音在哪播放？** 在**前端电脑**（打开网页的那台）。后端只发事件，前端 `<audio>` 播放 `/sound/man.wav`
* **多个页面同时打开？** 当前 SSE 为“共享队列，先到先得”，**不是广播**；要广播需改成“发布-订阅”（每连接一队列）
* **Windows 摄像头**：代码用 `cv2.CAP_DSHOW` 打开摄像头；如异常可去掉该标志或更换索引
* **性能调优**：从 `DETECT_INTERVAL`、`det_size=(640,640)`、ONNXRuntime 线程数等入手；人脸库样本清晰、统一也很关键
* **不同人不同声音**：可在事件里附带文件名，前端动态切换 `<audio>.src` 实现

---

## 🖼️ 截图（占位）

![Dashboard](./docs/screenshot.png)

---

## 🔐 隐私与安全

仅在受信任的网络中部署；识别日志仅包含时间与姓名，如需脱敏/访问控制，请自行扩展。

---

## 📄 许可

请根据你的项目选择合适的 License（MIT/Apache-2.0 等），并在仓库根目录添加 `LICENSE`。
