
# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu.
Đây là kiểm tra formative; không ghi điểm của người khác.

---

## Nhóm và provenance

- **Mã nhóm/phòng:** [CẦN ĐIỀN]
- **Thành viên:** xem `TEAMMATES.md`
  - Bùi Quang Thái
  - Võ Quốc Dinh
  - Hà Quang Huy
- **Trạng thái:** `executed-by-group`
- **Người trực tiếp thao tác máy chạy runner:** Bùi Quang Thái
- **Ngày chạy:** 01/10/2026
- **Thời gian chạy:** khoảng 16:01–16:02, UTC+7
- **Hệ điều hành máy host:** Windows
- **Runtime Docker:** Docker Desktop / Linux container
- **Architecture:** `amd64 / x86_64`
- **Python host:** Python 3.13.14
- **Docker Server:** 29.8.0
- **Giới hạn container:** 4 CPU / 4 GB RAM

### Docker image

- **Source tag:** `day13-pointpillars:lc-20261001-amd64`
- **Image ID:**

`sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`

### Repo / code

- **Repo revision:**

`0831856d921609312d42c7582c366e5a311bb7b1`

- **working_tree_dirty:** `true`
- **preannotate SHA256:**

`65edf6ac95926f799c27f2c30c8595b0da4e8e2429f0c1a9aedf5c9e1fceb5ca`

- **helper SHA256:**

`c177fc008f79223e94b8c06423d3a806e422787d48c8d39f1c5fd3ce2c4eeaa7`

### PCD / dữ liệu đầu vào

- **Dataset:** KITTI Student demo
- **Frame ID:** `demo`
- **Số điểm:** 17,238
- **Source frame:** `pcd-source`
- **Input SHA256:**

`3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`

- **Nơi chạy:** máy nhóm local bằng Docker Desktop
- **Robotaxi:** không sử dụng làm đầu vào cho phần A/B/C

### Checkpoint

- **Model:** PointPillars pretrained KITTI
- **Checkpoint path:**

`/opt/PointPillars/pretrained/epoch_160.pth`

- **Checkpoint SHA256:**

`482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`

### Cấu hình chung

- **ROI:** `front-window`
- **Score threshold:** `0.3`
- **z_ground ước lượng:** `0.075 m`
- **Kênh thứ tư / intensity:** reflectance thật của KITTI demo đã bị bỏ trong bản PCD chuyển đổi. RGB = 0 chỉ là placeholder; pipeline sử dụng adapter kênh hằng và không coi đây là intensity LiDAR thật.
- **Hệ tọa độ:** x/y giữ theo nguồn KITTI; z của PCD demo đã được dịch +1.73 m trong quá trình chuẩn bị dữ liệu. `z_ground` vẫn được ước lượng từ PCD khi chạy.

---

# Ba lượt inference thật

A, B và C là ba lượt inference trên cùng một PCD bằng cùng checkpoint PointPillars pretrained.

