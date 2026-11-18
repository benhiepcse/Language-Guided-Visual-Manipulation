# Language-Guided Visual Manipulation with Feedback Control

Đồ án nhóm 2 người: Lê Võ Nghĩa Hiệp (AI Perception, VLM/VLA) và Phan Thế Thông (Control & Simulation).

## 1. Bài toán

Robot nhận ảnh từ camera và câu lệnh như "gắp vật A rồi đặt bên trái vật B". Hệ thống xác định vật, ước lượng vị trí, lập mục tiêu, điều khiển tay gắp, quan sát lại và sửa hành động nếu có sai lệch. Phạm vi đầu tiên: robot một tay, thao tác trên bàn, lệnh pick-and-place, 3–5 loại vật. Mọi tuyên bố về VLA chỉ dùng khi đã thực sự tích hợp policy ảnh + ngôn ngữ -> hành động; baseline parser theo luật phải được gọi đúng tên.

## 2. Đầu vào / đầu ra

- Input: ảnh RGB (và camera parameters nếu cần), câu lệnh, trạng thái robot; depth là tùy chọn.
- Output: vật đích, vị trí gắp/đặt, lệnh điều khiển và trạng thái thành công/thất bại.
- Feedback loop: nhận quan sát mới sau mỗi bước điều khiển, cập nhật mục tiêu và sửa quỹ đạo.

## 3. Phân công

| Người | Phần việc | Bàn giao |
| --- | --- | --- |
| Hiệp | đọc dữ liệu, perception, ước lượng pose, language grounding, tùy chọn VLM/VLA nhỏ | model/config, bảng sai số, phân tích lỗi nhận diện |
| Thông | mô phỏng, inverse kinematics, trajectory, feedback controller, perturbations | môi trường chạy được, controller, đồ thị quỹ đạo và va chạm |
| Cả hai | interface, pipeline, evaluation, ablation, video, README | demo và benchmark tái lập được |

## 4. Ngôn ngữ và phân công theo file

Ngôn ngữ lập trình chính: **Python 3.11** (`.py`). Script cài đặt là **PowerShell** (`.ps1`) cho Windows. Cấu hình dùng YAML (`.yaml`) và TOML (`.toml`); tài liệu dùng Markdown (`.md`). MuJoCo/LIBERO là thư viện và môi trường mô phỏng, không phải ngôn ngữ lập trình.

| File / nhóm file | Người chính | Việc cần viết |
| --- | --- | --- |
| `src/lgvm/data/dataset.py` | Hiệp | Đọc episode, ảnh, lệnh và nhãn từ dataset |
| `src/lgvm/perception/detector.py` | Hiệp | Xác định vật đích trong ảnh, confidence |
| `src/lgvm/perception/pose_estimator.py` | Hiệp | Ước lượng tọa độ vật, đổi hệ camera sang robot |
| `src/lgvm/language/grounding.py` | Hiệp | Ghép câu lệnh với vật/quan hệ không gian |
| `scripts/inspect_dataset.py` | Hiệp | Kiểm tra metadata, video, episode |
| `scripts/train_perception.py` | Hiệp | Huấn luyện hoặc đánh giá perception; không mặc định là VLA |
| `src/lgvm/simulation/environment.py` | Thông | Adapter reset/observe/step cho LIBERO/MuJoCo |
| `src/lgvm/control/trajectory.py` | Thông | Sinh quỹ đạo tiếp cận, gắp và đặt |
| `src/lgvm/control/controller.py` | Thông | Điều khiển phản hồi, giới hạn hành động, sửa sai lệch |
| `tests/test_controller.py` | Thông | Test giới hạn và phản ứng với nhiễu |
| `src/lgvm/planning/task_planner.py` | Cả hai | Hiệp xác định mục tiêu; Thông chuyển thành các pha hành động |
| `src/lgvm/pipeline.py` | Cả hai | Nối perception, grounding, planner, controller và simulator |
| `src/lgvm/evaluation/metrics.py` | Cả hai | Định nghĩa điều kiện thành công, sai số và latency |
| `scripts/run_demo.py`, `scripts/evaluate.py` | Cả hai | Chạy demo, benchmark và lưu logs/video |
| `tests/test_interfaces.py`, `tests/test_metrics.py` | Cả hai | Kiểm tra contract và phép đo |
| `docs/INTERFACES.md`, `docs/EXPERIMENTS.md`, `docs/TASKS.md` | Cả hai | Chốt format dữ liệu, giao thức đánh giá và tiến độ |
| `configs/*.yaml`, `pyproject.toml`, `README.md`, `.gitignore`, `.env.example` | Cả hai | Cấu hình, dependency và tài liệu |
| `src/lgvm/**/__init__.py` | Cả hai | Khai báo Python package; có thể giữ rỗng ban đầu |
| `data/`, `models/`, `outputs/`, `notebooks/` | Cả hai | Dữ liệu, weights, kết quả và thử nghiệm; file `.gitkeep` chỉ giữ folder |

### Cách làm để hai phần tích hợp được

