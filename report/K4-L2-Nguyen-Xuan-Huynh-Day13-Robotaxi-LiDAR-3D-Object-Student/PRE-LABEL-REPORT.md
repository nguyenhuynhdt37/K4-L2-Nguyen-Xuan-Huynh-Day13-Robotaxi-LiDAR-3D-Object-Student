# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: _(chưa được cấp)_
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Xuân Huỳnh; 01/10/2026 ~14:49 UTC+7; macOS Apple Silicon arm64
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-arm64` / `sha256:dd6999ad5dd67962fdca18eb526132c08ce105980c9d89475d66cb1a5193f8a1`; repo revision `0831856`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` / frame `demo`; chạy trên máy cá nhân; input SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI có sẵn trong image; `/opt/PointPillars/pretrained/epoch_160.pth` SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance thật bị bỏ, dùng RGB=0 placeholder; z_ground ước lượng từ histogram = 0.075 m

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | boxes-demo-delta-0-voxel-0.16.json, side-demo-delta-0-voxel-0.16.png, summary.csv | Chỉ phát hiện 1 vehicles. Mất toàn bộ 9 xe, 2 người, 1 xe hai bánh so với B. |
| B | 1.73 | 0.16 | 13 | 1.034 | boxes-demo-delta-1.73-voxel-0.16.json, side-demo-delta-1.73-voxel-0.16.png, summary.csv | Baseline: 10 vehicles, 2 pedestrian, 1 two-wheels. Hộp bám cụm điểm, đáy gần mặt đường trên ảnh Side. |
| C | 1.73 | 0.32 | 6 | 1.091 | boxes-demo-delta-1.73-voxel-0.32.json, side-demo-delta-1.73-voxel-0.32.png, summary.csv | Mất hoàn toàn vehicles (0 xe), chỉ còn 6 pedestrian. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh side-A chỉ hiện 1 hộp vehicles nằm gần mặt đường, side-B hiện 13 hộp phân bố rõ theo cụm điểm. JSON-A chỉ có 1 entry, JSON-B có 13 entries gồm 3 class. Đây là chạy lại model trên input đã dịch z khác nhau, không chỉ dịch hộp cũ: delta=0 khiến phần lớn điểm lệch khỏi cửa sổ z [-3, +1] của model KITTI và trật khỏi anchor, nên model gần như không phát hiện được vật thể.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. B phát hiện đủ 3 class (10 vehicles, 2 pedestrian, 1 two-wheels); C mất hoàn toàn vehicles, chỉ còn 6 pedestrian. Pillar 0.32 gấp đôi 0.16, diện tích ô gấp 4, đặc trưng hình học bị gộp thô khiến model không nhận diện được xe. Trên PCD demo này, cấu hình B (voxel 0.16) rõ ràng phát hiện đầy đủ hơn C; tuy nhiên chỉ thử 1 frame KITTI nên chưa đủ cơ sở kết luận cấu hình nào luôn tốt hơn trên mọi dữ liệu.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Chỉ dùng front-window nên vật thể phía sau xe hoàn toàn không được quét; không thể kết luận model bỏ sót chỉ vì không thấy hộp ở phía sau. Ảnh Side chỉ hiện mặt cắt bên (x-z), không thấy được hướng yaw; cần kết hợp ảnh Top và camera để kiểm yaw.
- JSON nào còn chưa đủ cơ sở để import? Cả 3 JSON A/B/C đều là prediction KITTI trên PCD demo, không được import vào job Robotaxi vì khác frame và khác dữ liệu. Các ca QC (`case-*.json`) ghi rõ `training_only`, cũng không import.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Giữ nguyên — đường cơ sở | 13/13 hộp giữ nguyên toạ độ gốc từ prediction B; đáy bám mặt đường trên ảnh side-correct.png |
| case-batch-z | 13 / 13 | -1.805 m (delta 1.73 + z_ground 0.075) | Không đổi | **Dừng batch**: lỗi pipeline, không sửa tay | Toàn bộ 13 hộp đồng loạt chìm dưới mặt đất cùng 1.805 m trên ảnh side-batch-z.png; class/x/y/yaw giữ nguyên → lỗi hệ thống ở khâu chuyển đổi z ngược |
| case-one-box-z | 1 / 13 | -1.805 m | Không đổi | **Kiểm từng hộp**: không phải lỗi pipeline | Chỉ hộp đầu tiên bị chìm 1.805 m trên ảnh side-one-box-z.png, 12 hộp còn lại vẫn đúng vị trí → lỗi đối tượng đơn lẻ, cần kiểm hộp đó bằng nhiều góc nhìn |

Ghi rõ: helper tạo biến đổi có chủ đích từ prediction B thật (SHA256 `51f49ac6ba8c97f8a2458e289fee48a33e1d07dd94de99ba935c142fd7e26016`), không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Nguyễn Xuân Huỳnh — 2A202602206
- Vai trò đã làm: vận hành lệnh, kiểm cấu hình/JSON, xem hình học và ghi log cho cả 3 lượt A/B/C (làm một mình).
- Quan sát A/B/C có dẫn file: So `run-A/boxes-demo-delta-0-voxel-0.16.json` chỉ có 1 hộp vehicles vs `run-B/boxes-demo-delta-1.73-voxel-0.16.json` có 13 hộp (10 vehicles, 2 pedestrian, 1 two-wheels). Dịch z trước inference thay đổi hoàn toàn số lượng và loại hộp, không chỉ dịch vị trí hộp. So `run-C` mất hoàn toàn vehicles khi pillar tăng gấp đôi.
- Phép z thuận/ngược: `z_model = z_source - z_ground - delta`; lúc ra: `z_source = z_model + delta + z_ground`. Quên phép ngược → hộp bị chìm đúng 1.805 m (delta 1.73 + z_ground 0.075).
- Quyết định lỗi batch: case-batch-z cho thấy tất cả 13 hộp cùng chìm cùng lượng → dừng batch, báo kiểm pipeline chuyển đổi z, không sửa tay từng hộp. case-one-box-z chỉ 1 hộp lệch → kiểm riêng hộp đó.
- Điều chưa chắc: reflectance thật bị bỏ và thay bằng hằng số (0 hoặc 0.7 tuỳ class), chưa rõ ảnh hưởng cụ thể đến chất lượng detection pedestrian/two-wheels trên dữ liệu Robotaxi thật so với KITTI.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