- A/B chỉ thay đổi `delta`.
- B/C chỉ thay đổi kích thước Pillar XY.
- Không dùng A/C để quy nguyên nhân vì hai biến cùng thay đổi.
- Số hộp hoặc `mean_z` không phải tiêu chí trực tiếp để kết luận cấu hình nào tốt hơn.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV                                                                           | Quan sát có bằng chứng                                             |
| ------ | ----: | --------: | -------: | -----: | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| A      |     0 |      0.16 |        1 |  0.330 | `run-A/boxes-*.json`, `run-A/side-*.png`, `run-A/summary.csv`                          | Chỉ có 1 prediction thuộc class`vehicles`.                        |
| B      |  1.73 |      0.16 |       13 |  1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-*.png`, `run-B/summary.csv` | Có 13 prediction: 10`vehicles`, 2 `pedestrian`, 1 `two-wheels`. |
| C      |  1.73 |      0.32 |        6 |  1.091 | `run-C/boxes-*.json`, `run-C/side-*.png`, `run-C/summary.csv`                          | Có 6 prediction và cả 6 thuộc class`pedestrian`.                 |

---

## A/B — chỉ đổi delta

### Cấu hình

Lượt A:

- `delta = 0 m`
- Pillar XY = `0.16 m`

Lượt B:

- `delta = 1.73 m`
- Pillar XY = `0.16 m`

### Kết quả

- A: 1 hộp
- B: 13 hộp
- `mean_z` A: 0.330
- `mean_z` B: 1.034

Phân bố class:

- A: 1 `vehicles`
- B:
  - 10 `vehicles`
  - 2 `pedestrian`
  - 1 `two-wheels`

### Quan sát

Khi thay `delta` từ 0 m lên 1.73 m và giữ Pillar XY = 0.16 m, số prediction thay đổi từ 1 lên 13.

Thành phần class cũng thay đổi rõ rệt. A chỉ có một prediction thuộc `vehicles`, trong khi B xuất hiện cả `vehicles`, `pedestrian` và `two-wheels`.

Điều này cho thấy thay đổi z trước inference làm thay đổi input mà model nhận được và có thể làm thay đổi cả số lượng, class và vị trí prediction.

Không thể hiểu B đơn giản là lấy các cuboid của A rồi dịch toàn bộ chúng thêm 1.73 m sau inference.

### Kết luận

Chưa đủ bằng chứng để kết luận B chính xác hơn A chỉ vì B có nhiều prediction hơn.

Muốn đánh giá từng cuboid vẫn cần đối chiếu với PCD, nhiều góc nhìn và reference phù hợp.

### Điều chưa chắc

Ảnh Side là hình chiếu x-z nên các đối tượng có tọa độ y khác nhau có thể chồng lên nhau.

Vì vậy ảnh Side một mình không đủ để:

- xác nhận tâm y;
- xác nhận yaw;
- đánh giá đầy đủ footprint;
- kết luận chất lượng từng cuboid.

---

## B/C — chỉ đổi Pillar XY

### Cấu hình

Lượt B:

- `delta = 1.73 m`
- Pillar XY = `0.16 m`

Lượt C:

- `delta = 1.73 m`
- Pillar XY = `0.32 m`

### Kết quả

- B: 13 hộp
- C: 6 hộp
- `mean_z` B: 1.034
- `mean_z` C: 1.091

Phân bố class:

B:

- 10 `vehicles`
- 2 `pedestrian`
- 1 `two-wheels`

C:

- 6 `pedestrian`

### Quan sát

Khi giữ nguyên `delta = 1.73 m` nhưng tăng Pillar XY từ 0.16 m lên 0.32 m, số prediction giảm từ 13 xuống 6.

Thành phần class cũng thay đổi mạnh. Các prediction `vehicles` và `two-wheels` có ở B không còn xuất hiện trong kết quả C, trong khi C gồm 6 prediction `pedestrian`.

Điều này cho thấy kích thước pillar ảnh hưởng đến cách point cloud được chia/gom trên mặt phẳng x-y trước khi đưa vào mạng, từ đó làm thay đổi kết quả inference.

### Kết luận

Không đủ bằng chứng để kết luận B tốt hơn C hoặc C tốt hơn B chỉ dựa vào:

- số lượng box;
- class;
- `mean_z`;
- confidence.

Lượt C vẫn sử dụng cùng checkpoint pretrained. Đây là thí nghiệm thay đổi biểu diễn đầu vào chứ không phải model được train lại riêng cho Pillar XY = 0.32 m.

---

## Giới hạn ROI và ảnh Side

Các lượt A/B/C sử dụng cùng:

- checkpoint KITTI;
- score threshold = 0.3;
- front-window ROI.

Đối tượng nằm ngoài ROI không được dùng làm bằng chứng rằng model bỏ sót trong phép so A/B/C.

Ảnh Side là hình chiếu x-z toàn scene. Nhiều đối tượng khác nhau theo trục y có thể chồng lên nhau trên cùng ảnh.

Do đó:

- Side phù hợp để quan sát cao độ và lỗi z rõ ràng.
- Side không đủ để đánh giá chính xác center theo y.
- Side không đủ để xác nhận yaw.
- Side không đủ để đánh giá toàn bộ footprint.
- Khi chỉnh cuboid Robotaxi thật vẫn cần kiểm Top / Side / Front / free rotation và camera.

---

## JSON nào chưa đủ cơ sở để import?

Các JSON trong `run-A`, `run-B`, `run-C` là prediction trên KITTI demo phục vụ thí nghiệm pipeline.

Không JSON nào trong phần Student demo được import vào job Robotaxi.

Các prediction A/B/C:

- không phải ground truth;
- không phải reference;
- không chứng minh annotation đúng;
- không cùng frame với Robotaxi.

Prediction Robotaxi phải được lấy bằng nút **Nạp pre-label cho job này** trên portal để hệ thống lấy đúng prediction Robotaxi do LC chạy trước và kiểm đúng frame/schema.

---

# Ca QC có kiểm soát — không import CVAT

Ba ca QC được tạo từ prediction thật của lượt B.

### Source

- **Source prediction:** `boxes-demo-delta-1.73-voxel-0.16.json`
- **Source prediction SHA256:**

`c2a8db247353f0ef00299ff50acff997b7b4a87bdcfcefe792815acb5652cc80`

- **Frame ID:** `demo`
- **Source frame:** `pcd-source`
- **Dataset:** KITTI
- **delta:** `1.73 m`
- **z_ground:** `0.075 m`

Tổng offset:

`height_offset = delta + z_ground`

`height_offset = 1.73 + 0.075`

`height_offset = 1.805 m`

- **Tổng số box:** 13
- **training_only:** `true`

Các ca QC dưới đây là biến đổi có kiểm soát từ prediction B.

Chúng:

- không phải ba lần inference mới;
- không phải ground truth;
- không phải reference;
- không được import vào CVAT.

| Ca                 | Số hộp lệch z / tổng hộp | Lượng lệch       | Class/x/y/yaw có đổi?                      | Quyết định                                                                 | Bằng chứng                                                                                   |
| ------------------ | ----------------------------- | ------------------- | --------------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `case-correct`   | 0 / 13                        | 0 m                 | Không                                        | Dùng làm mốc đối chiếu; không coi là ground truth                     | `case-correct.json`, `side-correct.png`; prediction nguồn được giữ nguyên            |
| `case-batch-z`   | 13 / 13                       | `-1.805 m` theo z | Không; chỉ z của toàn bộ box bị dịch   | **Dừng batch**, kiểm transform/pipeline; không sửa tay từng cuboid | `case-batch-z.json`, `side-batch-z.png`; mọi box cùng bị dịch xuống 1.805 m           |
| `case-one-box-z` | 1 / 13                        | `-1.805 m` theo z | Không; chỉ z của box đầu tiên bị dịch | **Kiểm riêng object**, không kết luận toàn pipeline sai           | `case-one-box-z.json`, `side-one-box-z.png`; chỉ box đầu tiên bị dịch xuống 1.805 m |

---

## case-correct

`case-correct` giữ nguyên prediction nguồn của run-B.

Ca này được dùng làm mốc để đối chiếu với hai trường hợp lỗi z.

Tên `correct` ở đây chỉ có nghĩa là phép chuyển đổi nguồn không bị cố ý làm sai.

Không được hiểu:

`case-correct = ground truth`

hoặc:

`case-correct = cuboid đúng hoàn toàn`.

Nó vẫn chỉ là prediction của PointPillars.

---

## case-batch-z

Trong `case-batch-z`:

- tổng số box: 13;
- số box bị dịch z: 13;
- độ dịch: `-1.805 m`;
- class giữ nguyên;
- x giữ nguyên;
- y giữ nguyên;
- yaw giữ nguyên.

### Nhận định

Việc toàn bộ 13 box cùng lệch z một lượng giống nhau là dấu hiệu phù hợp với lỗi transform/pipeline toàn batch hơn là lỗi annotation riêng từng object.

### Quyết định

**Dừng sửa tay từng cuboid.**

Kiểm lại:

1. hệ tọa độ nguồn/model;
2. `z_ground`;
3. `delta`;
4. phép forward conversion;
5. phép inverse conversion;
6. input/frame đang dùng.

Nếu xác nhận lỗi pipeline, yêu cầu tạo lại prediction bằng pipeline đúng thay vì sửa thủ công 13 box.

---

## case-one-box-z

Trong `case-one-box-z`:

- tổng số box: 13;
- số box bị dịch z: 1;
- box bị tác động: box đầu tiên;
- độ dịch: `-1.805 m`;
- 12 box còn lại giữ nguyên.

### Nhận định

Một box bị lệch z trong khi các box khác không thay đổi chưa đủ cơ sở để kết luận toàn pipeline sai.

### Quyết định

Kiểm riêng object bằng nhiều view:

- Top;
- Side;
- Front;
- free rotation;
- cụm điểm LiDAR;
- đáy box;
- mặt đường cục bộ.

Nếu các box còn lại vẫn hợp lý thì ưu tiên xem đây là lỗi object-local hoặc trường hợp cần kiểm thêm.

---

# Trạng thái chạy

`smoke.json`:

- `status: passed`

Các bước:

- `docker-load`: passed
- `run-A`: passed
- `run-B`: passed
- `run-C`: passed
- `qc-cases`: passed

### Thời gian chạy

- Docker load: khoảng 48.32 giây
- Run A: khoảng 4.95 giây
- Run B: khoảng 4.48 giây
- Run C: khoảng 2.88 giây
- QC cases: khoảng 1.52 giây

Thời gian trên chỉ mô tả lần chạy thực tế trên máy nhóm, không dùng làm metric chất lượng model.

---

# Nhận xét cá nhân

## 1. Bùi Quang Thái

### Vai trò

- Trực tiếp chuẩn bị và vận hành Student bundle trên máy nhóm.
- Kiểm Python và Docker Desktop.
- Kiểm đúng architecture `amd64`.
- Chạy runner A/B/C.
- Kiểm `smoke.json`.
- Đọc log và `summary.csv`.
- Kiểm `qc-cases/manifest.json`.
- Tổng hợp báo cáo nhóm.

### Quan sát A/B

A sử dụng:

- `delta = 0`
- Pillar XY = `0.16`

và cho 1 prediction thuộc `vehicles`.

B sử dụng:

- `delta = 1.73`
- Pillar XY = `0.16`

và cho 13 prediction:

- 10 `vehicles`
- 2 `pedestrian`
- 1 `two-wheels`

Em nhận thấy khi chỉ thay delta, kết quả inference thay đổi rõ rệt về số lượng và class prediction.

Điều này cho thấy thay đổi z trước inference có thể ảnh hưởng trực tiếp đến output của model, không thể coi đây đơn thuần là dịch cùng các box cũ theo z.

### Quan sát B/C

B sử dụng Pillar XY = 0.16 m và tạo 13 prediction.

C sử dụng Pillar XY = 0.32 m và tạo 6 prediction.

Cả số lượng lẫn thành phần class đều thay đổi.

Điều này cho thấy kích thước pillar ảnh hưởng đến biểu diễn point cloud đầu vào và từ đó ảnh hưởng kết quả PointPillars.

Em không kết luận B tốt hơn C chỉ vì B có nhiều box hơn.

### Diễn giải phép z thuận/ngược

Pipeline sử dụng:

`z_model = z_source - z_ground - delta`

Khi trả prediction về hệ nguồn:

`z_source = z_model + z_ground + delta`

Trong B/C:

- `delta = 1.73 m`
- `z_ground ≈ 0.075 m`

nên:

`delta + z_ground = 1.805 m`

Em hiểu rằng dịch input trước inference khác với dịch box sau inference.

Khi input thay đổi, model có thể thay đổi:

- số lượng detection;
- class;
- vị trí;
- confidence;
- hình học prediction.

### Quyết định khi gặp lỗi batch

Nếu toàn bộ hoặc nhiều cuboid cùng lệch z một lượng giống nhau:

**Dừng sửa tay từng box và kiểm pipeline/transform.**

Bằng chứng là `case-batch-z`, nơi 13/13 box cùng bị dịch xuống 1.805 m.

Nếu chỉ một box sai:

**Kiểm riêng object bằng nhiều view.**

Bằng chứng là `case-one-box-z`, nơi chỉ 1/13 box bị dịch xuống 1.805 m.

### Điều chưa chắc

- Prediction không phải ground truth.
- Không thể kết luận cấu hình A/B/C nào chính xác nhất chỉ từ số box.
- `mean_z` không phải metric chất lượng.
- Side view không đủ để kiểm đầy đủ y/yaw.
- Khi làm Robotaxi vẫn phải dùng nhiều view và camera.

---

## 2. Võ Quốc Dinh

### Vai trò

- Kiểm cấu hình và JSON trong lượt A.
- Theo dõi output và kết quả inference ở lượt B.
- Kiểm kết quả hình học/class và đối chiếu B/C ở lượt C.

### Quan sát A/B

A có 1 prediction trong khi B có 13 prediction mặc dù hai lượt dùng cùng Pillar XY = 0.16 m và chỉ thay `delta` từ 0 lên 1.73 m.

Sự thay đổi này cho thấy delta được áp dụng trước inference và có thể làm thay đổi kết quả model, chứ không chỉ thay đổi tọa độ z của output sau khi model đã chạy.

### Quan sát B/C

B có:

- 13 prediction;
- 10 `vehicles`;
- 2 `pedestrian`;
- 1 `two-wheels`.

C có:

- 6 prediction;
- tất cả là `pedestrian`.

Hai lượt cùng dùng `delta = 1.73 m`, nhưng Pillar XY thay từ 0.16 lên 0.32 m.

Từ kết quả này em thấy thay đổi kích thước pillar có thể làm thay đổi đáng kể cách model nhận diện đối tượng.

Tuy nhiên em chưa có đủ reference để kết luận cấu hình nào chính xác hơn.

### Diễn giải phép z thuận/ngược

Trước model:

`z_model = z_source - z_ground - delta`

Sau model, khi trả về hệ nguồn:

`z_source = z_model + z_ground + delta`

Phép cộng ngược là cần thiết để prediction quay về đúng hệ tọa độ nguồn.

Nếu quên phép inverse conversion thì nhiều box có thể cùng bị lệch z một lượng giống nhau.

### Quyết định khi gặp lỗi batch

Nếu nhiều box cùng lệch z nhất quán:

**Dừng batch và kiểm transform/pipeline.**

Không sửa tay lần lượt từng box vì như vậy chỉ che lỗi hệ thống.

Nếu chỉ một box lệch:

**Kiểm riêng box đó bằng nhiều góc nhìn.**

### Điều chưa chắc

- Số lượng prediction không cho biết trực tiếp độ chính xác.
- Không đủ cơ sở để xác định prediction nào là ground truth.
- Side view có giới hạn do mất thông tin theo trục y.
- Cần thêm Top/Front/free rotation và camera khi đánh giá cuboid thật.

---

## 3. Hà Quang Huy

### Vai trò

- Xem hình học và ghi log ở lượt A.
- Kiểm cấu hình/JSON ở lượt B.
- Theo dõi output và tổng hợp log ở lượt C.

### Quan sát A/B

Khi so A và B:

- Pillar XY giữ nguyên 0.16 m.
- delta thay từ 0 lên 1.73 m.
- số prediction tăng từ 1 lên 13.
- `mean_z` thay từ 0.330 lên 1.034.

Em nhận thấy thay đổi delta trước inference làm model tạo ra output khác rõ rệt.

Do đó không thể hiểu B đơn giản là bản sao của A được nâng hoặc hạ theo một hằng số z.

### Quan sát B/C

B và C cùng giữ `delta = 1.73 m`.

Khi Pillar XY tăng từ 0.16 lên 0.32 m:

- số prediction giảm từ 13 xuống 6;
- class prediction cũng thay đổi mạnh.

Điều này cho thấy representation của point cloud có ảnh hưởng trực tiếp đến kết quả inference.

Không thể chọn cấu hình tốt hơn chỉ dựa vào số detection.

### Diễn giải phép z thuận/ngược

Pipeline đầu tiên đưa dữ liệu từ hệ nguồn sang hệ model:

`z_model = z_source - z_ground - delta`

Sau inference phải đổi ngược:

`z_source = z_model + z_ground + delta`

Trong run B/C:

`z_ground + delta = 0.075 + 1.73 = 1.805 m`

Nếu toàn bộ batch bị thiếu phép cộng ngược này thì nhiều cuboid sẽ cùng bị lệch z khoảng 1.805 m.

### Quyết định khi gặp lỗi batch

Với `case-batch-z`:

- 13/13 box cùng bị dịch xuống 1.805 m.

Em sẽ:

- dừng chỉnh tay;
- kiểm transform;
- kiểm delta;
- kiểm z_ground;
- báo LC/operator nếu cần.

Với `case-one-box-z`:

- chỉ 1/13 box bị lệch.

Em sẽ kiểm riêng object thay vì kết luận pipeline toàn batch sai.

### Điều chưa chắc

- `case-correct` không phải ground truth.
- Prediction của model không phải reference annotation.
- Chưa đủ bằng chứng để kết luận cấu hình B hoặc C tốt hơn.
- Khi gặp object bị che hoặc point cloud thưa cần kiểm thêm nhiều view thay vì đoán.

---

# Kết luận nhóm

Nhóm đã chạy thành công PointPillars pretrained KITTI trên Student demo bằng một máy Docker CPU.

Runner hoàn thành:

- A;
- B;
- C;
- ba ca QC;
- `smoke.json` có `status: passed`.

Nhóm quan sát được:

1. Thay đổi `delta` trước inference có thể làm thay đổi đáng kể số lượng và class prediction.
2. Thay đổi Pillar XY cũng có thể làm thay đổi mạnh output PointPillars.
3. Không thể đánh giá cấu hình tốt hơn chỉ dựa vào số box hoặc `mean_z`.
4. Nếu toàn batch cùng lệch z một lượng giống nhau thì phải kiểm pipeline/transform trước khi sửa cuboid.
5. Nếu chỉ một box lệch thì kiểm riêng object bằng nhiều view.
6. Prediction demo không phải ground truth/reference.
7. Các JSON KITTI và `case-*.json` không được import vào Robotaxi.

Sau khi LC kiểm báo cáo phần nhóm, nhóm chuyển sang phần cá nhân trên portal/CVAT.

---

# LC ghi nhận riêng

- **Quyền dùng PCD/image và đúng ca:** [LC ĐIỀN]
- **Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:** [LC ĐIỀN]
- **Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:** [LC ĐIỀN]
- **Nhận xét từng thành viên và quyết định dừng pipeline:** [LC ĐIỀN]
- **Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:** [LC ĐIỀ

# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu.
Đây là kiểm tra formative; không ghi điểm của người khác.

---

## Nhóm và provenance

- **Mã nhóm/phòng:** [CẦN ĐIỀN]
- **Thành viên:** xem `TEAMMATES.md`
  - Bùi Quang Thái
  - Võ Quốc Dinh
  - Hà Quang Huy
- **Trạng thái:** `executed-by-group`
- **Người trực tiếp thao tác máy chạy runner:** Bùi Quang Thái
- **Ngày chạy:** 01/10/2026
- **Thời gian chạy:** khoảng 16:01–16:02, UTC+7
- **Hệ điều hành máy host:** Windows
- **Runtime Docker:** Docker Desktop / Linux container
- **Architecture:** `amd64 / x86_64`
- **Python host:** Python 3.13.14
- **Docker Server:** 29.8.0
- **Giới hạn container:** 4 CPU / 4 GB RAM

### Docker image

- **Source tag:** `day13-pointpillars:lc-20261001-amd64`
- **Image ID:**

`sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`

### Repo / code

- **Repo revision:**

`0831856d921609312d42c7582c366e5a311bb7b1`

- **working_tree_dirty:** `true`
- **preannotate SHA256:**

`65edf6ac95926f799c27f2c30c8595b0da4e8e2429f0c1a9aedf5c9e1fceb5ca`

- **helper SHA256:**

`c177fc008f79223e94b8c06423d3a806e422787d48c8d39f1c5fd3ce2c4eeaa7`

### PCD / dữ liệu đầu vào

- **Dataset:** KITTI Student demo
- **Frame ID:** `demo`
- **Số điểm:** 17,238
- **Source frame:** `pcd-source`
- **Input SHA256:**

`3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`

- **Nơi chạy:** máy nhóm local bằng Docker Desktop
- **Robotaxi:** không sử dụng làm đầu vào cho phần A/B/C

### Checkpoint

- **Model:** PointPillars pretrained KITTI
- **Checkpoint path:**

`/opt/PointPillars/pretrained/epoch_160.pth`

- **Checkpoint SHA256:**

`482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`

### Cấu hình chung

- **ROI:** `front-window`
- **Score threshold:** `0.3`
- **z_ground ước lượng:** `0.075 m`
- **Kênh thứ tư / intensity:** reflectance thật của KITTI demo đã bị bỏ trong bản PCD chuyển đổi. RGB = 0 chỉ là placeholder; pipeline sử dụng adapter kênh hằng và không coi đây là intensity LiDAR thật.
- **Hệ tọa độ:** x/y giữ theo nguồn KITTI; z của PCD demo đã được dịch +1.73 m trong quá trình chuẩn bị dữ liệu. `z_ground` vẫn được ước lượng từ PCD khi chạy.

---

# Ba lượt inference thật

A, B và C là ba lượt inference trên cùng một PCD bằng cùng checkpoint PointPillars pretrained.

- A/B chỉ thay đổi `delta`.
- B/C chỉ thay đổi kích thước Pillar XY.
- Không dùng A/C để quy nguyên nhân vì hai biến cùng thay đổi.
- Số hộp hoặc `mean_z` không phải tiêu chí trực tiếp để kết luận cấu hình nào tốt hơn.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV                                                                           | Quan sát có bằng chứng                                             |
| ------ | ----: | --------: | -------: | -----: | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| A      |     0 |      0.16 |        1 |  0.330 | `run-A/boxes-*.json`, `run-A/side-*.png`, `run-A/summary.csv`                          | Chỉ có 1 prediction thuộc class`vehicles`.                        |
| B      |  1.73 |      0.16 |       13 |  1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-*.png`, `run-B/summary.csv` | Có 13 prediction: 10`vehicles`, 2 `pedestrian`, 1 `two-wheels`. |
| C      |  1.73 |      0.32 |        6 |  1.091 | `run-C/boxes-*.json`, `run-C/side-*.png`, `run-C/summary.csv`                          | Có 6 prediction và cả 6 thuộc class`pedestrian`.                 |