Hiệp trả về `object_id`, `confidence`, `target_pose` và tên `coordinate_frame`; Thông nhận những giá trị đó cùng `robot_state` để tạo `action`. Hai bạn chốt kiểu dữ liệu và đơn vị trong `docs/INTERFACES.md` trước khi code độc lập. `pipeline.py` gọi lần lượt từng phần, lấy quan sát mới sau `step(action)` rồi cập nhật mục tiêu khi cần.

## 5. Dữ liệu và mô phỏng

Script cài đặt tải **toàn bộ repository dataset `lerobot/libero`**, không chỉ một subset, vào `data/libero`. Theo tài liệu LeRobot, bộ này khoảng 1.9 GB, gồm 1,693 episodes, 273,465 frames và 40 tasks, với video từ hai camera. Cần video decoder tương thích khi đọc ảnh. Dataset là demonstration để học/thử baseline; nó không tự cài đặt simulator. Để chạy benchmark tương tác cần cài và kết nối LIBERO/MuJoCo theo tài liệu chính thức, sau đó hiện thực adapter `src/lgvm/simulation/environment.py`. Không trộn số liệu offline từ dataset với success rate online.

Nguồn: https://huggingface.co/docs/lerobot/main/libero ; https://huggingface.co/datasets/lerobot/libero ; https://github.com/Lifelong-Robot-Learning/LIBERO ; https://mujoco.readthedocs.io/en/stable/python.html

## 5. Kiến trúc và interface

`image + instruction + robot_state -> perception -> grounding -> target/grasp -> planner -> controller -> simulator -> new observation -> evaluation`. Giao diện thống nhất dùng mét, radian, timestamp, camera frame và robot base frame; document transform giữa các frame trong `docs/INTERFACES.md`. Lưu seed, cấu hình, dataset revision và phiên bản thư viện khi báo cáo.

## 6. Milestones và tiêu chí nghiệm thu

1. M0 — Dataset: tải xong, kiểm tra metadata, đọc được một episode cùng hai camera. Thành công = số file > 0 và decode được sample; script tạo repo chưa tự thực hiện bước decode.
2. M1 — Simulator: một tác vụ pick-and-place chạy bằng ground-truth state; camera/rendering và reset theo seed hoạt động.
3. M2 — Hiệp: định vị vật và grounding câu lệnh; báo cáo sai số, đúng vật đích và lỗi trên vật bị che/nhiễu.
4. M3 — Thông: controller quan sát trạng thái mới, sửa quỹ đạo khi vật bị dịch chuyển; báo cáo thời gian, va chạm, sai số cuối.
5. M4 — Tích hợp: chạy end-to-end trên lệnh và vị trí ngẫu nhiên; lưu video thành công/thất bại và logs.
6. M5 — Benchmark: ít nhất 3 seeds và 30 episodes mỗi điều kiện (mục tiêu đề xuất); so sánh oracle state + controller, predicted perception + controller, và tùy chọn VLA nếu huấn luyện được.

## 7. Metric, thí nghiệm và ablation

- Task success rate = số episode thành công / tổng episode; định nghĩa thành công dựa trên điều kiện của simulator.
- Object selection accuracy, position/grasp error (m), collision count, control latency (ms), completion time (s).
- Perturbation: vị trí vật, màu/ánh sáng, vật gây nhiễu, cách diễn đạt lệnh, dịch chuyển vật trong lúc gắp.
- Ablation: oracle vs predicted perception; open-loop vs feedback; có/không có replan; nếu dùng VLA thì so sánh cùng seed và điều kiện.
- Công bố cả seed, interval/bảng theo từng task và failure cases; không dùng video đơn lẻ làm bằng chứng về success rate.

## 8. Trình tự triển khai

Đọc `docs/INTERFACES.md` trước, rồi `data/dataset.py`, `simulation/environment.py`, `control/controller.py`, `perception/`, `language/`, `planning/`, `pipeline.py`, cuối cùng `evaluation/` và `scripts/`. Các file mã nguồn hiện là scaffold rỗng để nhóm tự triển khai; tạo xong repo và tải dataset **không có nghĩa là demo đã chạy được**.

## 9. Máy và cách chạy

- Hiệp: RTX 4060 Laptop 8 GB để thử perception và model nhỏ, giảm batch/độ phân giải nếu thiếu VRAM.
- Thông: i7-1355U, RAM 16 GB, Intel Iris Xe để viết và chạy controller, mô phỏng ít environment song song; kiểm tra tốc độ render thực tế.
- Windows PowerShell: chạy script setup từ ổ D. Sau khi tải xong, tạo venv `py -3.11 -m venv .venv`, kích hoạt `.\.venv\Scripts\Activate.ps1` và cài dependency theo từng milestone. Không mặc định rằng dataset tải xong là có policy hoặc simulator.

## 10. Deliverables cho CV

Repo có hướng dẫn tái lập, environment lock, dataset revision, demo video, báo cáo metric/ablation, đồ thị trajectory, phân tích failure và phần đóng góp riêng của từng người. Không commit `data/libero`, weights hoặc video lớn lên Git.
