# Giao diện dữ liệu — Language-Guided Visual Manipulation

**Phiên bản hợp đồng:** `0.1`  
**Người phụ trách đầu ra thị giác và ngôn ngữ:** Lê Võ Nghĩa Hiệp  
**Người dùng đầu ra để lập quỹ đạo và điều khiển:** Phan Thế Thông

## 1. Quyết định phạm vi

- Nhiệm vụ đầu tiên: nhập lệnh gắp một vật, đặt cạnh vật tham chiếu. Ví dụ: `Gắp khối đỏ và đặt bên trái khối xanh.`
- Nhập lệnh bằng **terminal** trong `scripts/run_demo.py`, trước mắt có thể dùng `input()` hoặc `--instruction`. Một trang HTML là phần mở rộng giao diện về sau; nó chỉ gửi cùng một chuỗi lệnh vào pipeline, không thay đổi hợp đồng giữa Hiệp và Thông.
- Quan sát bằng **một ảnh RGB từ camera cố định**.
- Hiệp cung cấp **tọa độ 3D trong hệ robot**, đơn vị mét, cho cả điểm gắp và điểm đặt.
- Khi không chắc chắn, trả trạng thái `uncertain`; controller không gắp và không tự đoán.

## 2. Chuỗi xử lý

```text
Terminal: instruction
         +
Simulator: RGB image + camera calibration + table geometry
         ↓
Hiệp: phát hiện vật → gắn vật với từ trong lệnh → ước lượng vị trí 3D
         ↓
ManipulationTarget hoặc trạng thái uncertain
         ↓
Thông: lập quỹ đạo → điều khiển → trả quan sát mới
         ↓
Hiệp: cập nhật mục tiêu từ ảnh mới khi pipeline yêu cầu
```

`src/lgvm/pipeline.py` là nơi nối các phần. Không để mã perception gọi thẳng hàm điều khiển, hoặc để controller phải hiểu câu lệnh tiếng Việt.

## 3. Quy ước tọa độ và điều kiện đo 3D

| Trường | Quy ước |
| --- | --- |
| Pixel `u, v` | `u` tăng sang phải, `v` tăng xuống dưới; pixel `(0, 0)` ở góc trên trái |
| Điểm 3D `[x, y, z]` | Đơn vị **mét**, biểu diễn trong hệ `robot_base` |
| Góc định hướng | **radian**; giai đoạn đầu chưa bắt buộc ước lượng góc xoay vật |
| Thời gian | `timestamp_s`: giây từ đồng hồ mô phỏng hoặc cùng một nguồn thời gian nhất quán |
| `camera_frame` | Tên frame camera cố định; không dùng ngầm tọa độ camera như tọa độ robot |
| Điểm đặt | Tâm vị trí mong muốn của vật sau khi thả, tính cả khoảng cách an toàn với vật tham chiếu |

**Giới hạn quan trọng:** Một ảnh RGB đơn lẻ không tự cho biết độ sâu tuyệt đối. Bản đầu tiên phải có camera intrinsics `K`, camera-to-robot extrinsics `T_robot_camera`, và mặt bàn/độ cao bề mặt vật được biết từ mô phỏng hoặc hiệu chuẩn. Hiệp chiếu tia từ pixel xuống mặt phẳng có độ cao đã biết để ước lượng 3D. Với vật cao khác nhau, vật bị che, camera đổi vị trí, hoặc mặt bàn không phẳng, kết quả có thể sai: đánh dấu `uncertain` hoặc bổ sung depth/multi-view trong phiên bản sau. Dùng ground-truth pose của simulator chỉ cho baseline và kiểm tra sai số; không gọi nó là perception từ RGB.

## 4. Input cho Hiệp: `PerceptionRequest`

Đây là **schema thiết kế** để hai bạn thống nhất; các file Python hiện là scaffold và chưa triển khai schema này.