---

## A/B — chỉ đổi delta

### Cấu hình

Lượt A:

- `delta = 0 m`
- Pillar XY = `0.16 m`

Lượt B:

- `delta = 1.73 m`
- Pillar XY = `0.16 m`

### Kết quả

- A: 1 hộp
- B: 13 hộp
- `mean_z` A: 0.330
- `mean_z` B: 1.034

Phân bố class:

- A: 1 `vehicles`
- B:
  - 10 `vehicles`
  - 2 `pedestrian`
  - 1 `two-wheels`

### Quan sát

Khi thay `delta` từ 0 m lên 1.73 m và giữ Pillar XY = 0.16 m, số prediction thay đổi từ 1 lên 13.

Thành phần class cũng thay đổi rõ rệt. A chỉ có một prediction thuộc `vehicles`, trong khi B xuất hiện cả `vehicles`, `pedestrian` và `two-wheels`.

Điều này cho thấy thay đổi z trước inference làm thay đổi input mà model nhận được và có thể làm thay đổi cả số lượng, class và vị trí prediction.

Không thể hiểu B đơn giản là lấy các cuboid của A rồi dịch toàn bộ chúng thêm 1.73 m sau inference.

### Kết luận

Chưa đủ bằng chứng để kết luận B chính xác hơn A chỉ vì B có nhiều prediction hơn.

