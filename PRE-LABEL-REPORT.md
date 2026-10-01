# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục riêng của nhóm để LC thu. Bài này dùng để xem nhóm đã hiểu và làm được đến đâu; không ghi điểm của người khác.

---

## Nhóm và provenance

Provenance là thông tin về nguồn dữ liệu và cách nhóm đã chạy bài. Các mã hash bên dưới là mã dùng để đối chiếu đúng file hoặc đúng bản phần mềm.

- Mã nhóm/phòng: `<điền mã nhóm/phòng LC cấp>`
- Thành viên: xem `TEAMMATES.md` (Tống Thanh Danh - 2A202602299, Nguyễn Anh Tuấn - 2A202602282, Tô Quang Hưng - 2A202602305, Tưởng Đức Tâm - 2A202602249).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; loại máy và kiến trúc xử lý:
  - Operator (người theo dõi và kiểm tra lượt chạy) luân phiên theo lượt: Tống Thanh Danh (A), Nguyễn Anh Tuấn (B), Tô Quang Hưng (C), xem `TEAMMATES.md`.
  - Runner (chương trình chạy bài) chạy cả ba lượt trong một lệnh trên máy của Tống Thanh Danh: `python3 student-bundle.py run --bundle . --out ../ket-qua-nhom-01`.
  - Thời gian theo `smoke.json`: `2026-10-01 14:44:36 → 14:45:03 (UTC+7)` / `07:44:36 → 07:45:03 (UTC)`.
  - Máy: macOS Apple Silicon (`arm64`); Docker Desktop server 29.8.1 (`docker version` lúc chạy); môi trường chạy đóng gói (container) `linux/arm64` chạy trực tiếp trên kiến trúc máy, không cần giả lập theo `smoke.json` (`image.architecture` và `runtime.architecture` đều `arm64`); giới hạn 4 CPU / 4 GB.
- Tên bản môi trường đóng gói (image tag), mã nhận diện bản đó (image ID) và phiên bản kho mã (repo):
  - `day13-pointpillars:lc-20261001-arm64` / `sha256:dd6999ad5dd67962fdca18eb526132c08ce105980c9d89475d66cb1a5193f8a1`.
  - Gói phát hành `student-prelabel-v1`, file `student-prelabel-arm64.zip`; SHA256 khớp `SHA256SUMS.txt` (`shasum -a 256 -c` báo OK lúc tải).
  - Mã phiên bản của kho mã ghi trong `smoke.json`: `0831856d921609312d42c7582c366e5a311bb7b1`, kèm `working_tree_dirty: true`, nghĩa là mã đang dùng có thay đổi so với bản đã lưu trong Git. Vì vậy, dùng mã hash của các tệp lệnh để xác định bản mã đã chạy: `preannotate_sha256 = 65edf6ac95926f799c27f2c30c8595b0da4e8e2429f0c1a9aedf5c9e1fceb5ca`, `helper_sha256 = c177fc008f79223e94b8c06423d3a806e422787d48c8d39f1c5fd3ce2c4eeaa7`.
- File đám mây điểm PCD được cấp / mã khung dữ liệu frame_id; nơi được phép chạy; mã nhận diện dữ liệu (fingerprint) nếu LC cấp:
  - `input/demo.pcd`, frame_id `demo`, 17 238 điểm.
  - Nguồn KITTI / MMDetection3D demo `000008.bin` (`source_sha256 = 3b9de6cc…20d902d1`), giấy phép CC BY-NC-SA 3.0, chạy thí nghiệm học thuật trên máy nhóm.
  - Chuyển đổi theo `input/provenance.json`: giữ nguyên x/y, dịch z **+1.73 m**, bỏ reflectance (mức phản xạ của điểm đo), đặt RGB = 0 làm giá trị màu tạm.
  - SHA256 PCD `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`, trùng `input_sha256` trong `smoke.json` và `pcd_sha256` trong `provenance.json`.
- Checkpoint (bản mô hình đã được huấn luyện và lưu lại): PointPillars KITTI `/opt/PointPillars/pretrained/epoch_160.pth`; SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi:
  - Vùng phía trước xe mà mô hình xét (front-window KITTI), tính trong hệ tọa độ của mô hình: `x ∈ (0.0, 69.12)`, `y ∈ (−39.68, 39.68)`, `z ∈ (−3.0, 1.0)` m. `in_range` trong `preannotate.py` dùng khoảng **mở**, nên điểm nằm đúng biên cũng bị loại.
  - Ngưỡng score (điểm tin cậy do mô hình đưa ra) là `0.3`.
  - Không chạy lượt `yaw180` để xét toàn cảnh, nên phía sau xe nằm ngoài phạm vi inference (bước mô hình dự đoán hộp bao quanh vật thể) (`rear = 0` ở cả ba lượt).
