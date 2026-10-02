# Thành viên nhóm — Day 13
Điền thông tin trong thư mục riêng của nhóm. Không đưa bản có thông tin cá nhân vào lịch sử Git (commit) của kho mã công khai.

Mã nhóm/phòng: `<điền mã nhóm/phòng LC cấp>`

Tên vai trò trong bảng được giữ theo bài:

- Operator: người theo dõi và kiểm tra lượt chạy.
- Config Inspector: người kiểm cấu hình.
- Geometry Inspector: người kiểm vị trí, kích thước và hướng hộp.
- Log & Report Recorder: người ghi kết quả chạy và báo cáo.

| Họ và tên | MSSV | Vai trò lượt A | Vai trò lượt B | Vai trò lượt C |
| --- | --- | --- | --- | --- |
| Tống Thanh Danh | 2A202602299 | Operator | Log & Report Recorder | Geometry Inspector |
| Nguyễn Anh Tuấn | 2A202602282 | Config Inspector | Operator | Log & Report Recorder |
| Tô Quang Hưng | 2A202602305 | Geometry Inspector | Config Inspector | Operator |
| Tưởng Đức Tâm | 2A202602249 | Log & Report Recorder | Geometry Inspector | Config Inspector |

Ghi chú vai trò:

- Bốn vai được đổi luân phiên. Mỗi lượt có đủ 4 vai, mỗi người làm một vai khác nhau. Không ai lặp lại vai của mình qua các lượt.
- Chương trình chạy bài (runner) `student-bundle.py` chạy lần lượt A → B → C trong **một lệnh** trên máy Mac Apple Silicon (arm64) của Tống Thanh Danh: `python3 student-bundle.py run --bundle . --out ../ket-qua-nhom-01` (bắt đầu `2026-10-01 14:44:36 UTC+7`, `smoke.json` → `status: passed`). Operator của từng lượt theo dõi bước `run-A` / `run-B` / `run-C` trong nhật ký chạy (log) và `smoke.json`. Người này kiểm `status: passed` (báo chạy đạt), số hộp và `prediction_sha256` (mã hash dùng để đối chiếu file dự đoán) của lượt đó.