Muốn đánh giá từng cuboid vẫn cần đối chiếu với PCD, nhiều góc nhìn và reference phù hợp.

### Điều chưa chắc

Ảnh Side là hình chiếu x-z nên các đối tượng có tọa độ y khác nhau có thể chồng lên nhau.

Vì vậy ảnh Side một mình không đủ để:

- xác nhận tâm y;
- xác nhận yaw;
- đánh giá đầy đủ footprint;
- kết luận chất lượng từng cuboid.

---

## B/C — chỉ đổi Pillar XY

### Cấu hình

Lượt B:

- `delta = 1.73 m`
- Pillar XY = `0.16 m`

Lượt C:

- `delta = 1.73 m`
- Pillar XY = `0.32 m`

### Kết quả

- B: 13 hộp
- C: 6 hộp
- `mean_z` B: 1.034
- `mean_z` C: 1.091

Phân bố class:

B:

- 10 `vehicles`
- 2 `pedestrian`
- 1 `two-wheels`

C:

- 6 `pedestrian`

### Quan sát

Khi giữ nguyên `delta = 1.73 m` nhưng tăng Pillar XY từ 0.16 m lên 0.32 m, số prediction giảm từ 13 xuống 6.

Thành phần class cũng thay đổi mạnh. Các prediction `vehicles` và `two-wheels` có ở B không còn xuất hiện trong kết quả C, trong khi C gồm 6 prediction `pedestrian`.