- Giả định về kênh dữ liệu thứ tư, intensity (mức phản xạ), và nguồn z_ground (độ cao mặt đất ước lượng):
  - PCD không có mức phản xạ thật. Phần mã chuyển dữ liệu cho mô hình (adapter) đọc đám mây điểm hai lần, mỗi lần gán cùng một mức phản xạ cho mọi điểm: `0.0` giữ class (loại vật thể) `vehicles` (xe), `0.7` giữ `pedestrian` (người đi bộ) và `two-wheels` (xe hai bánh).
  - Hộp `pedestrian`/`two-wheels` có tâm nằm trong **footprint XY** (vùng hộp chiếm trên mặt phẳng nhìn từ trên xuống) của một hộp `vehicles` thì bị loại.
  - `z_ground = 0.075 m` do `estimate_ground` chọn khoảng độ cao có nhiều điểm nhất trên biểu đồ đếm điểm theo z (histogram, mỗi khoảng hay bin rộng 0.05 m) của toàn bộ lần quét. Đây không phải mốc mặt đường được đo chính xác.

---

## Ba lượt inference thật

Các nhận xét dưới đây so sánh **prediction với prediction**, tức là so các kết quả dự đoán với nhau. Không có ground truth (nhãn đúng đã được kiểm chứng), nên dùng B làm baseline (mốc so sánh) theo đề, chứ không coi B là đáp án. Đáy hộp tính bằng `z − height/2`, vì JSON lưu z là độ cao tâm hộp trong hệ tọa độ của dữ liệu nguồn.

