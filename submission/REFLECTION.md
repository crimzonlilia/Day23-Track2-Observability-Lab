# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Thị Diệu Linh
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/crimzonlilia/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
(.venv) (base) minh@THILINH-LAP:/mnt/d/HUST/20252/vinvin/Day23-Track2-Observability-Lab$ python 00-setup/verify-docker.py
Docker:        OK  (29.4.0)
Compose v2:    OK  (2.40.3-desktop.1)
RAM available: 4.49 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /mnt/d/HUST/20252/vinvin/Day23-Track2-Observability-Lab/00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Điều làm mình ngạc nhiên nhất là tính nhạy cảm với chữ hoa/thường (case-sensitivity) của Grafana khi cấu hình Data Source. Chỉ vì đặt tên là "Prometheus" thay vì "prometheus" mà toàn bộ Dashboard bị lỗi không tìm thấy dữ liệu. Ngoài ra, việc Alertmanager không tự động nạp biến môi trường trong file cấu hình mà phải dùng giải pháp hardcode URL Webhook cũng là một bài học kinh nghiệm quý giá về tính ổn định khi deploy container.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```
Log: {"model": "llama3-mock", "input_tokens": 7, "output_tokens": 51, "quality": 0.817, "duration_seconds": 0.1221, "trace_id": "f040d91f26442ba7906949c4488bb648", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T12:44:03.497375Z"}
Trace ID: f040d91f26442ba7906949c4488bb648
```

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

Trong cấu hình OpenTelemetry, nếu chúng ta sử dụng `probabilistic_sampler` với tỷ lệ 0.1, thì hệ thống sẽ giữ lại 10% tổng số traces. 
Công thức: `Traces_kept = N * sampling_rate`. Với N=100 req/s, số trace được lưu trữ là 10 trace/s. Việc này giúp giảm tải cho backend lưu trữ (Jaeger) mà vẫn đảm bảo tính đại diện cho dữ liệu giám sát.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

- **PSI (Population Stability Index)**: Dùng cho `prompt_length` và `response_quality` vì nó đo lường sự thay đổi phân phối tổng thể rất tốt và dễ hiểu (phát hiện drift cực mạnh > 0.2).
- **KS Test (Kolmogorov-Smirnov)**: Phù hợp cho `embedding_norm` vì đây là biến liên tục, giúp so sánh xem hai mẫu có cùng một phân phối xác suất hay không mà không cần giả định về dạng phân phối.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Chỉ số về chi phí (Cost/Tokens) là khó triển khai nhất. Lý do là vì nó yêu cầu sự phối hợp chặt chẽ giữa logic ứng dụng (đếm token) và hệ thống giám sát. Chúng ta phải định nghĩa các Custom Metrics trong Prometheus và liên tục cập nhật chúng qua mỗi request, đồng thời phải tính toán đơn giá (pricing) ngay trong lúc truy vấn (Query) trên Grafana để ra được con số USD thực tế.

---

## 6. The single change that mattered most

Thay đổi quan trọng nhất giúp hệ thống từ trạng thái "chạy được" sang "có ích" chính là việc cấu hình thành công Alertmanager kết nối với Slack. Trong môi trường production, chúng ta không thể ngồi canh Dashboard 24/7. Việc có một cơ chế cảnh báo tự động thông báo ngay lập tức khi dịch vụ `inference-api` bị sập (ServiceDown) thông qua Slack giúp đội ngũ kỹ thuật phản ứng nhanh, giảm thiểu downtime. 

Điều này kết nối trực tiếp với khái niệm "Mean Time To Detect (MTTD)" trong bài học. Việc giám sát không chỉ là thu thập dữ liệu, mà là chuyển hóa dữ liệu đó thành hành động kịp thời để đảm bảo độ tin cậy (Reliability) của hệ thống AI. Ngoài ra, việc sửa lỗi UID của Data Source cũng giúp Dashboard trở nên ổn định, tránh tình trạng mất dữ liệu hiển thị khi khởi động lại hệ thống.