Điều này cho thấy kích thước pillar ảnh hưởng đến cách point cloud được chia/gom trên mặt phẳng x-y trước khi đưa vào mạng, từ đó làm thay đổi kết quả inference.

### Kết luận

Không đủ bằng chứng để kết luận B tốt hơn C hoặc C tốt hơn B chỉ dựa vào:

- số lượng box;
- class;
- `mean_z`;
- confidence.

Lượt C vẫn sử dụng cùng checkpoint pretrained. Đây là thí nghiệm thay đổi biểu diễn đầu vào chứ không phải model được train lại riêng cho Pillar XY = 0.32 m.

---

## Giới hạn ROI và ảnh Side

Các lượt A/B/C sử dụng cùng:

- checkpoint KITTI;
- score threshold = 0.3;
- front-window ROI.

Đối tượng nằm ngoài ROI không được dùng làm bằng chứng rằng model bỏ sót trong phép so A/B/C.

Ảnh Side là hình chiếu x-z toàn scene. Nhiều đối tượng khác nhau theo trục y có thể chồng lên nhau trên cùng ảnh.

Do đó:

- Side phù hợp để quan sát cao độ và lỗi z rõ ràng.
- Side không đủ để đánh giá chính xác center theo y.
- Side không đủ để xác nhận yaw.
- Side không đủ để đánh giá toàn bộ footprint.
- Khi chỉnh cuboid Robotaxi thật vẫn cần kiểm Top / Side / Front / free rotation và camera.

