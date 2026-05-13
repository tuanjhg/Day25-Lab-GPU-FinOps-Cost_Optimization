# Báo cáo GPU FinOps & Cost Optimization

## Thông tin sinh viên
- Họ và tên: Khổng Mạnh Tuấn
- MSSV: 2A202600086
- Ngày thực hiện: 2026-05-13

## 1. Giới thiệu
Bài lab hướng tới việc thực hành quy trình GPU FinOps trên hệ thống giả lập (mock cluster) và workload GPU thực tế. Mục tiêu chính là theo dõi hiệu suất GPU, phân bổ chi phí, phát hiện lãng phí, và tối ưu chi phí bằng các chiến lược như spot instance, autoscaling, và mixed precision.

GPU FinOps là tập hợp các thực hành giúp cân bằng giữa hiệu năng và chi phí: minh bạch chi phí theo workload, giảm lãng phí tài nguyên idle, và đưa ra khuyến nghị tối ưu theo rủi ro và effort.

## 2. Phân tích từng phần

### Part 1-7: Mock cluster

**Part 1: Cluster monitoring**
- Cluster có 4 nodes, tổng 8 GPUs (T4, A100, V100).
- Cluster metrics:
  - Busy GPUs: 5, Idle GPUs: 3
  - Avg utilization: 53.8%
  - Memory used: 185.2 GB / 288.0 GB
  - Total power draw: 949 W
- Điểm nổi bật: node-01 (A100) có utilization cao (80.7% và 89.2%), trong khi node-03 (T4) đang nhận tải thấp (11-13%). Điều này cho thấy có sự không đồng đều về tải giữa các node.

**Part 2: Workload submission & cost tracking**
- Sau khi submit workload, Busy GPUs = 8/8, utilization tăng lên 76.0%.
- Chi phí theo workload:
  - train-resnet-001: $0.0292 (on-demand)
  - train-bert-002: $0.6117 (on-demand)
  - inference-api-003: $0.0035 (spot, saved $0.0082)
  - train-llm-004: $0.5505 (spot, saved $1.2845)
- Billing summary: total cost $2.3898, total savings $2.5854, budget used 2.4%, alert OK.
- Nhận xét: Workload spot mang lại savings rõ ràng, tổng savings lớn hơn tổng cost, cho thấy spot đang tạo ra lợi ích chi phí đáng kể.

**Part 3: Spot instance management**
- Spot pricing:
  - T4: on-demand $0.35, spot $0.2271, discount 35.1% (availability medium)
  - A100: on-demand $3.67, spot $2.2660, discount 38.3% (availability medium)
  - V100: on-demand $2.48, spot $1.7119, discount 31.0% (availability low)
- Spot requests: 3/3 granted.
- Preemption: 1 instance bị preempt (spot-a100-001), còn 2 instance hoạt động.
- Savings report: spot cost $0.0001, on-demand equivalent $0.0002, saved $0.0002 (16.4%).
- Nhận xét: Spot giảm giá rõ ràng nhưng có rủi ro preemption, cần thiết kế workload fault-tolerant.

**Part 4: Autoscaling**
- Policy: scale_up_threshold 70%, scale_down_threshold 25%, cooldown 30s, min_nodes 2, max_nodes 10, preferred GPU T4, cost_aware True.
- Evaluation:
  - Action SCALE_UP (util 76.0% > 70%), nodes 4 -> 5
  - 5 cycles tiếp theo: no_action, util ~60.8%, nodes 5 -> 5
- Nhận xét: Autoscaler tăng node khi tải vượt ngưỡng, sau đó ổn định khi utilization giảm.

**Part 5: Cost analysis & recommendations**
- Cost snapshots (5 lần): total $0.040000/interval, idle $0.001944, waste 4.9%.
- Waste report: avg waste 14.0%, total idle cost $0.053885, total cost $0.390280, potential monthly save $1396.70, severity LOW.
- Recommendations:
  - USE_SPOT (MEDIUM): savings ~65%
  - SCHEDULING (LOW): savings ~20%
- Dashboard:
  - Cluster: 10 GPUs, utilization 60.8%, busy 8, idle 2
  - Billing: $2.3898 / $100, savings $2.5854, alert OK
  - Spot: saved $0.0032 (70.0%)
  - Waste: 14.0% (LOW)
- Nhận xét: Waste thấp và severity LOW, nhưng có tiềm năng tiết kiệm lớn nếu mở rộng theo tháng.

**Part 6: Visualization**
- Cost by GPU type: A100 chiếm phần lớn chi phí.
- Spot vs on-demand: spot cost thấp hơn rõ ràng.
- Budget utilization: 2.4% used.
- Time-series: cost active chiếm chủ đạo, waste dưới ngưỡng cảnh báo (30%).