Trong bảng, delta là lượng dịch độ cao trước khi đưa dữ liệu vào mô hình. Pillar là ô dạng cột dùng để gom các điểm; Pillar XY là kích thước ô trên mặt phẳng ngang. mean_z là độ cao trung bình của các tâm hộp. Side là ảnh nhìn từ bên cạnh. Yaw là góc quay của hộp trên mặt phẳng ngang; rad là đơn vị đo góc. Các ký hiệu l, w, h lần lượt là chiều dài, chiều rộng và chiều cao hộp.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png`<br>`run-A/summary.csv` | Chỉ có 1 hộp `vehicles` tại x = 13.15, y = −0.45, score 0.32 (sát ngưỡng 0.3), đáy ≈ −0.40 m (thấp hơn vạch z = 0 trên ảnh Side). Ít hơn B 12 hộp dự đoán; không có `pedestrian` hay `two-wheels`. Gần đó, B có hộp `vehicles` tại x = 14.77, y = −1.08; hai yaw lệch khoảng 2.97 rad (2.67 so với −0.30). Chưa xác nhận được đây là cùng một xe hay hướng nào đúng. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png`<br>`run-B/summary.csv` | 10 `vehicles` (score 0.50–0.93), 1 `two-wheels` (x = 10.32, score 0.38), 2 `pedestrian` (x = 18.67 và 34.03, score 0.34 và 0.32). Kích thước hộp xe nằm trong l 3.1–4.2 m, w 1.5–1.7 m, h 1.5–1.8 m. Đa số hộp gần có đáy trong khoảng −0.05 đến 0.15 m. Ngoại lệ: `vehicles` x = 9.38 (đáy 0.65 m) và `two-wheels` x = 10.32 (đáy 0.47 m). Ba xe xa x ≈ 33.7 / 41.0 / 55.6 m có đáy 0.46 / 0.41 / 0.32 m; điểm mặt đường vùng đó trên ảnh Side cũng cao hơn. Chưa rõ hộp bị đặt lơ lửng hay mặt đường ở đó cao hơn. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png`<br>`run-C/summary.csv` | Cả 6 hộp là `pedestrian`, không có `vehicles` hay `two-wheels` vượt ngưỡng. Một số hộp nằm gần hộp dự đoán của B: (9.11, 0.40) gần `vehicles` B (8.09, 1.21); (13.24, −0.95) gần `vehicles` B (14.77, −1.08); (10.46, 4.93) gần `two-wheels` B (10.32, 5.25). Ba hộp có l ≈ 1.03–1.07 m. Nghi là nhận nhầm loại vật thể; cần xem trong không gian 3D và ảnh camera để xác nhận. Vì không còn hộp `vehicles`, quy tắc loại hộp người/xe hai bánh nằm trong vùng hộp xe nhìn từ trên xuống không có tác dụng. |

---

### Phân tích kỹ thuật chuyên sâu

- **A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**
  **Khác.** Input là dữ liệu đưa vào, model là mô hình, còn output là kết quả trả ra. Đổi dữ liệu *trước* khi mô hình chạy sẽ đổi những gì mô hình nhìn thấy. Dịch kết quả sau khi chạy chỉ di chuyển những hộp đã có.
  1. *Giả định của checkpoint:* `delta = 1.73` là độ cao cảm biến (sensor) mà checkpoint KITTI giả định trong bài (PRE-LABEL.md). Sau khi chuẩn hóa mặt đường, mặt đường nằm ở z_model ≈ −1.73. Với delta = 0, `z_model = z_source − 0.075`, mặt đường nằm quanh 0, nên độ cao các điểm trong ô pillar khác với lúc mô hình được huấn luyện.
  2. *Cắt ROI (vùng dữ liệu được giữ lại để mô hình xét):* ROI giữ `z_model < 1.0`. Ở lượt A, mọi điểm có `z_source ≥ 1.075 m` bị loại trước khi mô hình dự đoán. Ảnh Side cho thấy nhiều điểm ở độ cao 1–2.5 m không được đưa vào mô hình.
  3. *Bằng chứng:* A có 1 hộp dự đoán, B có 13. Cộng 1.73 m vào kết quả của A chỉ di chuyển đúng 1 hộp đó, không tạo thêm hộp và không đổi góc quay hay loại vật thể của hộp. Chênh lệch 12 hộp dự đoán chỉ có thể đến từ việc đổi dữ liệu đầu vào.

- **B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**
  - Ở đây, cách biểu diễn dữ liệu đưa vào mô hình đã thay đổi. Đây không phải lỗi quên cộng lại z để đưa hộp về hệ tọa độ nguồn.
  - Pillar 0.32 m làm độ phân giải của lưới nhìn từ trên xuống (BEV) giảm một nửa theo mỗi trục so với 0.16 m mà checkpoint dùng. Kết quả: số hộp dự đoán giảm từ 13 xuống 6, không còn `vehicles` hay `two-wheels` có điểm tin cậy vượt 0.3, và một số hộp `pedestrian` nằm gần vị trí `vehicles` hoặc `two-wheels` của B.
  - mean_z gần nhau (1.034 và 1.091) không chứng minh các hộp của C đúng về độ cao, vì các hộp và loại vật thể đã khác hẳn.
  - **Chưa đủ bằng chứng để nói cấu hình nào đúng hơn**: không có ground truth và chỉ có 1 khung dữ liệu (frame). Chỉ có thể nói pillar 0.16 khớp cấu hình checkpoint nên B được giữ làm mốc cho bước kiểm tiếp. Nhiều hộp hơn hay điểm tin cậy cao hơn không tự chứng minh kết quả tốt hơn.

- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  - *ROI:* front-window chỉ xét `0 < x < 69.12`, `|y| < 39.68` và `z_model < 1.0`. Vật thể phía sau xe hoặc ngoài cửa sổ nằm **ngoài phạm vi dự đoán, không tính là miss (bỏ sót vật thể)** trong bài so cấu hình này. `rear = 0` không có nghĩa là phía sau không có vật thể.
  - *Ảnh Side (hình chiếu x–z của toàn cảnh):* phù hợp để kiểm độ cao, đáy hộp so với mặt đường và lỗi lệch cả batch (cả lô hộp được xử lý cùng nhau).
    - Hình chiếu gộp trục y, nên các hộp khác làn có thể chồng lên nhau (ví dụ x ≈ 6–11 m ở B).
    - Hình chữ nhật trên ảnh Side được vẽ **không sử dụng yaw**, nên không thể dùng ảnh này để xác định hướng, độ lệch ngang hay hộp trùng.
    - Chênh yaw giữa A và B chỉ thấy được khi đọc JSON.
    - Cần xem Top/Side/Front (ảnh nhìn từ trên, bên cạnh và phía trước) cùng ảnh camera trước khi duyệt vị trí, kích thước và hướng của từng hộp.

- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - **Cả ba JSON đều chưa đủ cơ sở để import, tức nhập vào công cụ làm nhãn.** A và C chỉ dùng để so cấu hình: A có ít hộp dự đoán và đáy dưới z = 0; C chỉ có `pedestrian`, nghi nhận nhầm loại vật thể.
  - **B** chỉ là pre-label, tức nhãn gợi ý ban đầu để người làm nhãn kiểm tra. Cần kiểm tiếp:
    1. 2 `pedestrian` và 1 `two-wheels` có score 0.32–0.38, sát ngưỡng.
    2. Các hộp có đáy cao (x = 9.38, x = 10.32 và ba xe x > 30 m): kiểm mặt đường cục bộ trước khi kéo hộp xuống.
    3. Yaw của từng xe trên BEV.
    4. Vùng ngoài front-window chưa được xét.
    5. Rà đủ 5 loại vật thể trong bộ quy định nhãn (schema) theo `LABEL_GUIDELINE.md`.
  - Các dự đoán này thuộc khung dữ liệu KITTI demo, **không nhập vào bất kỳ tác vụ gán nhãn (job) Robotaxi nào**.

---

## Ca QC có kiểm soát — không import CVAT

QC là bước kiểm tra chất lượng nhãn. Cả ba ca được tạo từ dự đoán thật của lượt B: `source_prediction_sha256 = 51f49ac6…6e26016`, trùng `prediction_sha256` của `run-B` trong `smoke.json`. Theo `qc-cases/manifest.json`: `delta_m = 1.73`, `z_ground_m = 0.075`, `height_offset_m = 1.805`. Các ca này chỉ cài lỗi độ cao z, không chứng minh các hộp khác là đúng.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| `case-correct` | 0 / 13 | 0 m | Không đổi | Không có lỗi z được cài vào. Kết quả vẫn chỉ do mô hình dự đoán, không phải đáp án; tiếp tục kiểm từng hộp như B. | Hộp 1 `vehicles` x = 8.09: z = 0.921 m, giống `run-B`. |
| `case-batch-z` | 13 / 13 | −1.805 m = −(delta + z_ground) | Không đổi; chỉ z | **Dừng cả lô hộp, kiểm pipeline (chuỗi bước xử lý dữ liệu).** Mọi hộp lệch cùng một lượng đúng bằng −(delta + z_ground), khớp với dấu hiệu quên bước nghịch `z_to_source`. Sửa phép chuyển tọa độ (transform) rồi chạy lại; không bắt người gán nhãn (annotator) kéo từng hộp. | Hộp 1: 0.921 → −0.884; hộp 2: 0.900 → −0.905. Cả 13 tâm hộp nằm dưới z = 0; 6/13 hộp vẫn có đỉnh trên z = 0 (`side-batch-z.png`). |
| `case-one-box-z` | 1 / 13 | −1.805 m (chỉ hộp 1) | Không đổi; 12 hộp còn lại giữ nguyên z | **Không dừng cả lô hộp; kiểm riêng hộp lệch.** Lỗi không xảy ra đồng loạt nên không phải dấu hiệu lỗi chuyển tọa độ của cả lô. 12 hộp còn lại không vì thế mà được coi là đúng, vẫn kiểm như B. Xem hộp lệch từ nhiều góc nhìn và trên ảnh camera rồi sửa z hoặc xóa; thiếu bằng chứng thì ghi chưa chắc. | Hộp 1 (x = 8.09): 0.921 → −0.884; hộp 2 vẫn 0.900. `side-one-box-z.png`: chỉ 1 hộp ở x ≈ 6–10 m tụt xuống. |

*Ghi chú: cả ba ca là biến đổi có chủ đích từ B, chỉ dùng để luyện nhận lỗi, không phải ground truth và không import vào CVAT.*

---

## Nhận xét cá nhân

> Bản nháp do nhóm tổng hợp từ kết quả chạy thật. Mỗi thành viên phải tự đọc, sửa lại bằng lời của mình và xác nhận đúng phần mình đã làm trước khi nộp LC.

### Tống Thanh Danh (MSSV: 2A202602299)
- **Vai trò thực hiện:** Tôi làm Operator ở lượt A: tải gói `student-prelabel-arm64.zip`, kiểm SHA256, giải nén, gõ lệnh runner trên Mac arm64 và theo dõi bước `run-A`. Ở lượt B, tôi làm Log & Report Recorder (người ghi kết quả và báo cáo), ghi số hộp, loại vật thể và mean_z của B vào bảng. Ở lượt C, tôi làm Geometry Inspector (người kiểm vị trí, kích thước và hướng hộp), xem `run-C/side-demo-delta-1.73-voxel-0.32.png`.
- **Quan sát kỹ thuật có dẫn chứng:**
  - Tôi kiểm `smoke.json` và thấy image cùng môi trường thực thi (runtime) đều là `arm64`, tức chạy trực tiếp trên kiến trúc máy.
  - Ba lượt dùng cùng `input_sha256` và cùng checkpoint SHA256, nên khác biệt A/B/C đến từ delta và pillar.
  - Trên ảnh Side của C, tôi thấy cả 6 hộp đều cao và hẹp. Một số nằm ở x ≈ 9–13 m, nơi B có hộp `vehicles`.
- **Diễn giải phép biến đổi z thuận/nghịch:** Tôi hiểu chiều thuận `z_model = z_source − z_ground − delta` là đưa đám mây điểm về hệ tọa độ mà checkpoint KITTI giả định, với mặt đường ≈ −1.73 m. Chiều nghịch `z_source = z_model + z_ground + delta` đưa hộp về lại hệ tọa độ nguồn. Hai chiều phải dùng cùng delta và z_ground, ở đây là 1.73 và 0.075.
- **Quyết định lỗi batch và hành động:** Nếu mọi hộp cùng lệch −1.805 m như `case-batch-z`, tôi dừng cả lô, kiểm lại mã chuyển tọa độ rồi chạy lại. Tôi không chuyển các hộp này sang bước sửa tay.
- **Điều chưa chắc chắn:** Tôi mới chạy trên một máy và một khung dữ liệu. Tôi chưa biết số hộp có giữ nguyên trên máy amd64 không; README ghi không bắt buộc số hộp phải trùng giữa các máy.

### Nguyễn Anh Tuấn (MSSV: 2A202602282)
- **Vai trò thực hiện:** Ở lượt A, tôi làm Config Inspector (người kiểm cấu hình): đối chiếu delta = 0, pillar 0.16, score 0.3 và đọc JSON A. Sang lượt B, tôi làm Operator, theo dõi bước `run-B`, kiểm `status: passed`, 13 hộp và `prediction_sha256`. Lượt C, tôi làm Log & Report Recorder, ghi số liệu `run-C/summary.csv`.
- **Quan sát kỹ thuật có dẫn chứng:** Tôi chú ý đến góc quay của hộp. Ở `run-A`, hộp duy nhất tại (x = 13.15, y = −0.45) có yaw 2.67. Hộp `vehicles` gần đó ở `run-B`, tại (x = 14.77, y = −1.08), có yaw −0.30. Hai góc lệch khoảng 2.97 rad. Tôi chỉ thấy chênh lệch này khi đọc JSON, vì ảnh Side không vẽ góc quay.
- **Diễn giải phép biến đổi z thuận/nghịch:**
  - Theo cách tôi hiểu, chiều thuận trừ `z_ground + delta` trước khi mô hình dự đoán. Chiều nghịch cộng lại đúng lượng đó sau khi dự đoán.
  - `delta = 1.73` là giả định của checkpoint KITTI sau khi đã chuẩn hóa mặt đường, không phải độ cao cảm biến của một xe bất kỳ. Với cảm biến hoặc mô hình khác, tôi phải dùng giả định của checkpoint sẽ dùng, không tự áp 1.73.
  - Đổi delta là đổi dữ liệu đầu vào của mô hình. Vì vậy, không bảo đảm kết quả chỉ lệch một lượng cố định.
- **Quyết định lỗi batch và hành động:** Nếu mọi hộp lệch cùng một lượng và cùng chiều, tôi coi đó là dấu hiệu lỗi chuyển tọa độ và dừng chuỗi xử lý. Nếu chỉ một hộp lệch, tôi kiểm riêng hộp đó.
- **Điều chưa chắc chắn:** Tôi chưa xác nhận hộp A và hộp B (14.77, −1.08) có phải cùng một xe không, vì tâm cách nhau khoảng 1.7 m. Tôi cũng chưa biết hướng nào đúng. Cần xem BEV/3D và ảnh camera.

### Tô Quang Hưng (MSSV: 2A202602305)
- **Vai trò thực hiện:** Tôi bắt đầu ở vai Geometry Inspector lượt A, xem `run-A/side-demo-delta-0-voxel-0.16.png`. Ở lượt B, tôi làm Config Inspector, xác nhận B chỉ đổi delta so với A. Đến lượt C, tôi làm Operator, theo dõi bước `run-C`, kiểm `status: passed` và 6 hộp.
- **Quan sát kỹ thuật có dẫn chứng:**
  - Điều tôi chú ý ở `run-C` là cả 6 hộp đều được nhận là `pedestrian`.
  - Hộp (9.11, 0.40) và (13.24, −0.95) nằm gần hộp `vehicles` của B; hộp (10.46, 4.93) gần `two-wheels` của B.
  - Ba hộp có l ≈ 1.03–1.07 m. Tôi nghi nhận nhầm loại vật thể, nhưng chưa khẳng định khi chưa xem 3D.
- **Diễn giải phép biến đổi z thuận/nghịch:**
  - Tôi phân biệt hai bước như sau: chiều thuận `z_model = z_source − z_ground − delta` thực hiện trước khi mô hình chạy; chiều nghịch `z_source = z_model + z_ground + delta` thực hiện trên hộp sau khi mô hình chạy.
  - Lượt C giữ nguyên delta = 1.73 và z_ground như B, nên khác biệt B/C đến từ pillar, không phải phép chuyển z.
  - mean_z gần nhau không chứng minh các hộp của C đúng về độ cao.
- **Quyết định lỗi batch và hành động:** C là lượt thử đổi pillar theo đề, không phải ca quên chuyển z ngược. Tôi không dùng C làm nhãn gợi ý và không sửa tay 6 hộp này. Nếu gặp lỗi cả lô thật, tức mọi hộp cùng lệch −(delta + z_ground), tôi sẽ dừng chuỗi xử lý và kiểm phép chuyển tọa độ.
- **Điều chưa chắc chắn:** Tôi chưa rõ phần mô hình dự đoán `vehicles` (head) ở pillar 0.32 có hộp nào ngay dưới ngưỡng 0.3 không. File kết quả chỉ lưu hộp đã qua ngưỡng nên chưa trả lời được điều này.

### Tưởng Đức Tâm (MSSV: 2A202602249)
- **Vai trò thực hiện:** Lượt A, tôi làm Log & Report Recorder: ghi `run-A/summary.csv`, đối chiếu `smoke.json` và `qc-cases/manifest.json`. Lượt B, tôi làm Geometry Inspector, xem `run-B/side-demo-delta-1.73-voxel-0.16.png` và ba ảnh QC. Lượt C, tôi làm Config Inspector, xác nhận C chỉ đổi pillar 0.16 → 0.32 so với B.
- **Quan sát kỹ thuật có dẫn chứng:**
  - Tôi đối chiếu mã và thấy `source_prediction_sha256` trong bản kê các ca QC (manifest) trùng `prediction_sha256` của `run-B` (`51f49ac6…`).
  - Trong `case-batch-z`, 13/13 hộp có Δz = −1.805 m; Δz là lượng thay đổi độ cao. Cả 13 tâm nằm dưới z = 0 nhưng 6 hộp vẫn có đỉnh trên z = 0.
  - Trong `case-one-box-z`, chỉ hộp 1 có Δz = −1.805 m, còn 12 hộp khác có Δz = 0.
- **Diễn giải phép biến đổi z thuận/nghịch:**
  - Tôi theo dõi lượng trừ và cộng: chiều thuận trừ `z_ground + delta` = 1.805 m trước khi mô hình dự đoán; chiều nghịch `z_to_source` cộng lại 1.805 m cho hộp.
  - Δz = −1.805 m = −(delta + z_ground) trên mọi hộp đúng bằng phần còn thiếu của chiều nghịch. Vì vậy, tôi dùng đây làm dấu hiệu nhận biết lỗi "quên `z_to_source`".
- **Quyết định lỗi batch và hành động:** Tôi nhìn vào tỉ lệ hộp bị lệch và xem chúng có lệch đều nhau không. Nếu 13/13 hộp cùng lệch một giá trị thì tôi dừng cả lô; nếu 1/13 thì tôi kiểm riêng hộp đó.
- **Điều chưa chắc chắn:** Trên `run-B`, tôi thấy ba xe xa (x ≈ 33.7 / 41.0 / 55.6 m) có đáy 0.46 / 0.41 / 0.32 m. Đường dốc hoặc việc dùng một giá trị z_ground ước lượng cho toàn bộ lần quét chỉ là giả thuyết. Tôi cần kiểm mặt đường ngay tại từng vị trí trên 3D trước khi kết luận hộp bị đặt lơ lửng.

---

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