---

## JSON nào chưa đủ cơ sở để import?

Các JSON trong `run-A`, `run-B`, `run-C` là prediction trên KITTI demo phục vụ thí nghiệm pipeline.

Không JSON nào trong phần Student demo được import vào job Robotaxi.

Các prediction A/B/C:

- không phải ground truth;
- không phải reference;
- không chứng minh annotation đúng;
- không cùng frame với Robotaxi.

Prediction Robotaxi phải được lấy bằng nút **Nạp pre-label cho job này** trên portal để hệ thống lấy đúng prediction Robotaxi do LC chạy trước và kiểm đúng frame/schema.

---

# Ca QC có kiểm soát — không import CVAT

Ba ca QC được tạo từ prediction thật của lượt B.

### Source

- **Source prediction:** `boxes-demo-delta-1.73-voxel-0.16.json`
- **Source prediction SHA256:**

`c2a8db247353f0ef00299ff50acff997b7b4a87bdcfcefe792815acb5652cc80`

- **Frame ID:** `demo`
- **Source frame:** `pcd-source`
- **Dataset:** KITTI
- **delta:** `1.73 m`
- **z_ground:** `0.075 m`

Tổng offset:

`height_offset = delta + z_ground`

`height_offset = 1.73 + 0.075`

`height_offset = 1.805 m`

- **Tổng số box:** 13
- **training_only:** `true`

Các ca QC dưới đây là biến đổi có kiểm soát từ prediction B.

Chúng:

- không phải ba lần inference mới;
- không phải ground truth;
- không phải reference;
- không được import vào CVAT.

| Ca                 | Số hộp lệch z / tổng hộp | Lượng lệch       | Class/x/y/yaw có đổi?                      | Quyết định                                                                 | Bằng chứng                                                                                   |
| ------------------ | ----------------------------- | ------------------- | --------------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `case-correct`   | 0 / 13                        | 0 m                 | Không                                        | Dùng làm mốc đối chiếu; không coi là ground truth                     | `case-correct.json`, `side-correct.png`; prediction nguồn được giữ nguyên            |
| `case-batch-z`   | 13 / 13                       | `-1.805 m` theo z | Không; chỉ z của toàn bộ box bị dịch   | **Dừng batch**, kiểm transform/pipeline; không sửa tay từng cuboid | `case-batch-z.json`, `side-batch-z.png`; mọi box cùng bị dịch xuống 1.805 m           |
| `case-one-box-z` | 1 / 13                        | `-1.805 m` theo z | Không; chỉ z của box đầu tiên bị dịch | **Kiểm riêng object**, không kết luận toàn pipeline sai           | `case-one-box-z.json`, `side-one-box-z.png`; chỉ box đầu tiên bị dịch xuống 1.805 m |

---

## case-correct

`case-correct` giữ nguyên prediction nguồn của run-B.

Ca này được dùng làm mốc để đối chiếu với hai trường hợp lỗi z.

Tên `correct` ở đây chỉ có nghĩa là phép chuyển đổi nguồn không bị cố ý làm sai.

Không được hiểu:

`case-correct = ground truth`

hoặc:

`case-correct = cuboid đúng hoàn toàn`.

Nó vẫn chỉ là prediction của PointPillars.

---

## case-batch-z

Trong `case-batch-z`:

- tổng số box: 13;
- số box bị dịch z: 13;
- độ dịch: `-1.805 m`;
- class giữ nguyên;
- x giữ nguyên;
- y giữ nguyên;
- yaw giữ nguyên.

### Nhận định

Việc toàn bộ 13 box cùng lệch z một lượng giống nhau là dấu hiệu phù hợp với lỗi transform/pipeline toàn batch hơn là lỗi annotation riêng từng object.

### Quyết định

**Dừng sửa tay từng cuboid.**

Kiểm lại:

1. hệ tọa độ nguồn/model;
2. `z_ground`;
3. `delta`;
4. phép forward conversion;
5. phép inverse conversion;
6. input/frame đang dùng.

Nếu xác nhận lỗi pipeline, yêu cầu tạo lại prediction bằng pipeline đúng thay vì sửa thủ công 13 box.

---

## case-one-box-z

Trong `case-one-box-z`:

- tổng số box: 13;
- số box bị dịch z: 1;
- box bị tác động: box đầu tiên;
- độ dịch: `-1.805 m`;
- 12 box còn lại giữ nguyên.

### Nhận định

Một box bị lệch z trong khi các box khác không thay đổi chưa đủ cơ sở để kết luận toàn pipeline sai.

### Quyết định

Kiểm riêng object bằng nhiều view:

- Top;
- Side;
- Front;
- free rotation;
- cụm điểm LiDAR;
- đáy box;
- mặt đường cục bộ.

Nếu các box còn lại vẫn hợp lý thì ưu tiên xem đây là lỗi object-local hoặc trường hợp cần kiểm thêm.

---

# Trạng thái chạy

`smoke.json`:

- `status: passed`

Các bước:

- `docker-load`: passed
- `run-A`: passed
- `run-B`: passed
- `run-C`: passed
- `qc-cases`: passed

### Thời gian chạy

- Docker load: khoảng 48.32 giây
- Run A: khoảng 4.95 giây
- Run B: khoảng 4.48 giây
- Run C: khoảng 2.88 giây
- QC cases: khoảng 1.52 giây

Thời gian trên chỉ mô tả lần chạy thực tế trên máy nhóm, không dùng làm metric chất lượng model.

---

# Nhận xét cá nhân

## 1. Bùi Quang Thái

### Vai trò

- Trực tiếp chuẩn bị và vận hành Student bundle trên máy nhóm.
- Kiểm Python và Docker Desktop.
- Kiểm đúng architecture `amd64`.
- Chạy runner A/B/C.
- Kiểm `smoke.json`.
- Đọc log và `summary.csv`.
- Kiểm `qc-cases/manifest.json`.
- Tổng hợp báo cáo nhóm.

### Quan sát A/B

A sử dụng:

- `delta = 0`
- Pillar XY = `0.16`

và cho 1 prediction thuộc `vehicles`.

B sử dụng:

- `delta = 1.73`
- Pillar XY = `0.16`

và cho 13 prediction:

- 10 `vehicles`
- 2 `pedestrian`
- 1 `two-wheels`

Em nhận thấy khi chỉ thay delta, kết quả inference thay đổi rõ rệt về số lượng và class prediction.

Điều này cho thấy thay đổi z trước inference có thể ảnh hưởng trực tiếp đến output của model, không thể coi đây đơn thuần là dịch cùng các box cũ theo z.

### Quan sát B/C

B sử dụng Pillar XY = 0.16 m và tạo 13 prediction.

C sử dụng Pillar XY = 0.32 m và tạo 6 prediction.

Cả số lượng lẫn thành phần class đều thay đổi.

Điều này cho thấy kích thước pillar ảnh hưởng đến biểu diễn point cloud đầu vào và từ đó ảnh hưởng kết quả PointPillars.

Em không kết luận B tốt hơn C chỉ vì B có nhiều box hơn.

### Diễn giải phép z thuận/ngược

Pipeline sử dụng:

`z_model = z_source - z_ground - delta`

Khi trả prediction về hệ nguồn:

`z_source = z_model + z_ground + delta`

Trong B/C:

- `delta = 1.73 m`
- `z_ground ≈ 0.075 m`

nên:

`delta + z_ground = 1.805 m`

Em hiểu rằng dịch input trước inference khác với dịch box sau inference.

Khi input thay đổi, model có thể thay đổi:

- số lượng detection;
- class;
- vị trí;
- confidence;
- hình học prediction.

### Quyết định khi gặp lỗi batch

Nếu toàn bộ hoặc nhiều cuboid cùng lệch z một lượng giống nhau:

**Dừng sửa tay từng box và kiểm pipeline/transform.**

Bằng chứng là `case-batch-z`, nơi 13/13 box cùng bị dịch xuống 1.805 m.

Nếu chỉ một box sai:

**Kiểm riêng object bằng nhiều view.**

Bằng chứng là `case-one-box-z`, nơi chỉ 1/13 box bị dịch xuống 1.805 m.

### Điều chưa chắc

- Prediction không phải ground truth.
- Không thể kết luận cấu hình A/B/C nào chính xác nhất chỉ từ số box.
- `mean_z` không phải metric chất lượng.
- Side view không đủ để kiểm đầy đủ y/yaw.
- Khi làm Robotaxi vẫn phải dùng nhiều view và camera.

---

## 2. Võ Quốc Dinh

### Vai trò

- Kiểm cấu hình và JSON trong lượt A.
- Theo dõi output và kết quả inference ở lượt B.
- Kiểm kết quả hình học/class và đối chiếu B/C ở lượt C.

### Quan sát A/B

A có 1 prediction trong khi B có 13 prediction mặc dù hai lượt dùng cùng Pillar XY = 0.16 m và chỉ thay `delta` từ 0 lên 1.73 m.