**Part 7: Full FinOps workflow**
- Initial: GPUs 10, util 60.8%, idle 2
- After load: util 78.7%, busy 10/10
- Autoscaler: scale_up
- Cost/interval: $0.041944, waste 4.6%
- Spot savings: $0.0062 (70.0%)
- Final billing: total spend $2.5589, total saved $2.7078, budget 2.6% used
- Nhận xét: Chu trình tối ưu đầy đủ cho thấy autoscaler và spot kết hợp giúp giảm chi phí trong khi duy trì utilization cao.

### Part 8: Real GPU training

**GPU profile**
- GPU: Tesla T4, 15.6 GB, $0.35/hr, CUDA 12.8
- Pynvml available, metrics diagnostic OK (power ~10.5W, temp ~38C lúc nhàn).

**FP32 vs AMP (ResNet-18 on CIFAR-10)**
- FP32:
  - Total time: 124.3s
  - Peak mem: 0.82 GB
  - Avg GPU util: 94.5%, Max util: 98.0%
  - Avg power: 67.1W, Avg temp: 65.2C
  - Estimated cost: $0.012083
- AMP:
  - Total time: 61.7s
  - Peak mem: 0.60 GB
  - Avg GPU util: 88.3%, Max util: 93.0%
  - Avg power: 63.4W, Avg temp: 77.4C
  - Estimated cost: $0.005999

**So sánh và tiết kiệm**
- Speedup: 2.01x nhanh hơn
- Mem saving: 0.22 GB
- Cost saving: $0.006084 (~50.4%)
- Extrapolated savings:
  - 1 day: save $4.23
  - 1 week: save $29.61
  - 1 month: save $126.88

**Reporting to gateway**
- FP32 cost: $0.012100 (rate $0.35/hr)
- AMP (spot): cost $0.001800, saved $0.004200
- Project real-gpu-lab: total cost $0.013900, savings $0.004200
- Final dashboard: total platform cost $2.5589, total savings $2.7078, budget used 2.6%, alert OK

**GPU utilization patterns**
- FP32 có utilization cao và ổn định (~95%+), AMP giảm nhẹ util nhưng rút ngắn thời gian.
- Power draw AMP thấp hơn FP32; nhiệt độ AMP nhìn chung cao hơn do temperature được duy trì ở mức cao trong thời gian ngắn hơn.

### Part 8.5: Advanced analysis

**Multi-GPU scaling efficiency**
- Kết quả (T4, base_time 2h):
  - 1 GPU: speedup 0.95, efficiency 95.0%, time 2.105h, cost $0.7368
  - 2 GPU: speedup 1.8025, efficiency 90.1%, time 1.110h, cost $0.7767
  - 4 GPU: speedup 3.2490, efficiency 81.2%, time 0.616h, cost $0.8618
  - 8 GPU: speedup 5.8563, efficiency 73.2%, time 0.342h, cost $0.9562
- Best cost efficiency: 8 GPUs (cost_per_speedup $0.1633).
- Nhận xét: Efficiency giảm dần khi scale, nhưng 8 GPUs vẫn tối ưu về cost_per_speedup theo giả định.

**Project cost forecasting**
- Total base cost: $3551.20
- Contingency: $710.24
- Expected total cost: $4261.44
- 95% CI: $2913.09 -> $5609.79
- Nhận xét: Giai đoạn Model Training và Hyperparameter Tuning chiếm phần lớn chi phí.

**Optimization strategy prioritization**
- Ưu tiên theo score:
  1) Switch to Mixed Precision (AMP) - adj savings 25%, low effort, low risk
  2) Use Spot Instances - adj savings 39%, medium effort, high risk
  3) Optimize Batch Size - adj savings 15%, low effort, low risk
- Quick wins: AMP, Optimize Batch Size
- Nhận xét: Kết hợp AMP + batch size là giải pháp nhanh, ít rủi ro, hiệu quả.

**Challenge scenario (LLM fine-tuning)**
- Baseline: 8xA100, 200h, cost $5872.00 (budget $5000)
- Selected strategies: AMP, Optimize Batch Size, Switch to More Efficient GPU Type
- Final expected cost: $2841.24
- 95% CI: $1630.63 -> $4051.86
- Budget check: OK

## 3. Kết luận và học hỏi
- Kỹ năng FinOps học được: theo dõi GPU, ghi nhận chi phí theo workload, đánh giá waste, autoscaling theo ngưỡng, lập báo cáo chi phí và dự báo.
- Chiến lược tối ưu hiệu quả: AMP (giảm 50% chi phí), spot instances (tiết kiệm lớn nhưng cần fault-tolerant), scheduling off-peak, autoscaling.
- Ứng dụng thực tế: áp dụng cho dự án training dài hạn, ước tính chi phí theo phase, và chọn chiến lược tối ưu theo risk/effort.

## Minh chứng
Screenshots và biểu đồ được lưu trong thư mục submission/ và generated_charts (nếu có).