| Trường | Kiểu | Bắt buộc | Ý nghĩa |
| --- | --- | --- | --- |
| `instruction` | `str` | Có | Lệnh người dùng, ví dụ `Gắp khối đỏ và đặt bên trái khối xanh` |
| `rgb` | `uint8[H, W, 3]` | Có | Ảnh RGB, thứ tự kênh RGB (không phải BGR) |
| `timestamp_s` | `float` | Có | Thời điểm chụp ảnh |
| `camera_frame` | `str` | Có | ID camera, ví dụ `front_camera` |
| `K` | `float[3, 3]` | Có | Ma trận nội tại camera theo độ phân giải của `rgb` |
| `T_robot_camera` | `float[4, 4]` | Có | Biến đổi tọa độ từ camera sang `robot_base`; hàng cuối `[0, 0, 0, 1]` |
| `table_height_m` | `float` | Có | Độ cao mặt bàn trong `robot_base` |
| `object_geometry` | cấu hình theo lớp vật | Có ở bản đầu | Kích thước/độ cao ước lượng để tính tiếp xúc và vị trí đặt |

`rgb` là tensor/array trong tiến trình Python, **không** đưa trực tiếp mảng pixel vào JSON log. Nếu ghi log, lưu đường dẫn ảnh và metadata.

## 5. Output Hiệp gửi Thông: `ManipulationTarget`

Luôn trả một trong hai trạng thái: `ready` hoặc `uncertain`. Các trường tọa độ chỉ được dùng khi `status == "ready"`.

| Trường | Kiểu | Khi nào có | Ý nghĩa |
| --- | --- | --- | --- |
| `status` | `"ready" | "uncertain"` | Luôn có | Có được phép lập hành động gắp hay không |
| `reason` | `str | null` | Khi `uncertain` | Lý do: `object_not_found`, `ambiguous_target`, `low_confidence`, `invalid_geometry`, `outside_workspace`, `stale_observation` |
| `instruction` | `str` | Luôn có | Lệnh đã xử lý để lưu trace |
| `timestamp_s` | `float` | Luôn có | Thời gian của quan sát đã dùng |
| `coordinate_frame` | `"robot_base"` | Luôn có | Frame chung cho tất cả điểm 3D ở output |
| `target_object_id` | `str | null` | Khi `ready` | ID vật cần gắp trong một episode, ví dụ `red_block_1` |
| `reference_object_id` | `str | null` | Khi `ready` | ID vật tham chiếu, ví dụ `blue_block_1` |
| `target_confidence` | `float | null` | Khi `ready` | Mức tin cậy trong `[0, 1]` của chọn đúng vật; cần hiệu chuẩn trước khi xem là xác suất |
| `reference_confidence` | `float | null` | Khi `ready` | Mức tin cậy của vật tham chiếu |
| `pick_position_m` | `float[3] | null` | Khi `ready` | Điểm tiếp cận/gắp trong `robot_base` |
| `place_position_m` | `float[3] | null` | Khi `ready` | Tâm vị trí đặt mong muốn trong `robot_base` |
| `relation` | `"left_of" | "right_of" | "in_front_of" | "behind" | null` | Khi `ready` | Quan hệ suy ra từ lệnh; chỉ để kiểm tra/ghi log, tọa độ đặt đã được Hiệp tính |

**Điều kiện `ready`:** tìm được hai vật khác nhau; các confidence vượt ngưỡng cấu hình; phép biến đổi camera hợp lệ; điểm gắp/đặt hữu hạn và nằm trong vùng thao tác đã thống nhất. Nếu thiếu điều kiện nào, trả `uncertain`, tọa độ bằng `null`. Không tự chọn vật khác chỉ để tiếp tục chạy.

**Thứ tự ưu tiên khi cập nhật ảnh:** dùng `timestamp_s` để không điều khiển dựa trên ảnh cũ; khi vật bị dịch chuyển, Hiệp xuất `ManipulationTarget` mới với cùng `target_object_id` nếu vẫn theo dõi được. Nếu mất dấu, trả `uncertain`.

## 6. Output tối thiểu của Thông cho pipeline

Thông nhận `ManipulationTarget(status="ready")` cùng robot state. Phần simulation/control nên cung cấp:

- `reset(seed) -> Observation`: tạo cùng một episode khi dùng cùng seed.
- `observe() -> Observation`: ảnh RGB, camera calibration, robot state, timestamp.
- `step(action) -> Observation`: thực hiện một bước điều khiển có giới hạn rồi quan sát lại.
- `success() -> bool`: tiêu chí hoàn thành từ simulator, chỉ dùng để đánh giá, không rò rỉ vào perception.
- Khi nhận `uncertain`: giữ robot ở trạng thái an toàn, ghi reason; pipeline có thể quan sát lại hoặc kết thúc episode theo cấu hình.

Chi tiết `action` (joint target, end-effector delta hay torque), tốc độ điều khiển, joint limits và convention gripper cần được Thông xác định theo simulator đã chọn rồi cập nhật phiên bản hợp đồng này **trước khi ghép code**.

## 7. Ví dụ schema minh họa (không phải kết quả đã chạy)

Lệnh từ terminal:

```text
Gắp khối đỏ và đặt bên trái khối xanh
```

Output minh họa của Hiệp; các con số dưới đây chỉ minh họa định dạng, chưa phải tọa độ đo từ simulator:

```json
{
  "status": "ready",
  "reason": null,
  "instruction": "Gắp khối đỏ và đặt bên trái khối xanh",
  "timestamp_s": 2.4,
  "coordinate_frame": "robot_base",
  "target_object_id": "red_block_1",
  "reference_object_id": "blue_block_1",
  "target_confidence": 0.93,
  "reference_confidence": 0.96,
  "pick_position_m": [0.32, -0.12, 0.04],
  "place_position_m": [0.48, 0.06, 0.04],
  "relation": "left_of"
}
```

Ví dụ khi hai khối đỏ chồng lấp khiến không chọn chắc được vật đích:

```json
{
  "status": "uncertain",
  "reason": "ambiguous_target",
  "instruction": "Gắp khối đỏ và đặt bên trái khối xanh",
  "timestamp_s": 2.5,
  "coordinate_frame": "robot_base",
  "target_object_id": null,
  "reference_object_id": null,
  "target_confidence": null,
  "reference_confidence": null,
  "pick_position_m": null,
  "place_position_m": null,
  "relation": null
}
```

## 8. Việc cần chốt khi bắt đầu tích hợp

1. `robot_base`: chiều dương của trục `x, y, z` và góc nhìn dùng để hiểu `left_of`; tuyệt đối không suy ra `left_of` từ trục `x` hay `y` trước khi thống nhất quy ước. 
2. Kích thước vật, khoảng cách an toàn đến vật tham chiếu, điểm gắp và offset gripper.
3. Ngưỡng confidence, thời gian ảnh được phép cũ, vùng thao tác, định nghĩa `uncertain` và chính sách thử lại.
4. Schema `Action` theo robot/simulator thực tế, tần số điều khiển, quy tắc va chạm.
5. Metric đánh giá độc lập: độ chính xác chọn vật, sai số tọa độ (m), success rate của cả episode, latency (ms). Ghi seed và version dữ liệu.

## 9. Checklist riêng của Hiệp

- [ ] `scripts/inspect_dataset.py`: đọc metadata và một episode, xem camera và lệnh.
- [ ] `src/lgvm/data/dataset.py`: chuẩn hóa sample và không để episode train lọt vào validation.
- [ ] `src/lgvm/perception/detector.py`: vật, ID, confidence và vị trí pixel.
- [ ] `src/lgvm/perception/pose_estimator.py`: đổi pixel sang `robot_base` với các giả định trên; xử lý trường hợp vô nghiệm.
- [ ] `src/lgvm/language/grounding.py`: chọn vật đích, vật tham chiếu và quan hệ trong lệnh.
- [ ] Phần Hiệp của `src/lgvm/planning/task_planner.py`: xuất `pick_position_m` và `place_position_m`.
- [ ] `tests/test_interfaces.py`: kiểm tra schema, đơn vị, `ready`/`uncertain` và output không chứa NaN.