Sự thay đổi này cho thấy delta được áp dụng trước inference và có thể làm thay đổi kết quả model, chứ không chỉ thay đổi tọa độ z của output sau khi model đã chạy.

### Quan sát B/C

B có:

- 13 prediction;
- 10 `vehicles`;
- 2 `pedestrian`;
- 1 `two-wheels`.

C có:

- 6 prediction;
- tất cả là `pedestrian`.

Hai lượt cùng dùng `delta = 1.73 m`, nhưng Pillar XY thay từ 0.16 lên 0.32 m.

Từ kết quả này em thấy thay đổi kích thước pillar có thể làm thay đổi đáng kể cách model nhận diện đối tượng.

Tuy nhiên em chưa có đủ reference để kết luận cấu hình nào chính xác hơn.

### Diễn giải phép z thuận/ngược

Trước model:

`z_model = z_source - z_ground - delta`

Sau model, khi trả về hệ nguồn:

`z_source = z_model + z_ground + delta`

Phép cộng ngược là cần thiết để prediction quay về đúng hệ tọa độ nguồn.

Nếu quên phép inverse conversion thì nhiều box có thể cùng bị lệch z một lượng giống nhau.

### Quyết định khi gặp lỗi batch

Nếu nhiều box cùng lệch z nhất quán:

**Dừng batch và kiểm transform/pipeline.**

Không sửa tay lần lượt từng box vì như vậy chỉ che lỗi hệ thống.

Nếu chỉ một box lệch:

**Kiểm riêng box đó bằng nhiều góc nhìn.**

### Điều chưa chắc

- Số lượng prediction không cho biết trực tiếp độ chính xác.
- Không đủ cơ sở để xác định prediction nào là ground truth.
- Side view có giới hạn do mất thông tin theo trục y.
- Cần thêm Top/Front/free rotation và camera khi đánh giá cuboid thật.

---

## 3. Hà Quang Huy

### Vai trò

- Xem hình học và ghi log ở lượt A.
- Kiểm cấu hình/JSON ở lượt B.
- Theo dõi output và tổng hợp log ở lượt C.

### Quan sát A/B

Khi so A và B:

- Pillar XY giữ nguyên 0.16 m.
- delta thay từ 0 lên 1.73 m.
- số prediction tăng từ 1 lên 13.
- `mean_z` thay từ 0.330 lên 1.034.

Em nhận thấy thay đổi delta trước inference làm model tạo ra output khác rõ rệt.

Do đó không thể hiểu B đơn giản là bản sao của A được nâng hoặc hạ theo một hằng số z.

### Quan sát B/C

B và C cùng giữ `delta = 1.73 m`.

Khi Pillar XY tăng từ 0.16 lên 0.32 m:

- số prediction giảm từ 13 xuống 6;
- class prediction cũng thay đổi mạnh.

Điều này cho thấy representation của point cloud có ảnh hưởng trực tiếp đến kết quả inference.

Không thể chọn cấu hình tốt hơn chỉ dựa vào số detection.

### Diễn giải phép z thuận/ngược

Pipeline đầu tiên đưa dữ liệu từ hệ nguồn sang hệ model:

`z_model = z_source - z_ground - delta`

Sau inference phải đổi ngược:

`z_source = z_model + z_ground + delta`

Trong run B/C:

`z_ground + delta = 0.075 + 1.73 = 1.805 m`

Nếu toàn bộ batch bị thiếu phép cộng ngược này thì nhiều cuboid sẽ cùng bị lệch z khoảng 1.805 m.

### Quyết định khi gặp lỗi batch

Với `case-batch-z`:

- 13/13 box cùng bị dịch xuống 1.805 m.

Em sẽ:

- dừng chỉnh tay;
- kiểm transform;
- kiểm delta;
- kiểm z_ground;
- báo LC/operator nếu cần.

Với `case-one-box-z`:

- chỉ 1/13 box bị lệch.

Em sẽ kiểm riêng object thay vì kết luận pipeline toàn batch sai.

### Điều chưa chắc

- `case-correct` không phải ground truth.
- Prediction của model không phải reference annotation.
- Chưa đủ bằng chứng để kết luận cấu hình B hoặc C tốt hơn.
- Khi gặp object bị che hoặc point cloud thưa cần kiểm thêm nhiều view thay vì đoán.

---

# Kết luận nhóm

Nhóm đã chạy thành công PointPillars pretrained KITTI trên Student demo bằng một máy Docker CPU.

Runner hoàn thành:

- A;
- B;
- C;
- ba ca QC;
- `smoke.json` có `status: passed`.

Nhóm quan sát được:

1. Thay đổi `delta` trước inference có thể làm thay đổi đáng kể số lượng và class prediction.
2. Thay đổi Pillar XY cũng có thể làm thay đổi mạnh output PointPillars.
3. Không thể đánh giá cấu hình tốt hơn chỉ dựa vào số box hoặc `mean_z`.
4. Nếu toàn batch cùng lệch z một lượng giống nhau thì phải kiểm pipeline/transform trước khi sửa cuboid.
5. Nếu chỉ một box lệch thì kiểm riêng object bằng nhiều view.
6. Prediction demo không phải ground truth/reference.
7. Các JSON KITTI và `case-*.json` không được import vào Robotaxi.

Sau khi LC kiểm báo cáo phần nhóm, nhóm chuyển sang phần cá nhân trên portal/CVAT.

---

# LC ghi nhận riêng

- **Quyền dùng PCD/image và đúng ca:** [LC ĐIỀN]
- **Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:** [LC ĐIỀN]
- **Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:** [LC ĐIỀN]
- **Nhận xét từng thành viên và quyết định dừng pipeline:** [LC ĐIỀN]
- **Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:** [LC ĐIỀN]
