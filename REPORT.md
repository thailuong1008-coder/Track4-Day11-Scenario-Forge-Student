MSSV: 2A202602932

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): A

---

## 1. Phát lại log có trung thực không? (bước A)

**Bảng số liệu từ `replay.json`:**

| Trường dữ liệu | Số lượng | Chi tiết loại tiếp xúc (`contact`) |
|---|---|---|
| Số kịch bản bị bỏ do người lái gây va chạm (`dropped_human_crash_logs`) | 1 | — |
| Số kịch bản đưa vào kiểm thử (`n_used` / `n`) | 399 / 400 | — |
| Va chạm khi planner chạy trên log phát lại (`replay.crashes`) | 110 | `rear`: 103, `side`: 7, `front`: 0 |
| Va chạm trong thế giới phản ứng (`reactive.crashes`) | 0 | Không có va chạm nào (`contact`: `{}`) |
| Va chạm xảy ra ở cả hai môi trường (`crash_in_both`) | 0 | — |

* **Vì sao phần lớn va chạm của "phát lại" là `rear` và là va chạm giả?**  
  Trong chế độ phát lại log (`replay`), các xe xung quanh di chuyển theo đúng quỹ đạo cố định đã ghi từ trước mà không hề phản ứng lại trạng thái của xe ta (hở vòng - open-loop). Khi planner của xe ta phát hiện chướng ngại vật phía trước và phanh sớm hơn người lái người thật, xe bám đuôi (`TG`) trong log không phanh theo (vì log ghi lại lúc xe người lái chưa hề phanh gấp tại thời điểm đó) và lao thẳng vào đuôi xe ta (`rear`: 103/110 vụ). Đây là **va chạm giả** (sim artifact) vì trong thực tế (`reactive`), người lái xe phía sau sẽ quan sát thấy khoảng cách thu hẹp và đạp phanh giảm tốc theo mô hình IDM (bằng chứng là trong `reactive`, số va chạm là 0).

* **Phân tích một kịch bản trong `replay_only` (Kịch bản 7 - `scenario: 7`):**  
  Ở kịch bản 7, trong `replay.json`, xe ta bị đâm đuôi (`contact: "rear"`). Tuy nhiên, trên cùng kịch bản này trong thế giới phản ứng, khoảng cách an toàn thời gian nhỏ nhất đạt tới `min_ttc_reactive = 7.46 s`, xe `TG` phản ứng phanh giữ cự ly hoàn hảo và không hề có va chạm. Sự cố trong log phát lại hoàn toàn do xe `TG` chạy mù quáng theo chuỗi tọa độ cũ.

---

## 2. Kịch bản bất khả thi (bước B)

**Các luật trong `is_feasible` và căn cứ vật lý:**
1. **Gia tốc ngang đỉnh chuyển làn $a_{lat} = (\pi / cutDur)^2 \cdot 3.5 / 2 \le \mu g$:**  
   *Căn cứ:* Giới hạn bám ma sát của vòng tròn ma sát lốp xe (friction circle). Khi đổi làn 3.5 m với quỹ đạo cosine trong thời gian $cutDur$ quá ngắn trên đường trơn ma sát thấp $\mu$, lực ma sát ngang không đủ cung cấp gia tốc hướng tâm, xe sẽ bị trượt văng mất lái.
2. **Gia tốc phanh sau nhập làn $cutBrake \le \mu g$:**  
   *Căn cứ:* Định luật ma sát Coulomb $F_{brake} \le \mu m g \implies a_{brake} \le \mu g$. Một chiếc xe không thể phanh giảm tốc với gia tốc lớn hơn giới hạn gia tốc ma sát mặt đường.
3. **Vận tốc người đi bộ $pedV \le 2.5\text{ m/s}$ (~9 km/h):**  
   *Căn cứ:* Giới hạn sinh học vận động học của người đi bộ trong đô thị. Tốc độ đi bộ thông thường là 1.2–1.5 m/s, chạy nước rút ngắn qua đường tối đa không quá 2.5 m/s.
4. **Vận tốc xe nhập làn $0 < cutV \le 22.0\text{ m/s}$ (~79.2 km/h):**  
   *Căn cứ:* Phương tiện di chuyển trên đường phố đô thị phải có vận tốc dương (tiến về phía trước) và không vượt quá tốc độ tối đa cho phép/khả thi của thiết kế hình học đường phố.

**Phân tích từ `hunt.json`:**
* Cuộc săn sinh nhiều kịch bản bất khả thi nhất là **`crosswalk_block`** với **12 kịch bản bị loại** (`hunt.json: crosswalk_block.infeasible = 12`), trong khi `rear_ended` là 5, `cut_in` là 3, và `bus_bypass` là 2.
* *Vì sao CEM bị kéo về phía đó:* Điểm nguy hiểm của `crosswalk_block` dựa trên khoảng phanh còn lại trước người đi bộ (`ped_room`). Để tối đa hóa mức độ nguy hiểm, thuật toán CEM có xu hướng đẩy vận tốc người đi bộ $pedV$ lên mức phi thực tế ($> 2.5$ m/s) và đẩy ma sát xuống thấp kết hợp phanh gắt để xe ta không thể dừng trước vạch.
* *Hậu quả nếu đếm chúng:* Nếu đếm các kịch bản bất khả thi là va chạm, xác suất rủi ro sẽ bị **thổi phồng sai lệch** (overestimation), tạo ra các va chạm "ma" không thể xảy ra trong thế giới thực, khiến kỹ sư tinh chỉnh planner trở nên thận trọng thái quá một cách vô ích.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

* **Số liệu từ `mc.json`:**
  * Số lần thử ($n$): `3000` (`mc.json: n = 3000`)
  * Số va chạm ghi nhận ($k$): `9` (`mc.json: crashes = 9`)
  * Ước lượng điểm ($\hat{p}$): `0.003` ($3.00 \times 10^{-3}$) (`mc.json: p = 0.003`)
  * Khoảng tin cậy Wilson 95%: `[0.001579, 0.005692]` ($\approx [1.58 \times 10^{-3}, 5.69 \times 10^{-3}]$) (`mc.json: lo`, `hi`)
  * Phân loại theo kiểu: Tất cả 9 vụ đều là `bus_bypass` (`mc.json: modes = {"bus_bypass": 9}`).

* **Số lần chạy cần thiết để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:**  
  Biên sai số yêu cầu: $E = 0.2 \cdot p$.  
  Công thức khoảng tin cậy tỷ lệ: $1.96 \sqrt{\frac{p(1-p)}{n}} \le 0.2 p \implies n \ge \frac{1.96^2 (1-p)}{0.04 p} \approx \frac{96}{p}$.  
  Với $\hat{p} = 0.003$:
  $$n \approx \frac{96}{0.003} = 32{,}000 \text{ lần chạy}$$
  (Tính chính xác theo công thức: $n \approx \frac{3.8416 \times 0.997}{0.00012} \approx 31{,}917$ lần chạy).

* **Các kiểu thất bại chưa hề xuất hiện trong 3.000 lần chạy:**  
  Các kiểu `cut_in`, `crosswalk_block`, `rear_ended` và `other` đều có $k = 0$ vụ va chạm (`mc.json: per_mode.<mode>.k = 0`).  
  **Con số đúng cần báo:** Không được báo xác suất bằng 0. Con số đúng phải báo là **cận trên của khoảng tin cậy Wilson 95%** là **$1.28 \times 10^{-3}$** (`mc.json: per_mode.cut_in.hi = 0.00127889566`), hoặc theo quy tắc số ba (Rule of Three): $\text{Upper Bound} \approx 3 / n = 3 / 3000 = 1.0 \times 10^{-3}$.

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

**Bảng kết quả từ `estimate.json`:**

| Kiểu thất bại | Seed | Xác suất ước lượng $p$ | Số va chạm (hits) | Cỡ mẫu hiệu dụng (ESS) | Tỷ trọng mẫu lớn nhất (`top_share`) |
|---|:---:|:---:|:---:|:---:|:---:|
| `bus_bypass` | 0 | $1.12 \times 10^{-3}$ | 30 | 125.2 | 0.409 |
| `bus_bypass` | 1 | $7.43 \times 10^{-4}$ | 28 | 128.9 | 0.681 |
| `bus_bypass` | 2 | $4.95 \times 10^{-4}$ | 28 | 122.9 | 0.269 |
| `cut_in` | 0 | 0.0 (cận trên $2.50 \times 10^{-2}$) | 0 | 114.8 | 0.000 |
| `cut_in` | 1 | 0.0 (cận trên $2.50 \times 10^{-2}$) | 0 | 118.8 | 0.000 |
| `cut_in` | 2 | 0.0 (cận trên $2.50 \times 10^{-2}$) | 0 | 114.8 | 0.000 |
| `crosswalk_block` | 0 | $1.98 \times 10^{-6}$ | 58 | 121.7 | 0.478 |
| `crosswalk_block` | 1 | $5.21 \times 10^{-6}$ | 59 | 116.4 | 0.871 |
| `crosswalk_block` | 2 | $6.88 \times 10^{-7}$ | 54 | 132.0 | 0.460 |
| `rear_ended` | 0 | 0.0 (cận trên $2.50 \times 10^{-2}$) | 0 | 118.1 | 0.000 |
| `rear_ended` | 1 | 0.0 (cận trên $2.50 \times 10^{-2}$) | 0 | 136.9 | 0.000 |
| `rear_ended` | 2 | 0.0 (cận trên $2.50 \times 10^{-2}$) | 0 | 128.1 | 0.000 |
| **Tổng (`total`)** | **0** | **$1.12 \times 10^{-3}$** $[0.00, 3.58 \times 10^{-3}]$ | — | — | — |
| **Tổng (`total`)** | **1** | **$7.48 \times 10^{-4}$** $[0.00, 3.03 \times 10^{-3}]$ | — | — | — |
| **Tổng (`total`)** | **2** | **$4.95 \times 10^{-4}$** $[1.47 \times 10^{-4}, 2.12 \times 10^{-3}]$ | — | — | — |

* **Ba seed có đồng ý với nhau và với Monte Carlo không?**  
  * Giữa 3 seed: Các giá trị tổng $p$ dao động từ $4.95 \times 10^{-4}$ đến $1.12 \times 10^{-3}$ (chênh lệch cỡ 2.3 lần), nằm cùng một bậc độ lớn ($10^{-3} - 10^{-4}$). Với `crosswalk_block`, giá trị dao động từ $6.88 \times 10^{-7}$ đến $5.21 \times 10^{-6}$ (chênh lệch ~7.6 lần).
  * So với Monte Carlo: MC ước lượng $p = 3.00 \times 10^{-3}$, cao hơn ước lượng IS khoảng 2.7–6 lần. Điều này hoàn toàn khớp với lý thuyết: phân phối đề xuất $q$ của CEM sau các vòng lặp bị co hẹp (tighten) quanh một phương thức va chạm dễ nhất, khiến ước lượng IS bị chệch dưới (underestimate) so với xác suất tự nhiên thực tế.
* **Độ suy biến trọng số (`top_share` > 0.5):**  
  Xảy ra ở Seed 1 của `bus_bypass` (`top_share = 0.681`) và đặc biệt ở Seed 1 của `crosswalk_block` (`top_share = 0.871`). Khi `top_share` lớn hơn 0.5, một mẫu duy nhất chiếm tới 68% hay 87% toàn bộ tổng trọng số xác suất! Khi đó, con số ước lượng thực chất bị chi phối bởi trọng số $w = p/q$ của đúng một mẫu đơn lẻ may mắn rơi vào vùng hiếm, làm mất đi tính đại diện trung bình của mẫu.
* **So sánh ngân sách (`budget`):**  
  Ngân sách IS thực hiện là `budget = 16,800` lần chạy (`estimate.json: budget = 16800`). Để Monte Carlo tìm thấy và đo đạc được sự kiện hiếm cỡ $10^{-6}$ như `crosswalk_block` với cùng sai số tương đối $\pm 20\%$, Monte Carlo cần $n \approx 96 / 10^{-6} \approx 96{,}000{,}000$ (96 triệu) lần chạy. IS giúp tiết kiệm chi phí tính toán gần 6.000 lần.
* **Phần `other` trong tổng:**  
  Lấy từ bước Monte Carlo C1 (`mc.json: modes.other`, với $k=0, n=3000, p=0, hi=0.001279$). Không săn `other` vì đây là phần dư không có cơ chế hành vi vật lý đồng nhất, không thể xây dựng hàm mục tiêu (danger score) dẫn đường cho CEM.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

* **Kiểu `bus_bypass`:**  
  *Ảnh chụp màn hình:* `evidence/shots/bus_bypass.png`  
  *Chuỗi sự kiện:* Xe buýt đỗ ở làn E2 tại trạm xe che khuất tầm nhìn. Xe cắt làn `C2` lách qua xe buýt với tốc độ chậm ($v < 6$ m/s, $x < 112$ m) để nhập vào làn E1 ngay trước đầu xe ta. Xe ta bị che khuất tầm quan sát bởi thân xe buýt cao, đến khi phát hiện `C2` thì cự ly đã quá gần khiến va chạm xảy ra.
* **Kiểu `crosswalk_block`:**  
  *Ảnh chụp màn hình:* `evidence/shots/crosswalk_block.png`  
  *Chuỗi sự kiện:* Người đi bộ `P1` bước qua đường tại vạch qua đường ($x = 122$ m) từ phía sau xe buýt đỗ. Xe ta phanh gấp để tránh đâm vào người nhưng phanh quá muộn, dừng hẳn lại khi thân xe đã nằm đè lên vạch qua đường, chặn lối đi của người đi bộ.
* **Kiểu nào cuộc săn KHÔNG tìm thấy?**  
  Cuộc săn không tìm thấy `cut_in` (ngoài khu vực xe buýt) và `rear_ended` (`hunt.json: cut_in.found = {'bus_bypass': 17}`, `rear_ended.found = {'crosswalk_block': 5, 'bus_bypass': 3}`). Ta có mức độ tin tưởng cao rằng trong điều kiện giao thông tự nhiên bình thường của thị trấn A, xe ta duy trì cự ly IDM an toàn nên xe phía sau rất khó tông đuôi (`rear_ended`), còn các tình huống cắt làn ngoài đoạn xe buýt đều có đủ tầm nhìn để xe ta xử lý êm xuôi.
* **Nếu chỉ chạy MỘT cuộc săn chung cho mọi kiểu:**  
  Hiện tượng co cụm nghiệm (mode collapse) sẽ xảy ra: Thuật toán CEM sẽ bị hút toàn bộ về kiểu va chạm dễ đạt điểm nguy hiểm cao nhất là `bus_bypass`. Bằng chứng rõ ràng trong `hunt.json` là khi săn `cut_in`, thuật toán đã bị trượt và tìm ra 17 vụ `bus_bypass`. Nếu không tách riêng từng kiểu săn, kiểu `crosswalk_block` sẽ hoàn toàn bị che lấp và không bao giờ được tìm thấy.

---

## 6. Sửa planner (bước E + 3D)

**Bảng tổng hợp kết quả sửa planner từ `fix.json`:**

| Đại lượng | Tập TRAIN | Tập TEST (Bản sửa: `T=2.0`) | Tập TEST (Bản gốc `baseline`) |
|---|:---:|:---:|:---:|
| Chính sách (`policy`) | `{'T': 2.0}` | `{'T': 2.0}` | Mặc định (`T=1.6, v0=12.5,...`) |
| Tỷ lệ thất bại ước lượng (`fail`) | $1.19 \times 10^{-3}$ | $7.12 \times 10^{-4}$ | $4.20 \times 10^{-4}$ |
| Cận trên thất bại quan sát (`fail_hi_seen`) | $2.37 \times 10^{-3}$ | $1.16 \times 10^{-3}$ | $6.60 \times 10^{-4}$ |
| Cận trên hợp lệ 95% (`fail_hi`) | $2.37 \times 10^{-3}$ | **$2.44 \times 10^{-3}$** | **$1.94 \times 10^{-3}$** |
| Kiểu chưa thấy (`unseen`) | `[]` | `[]` | `[]` |
| Quãng đường trung bình (`progress`) | 203.7 m | **205.0 m** (giảm 4.5% $\le 5\%$) | **214.6 m** |
| Số khung phanh gấp (`harsh`, $a < -3$) | 7.80 | **7.49 khung** | **8.20 khung** |

* **Vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật?**  
  `fail` chỉ là giá trị trung bình điểm (point estimate), trong khi `fail_hi` là cận trên bảo thủ 95% tích hợp cả biên độ sai số chuẩn $1.96 \cdot SE$ và cận của các va chạm ngoài tập săn ($3/n$). Khi một chính sách giảm thiểu va chạm mạnh, số lượng va chạm thực tế ít đi khiến phương sai trọng số IS tăng lên ở các mẫu hiếm, đẩy biên độ sai số $SE$ rộng hơn.
* **Chính sách được chọn và đánh đổi:**  
  Chính sách được chọn là tăng khoảng đệm thời gian bám đuôi **`T: 2.0`** (tăng từ 1.6 s lên 2.0 s). Bản sửa giữ được quãng đường 205.0 m (chỉ giảm 4.5% so với mức gốc 214.6 m, thỏa mãn điều kiện mất không quá 5%), đồng thời giảm tần suất phanh gấp từ 8.20 xuống 7.49 khung.
* *Ảnh chụp 3D có bóng mờ (ghost car):* `evidence/shots/ghost_fix.png` (minh họa xe bản sửa `T=2.0` phanh êm ái giữ khoảng cách xa hơn, tránh được cú tạt đầu trong khi bóng mờ của xe gốc bị đâm).
* **Núm chỉnh "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại:**  
  - Điển hình là núm **`cutinLat`** (ngưỡng khoảng cách ngang để coi xe khác nhập làn). Khi tăng `cutinLat` lên 3.0 m hoặc 3.3 m (nghe có vẻ "rộng lượng" nhường đường hơn), trên tập TRAIN tỷ lệ thất bại vọt lên $1.25 \times 10^{-2}$ (gấp hơn 10 lần so với `T: 2.0`!).  
  - *Giải thích theo phản ứng của thế giới:* Xe ta không nhận diện xe lấn làn từ sớm mà đợi nó vào sâu tim làn mới coi là xe dẫn đầu. Lúc này xe lấn làn đã quá gần, planner buộc phải phanh cực gấp, dẫn đến xe bám đuôi phía sau không kịp phản ứng hoặc xe ta đâm vào hông xe chuyển làn. Tương tự, hạ `cautionV` xuống 5.0 m/s khiến xe chạy quá chậm (quãng đường giảm mạnh chỉ còn 171.1 m, bị loại bởi `max_loss = 5%`) và phanh gấp tăng vọt lên 13.0 khung.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định ảnh hưởng nhiều nhất đến con số rủi ro trong `space.py`:

1. **Khoảng cách thời gian chấp nhận nhập làn của xe chuyển làn (`cutGap`):**  
   *Dữ liệu thật thay thế:* Bộ dữ liệu quỹ đạo xe thực tế quy mô lớn (như NGSIM, HighD, Waymo Open Dataset).  
   *Nếu giả định sai:* Nếu ngoài đời các tài xế liều lĩnh hơn (chấp nhận khe hở nhỏ hơn 12 m), tần suất va chạm tạt đầu cắt làn (`cut_in` và `bus_bypass`) ngoài đời thực sẽ **cao hơn nhiều** so với mô phỏng.
2. **Tỉ lệ người đi bộ ngó qua đầu xe buýt trước khi qua đường (`pedLook = 0.85`):**  
   *Dữ liệu thật thay thế:* Dữ liệu camera giám sát ngã tư/vạch qua đường đô thị, nghiên cứu hành vi người đi bộ qua đường (naturalistic pedestrian crossing studies).  
   *Nếu giả định sai:* Nếu thực tế người đi bộ thường xuyên cắm mặt vào điện thoại hoặc bị khuất tầm nhìn mà không thèm ngó (tỉ lệ ngó chỉ khoảng 50%), số vụ va chạm chặn lối `crosswalk_block` thực tế sẽ **tăng vọt** so với mô phỏng.
3. **Hệ số ma sát đường ướt (`wet = 0.20`, $\mu = 0.5$ thay vì $0.8$):**  
   *Dữ liệu thật thay thế:* Số liệu khí tượng thủy văn thực tế theo mùa của địa phương (tần suất ngày mưa) và thử nghiệm đo ma sát mặt đường (Skid Resistance Tests) của Cục Đường bộ.  
   *Nếu giả định sai:* Nếu thị trấn nằm ở vùng mưa nhiệt đới kéo dài (đường ướt chiếm 40–50%), cự ly phanh thực tế bị kéo dài thường xuyên hơn, khiến ước lượng an toàn hiện tại đang bị **lạc quan quá mức**.

---

## 8. Giới hạn

Ba điều mà xác suất báo cáo **KHÔNG** phản ánh:
1. **Hình học không gian quá đơn giản:** Thế giới mô phỏng chỉ là một đoạn đường thẳng duy nhất với 2 làn ngược chiều, không có giao lộ phức tạp, bùng binh, xe máy đi ngược chiều hay phương tiện dừng đỗ trái phép san sát.
2. **Hành vi tác nhân quá lý tưởng:** Các xe khác lái hoàn toàn theo mô hình IDM chuẩn tắc, chuyển làn theo hàm cosine mượt mà. Đời thực có các hành vi khó đoán như lạng lách bất ngờ, không xi-nhan, bấm còi vượt ẩu, hoặc tài xế say xỉn.
3. **Mô hình cảm biến lý tưởng:** Cảm biến xe ta nhận diện hoàn hảo vị trí và vận tốc của mọi đối tượng xung quanh (trừ góc khuất hình học sau xe buýt). Trong thực tế có nhiễu cảm biến, độ trễ xử lý của mô hình Object Detection/Tracking, điểm mù do mưa bụi bẩn bám camera/lidar.

*Nếu được thêm một tuần, bạn sẽ đo gì tiếp?*  
Tôi sẽ tích hợp **độ trễ nhận thức (perception latency)** biến thiên từ 100–300 ms và **nhiễu định vị (bounding box jitter)** vào planner, đồng thời mở rộng mô phỏng trên mạng lưới giao thông ngã tư đô thị có sự tham gia của xe máy hỗn hợp.

---

## 9. Rắc rối của bạn (bước G + chương 10)

* **Rắc rối:**  
  Tình huống **xe đạp đi ở làn ngoài (E2) bất ngờ lạng sang trái 2.2 – 2.8 m** (để tránh chướng ngại vật/hố ga trên đường) đúng lúc xe ta chạy tới gần ở làn E1 (cách 14 – 24 m). Ngoài đời, tình huống này cực kỳ phổ biến tại các đô thị giao thông hỗn hợp khi xe hai bánh tránh ổ gà, vũng nước hoặc phương tiện mở cửa xe bên lề.
* **Căn cứ vì sao planner xử lý kém:**  
  Planner của xe ta không có logic dự đoán quỹ đạo lạng của phương tiện hai bánh (`bike`) mà chỉ coi phương tiện cùng chiều là chướng ngại vật khi nó đã lọt vào tim làn (`cutinLat`). Khi xe đạp lạng nhanh ở cự ly gần (14–24 m), cự ly phanh của xe ta không đủ nếu chỉ kích hoạt phanh IDM trễ, dẫn đến va quẹt vào hông/đuôi xe đạp.
* **Giả định `space` và `exposure` (`hazard.json: why`):**  
  * `space`: `{"h0_x": [60.0, 100.0], "h0_swerve": [2.2, 2.8], "h0_trig": [14.0, 24.0]}`  
    Gia tốc ngang đỉnh của xe đạp: $a_{lat} = \pi^2 \cdot \text{swerve} / (2 \cdot swDur^2) = \pi^2 \cdot 2.8 / (2 \cdot 2.2^2) \approx 2.85\text{ m/s}^2 \le 3.0\text{ m/s}^2$, hoàn toàn khả thi về mặt động học và bám đường.
  * `exposure`: `0.03` (xảy ra khoảng 3% số chuyến đi đô thị, căn cứ theo số liệu khảo sát xung đột vi mô giữa xe cơ giới và xe hai bánh trong đô thị của NHTSA/FHWA).
* **Kết quả từ `mine.json`:**
  * Kịch bản cụ thể đã dựng (`exact`): Planner va chạm với `H0` tại $t = 12.55\text{ s}$ (`exact: planner = [('ego', 'H0', 12.55)]`).
  * Tỉ lệ kịch bản khả thi: `1493 / 1500` kịch bản (**100% accepted** $\ge 50\%$).
  * **So sánh chính sách gốc (`baseline`) và chính sách chọn (`chosen`, $T=2.0$):**
    * Chính sách gốc (`baseline`): Có **167 va chạm** = **165 tránh được (`avoidable`)** + **0 không tránh được (`unavoidable`)** + **2 không quy được (`unattributed`)** (trong đó có 15 vụ xảy ra ở vận tốc thấp $< 1$ m/s). $P(\text{lỗi planner} \mid \text{rắc rối}) = \mathbf{1.13 \times 10^{-1}}$ ($11.3\%$), khoảng Wilson $[9.77 \times 10^{-2}, 1.30 \times 10^{-1}]$. Tần suất mỗi chuyến đi: $\mathbf{3.39 \times 10^{-3}}$ lần/chuyến (cận trên $3.91 \times 10^{-3}$).
    * Chính sách đã chọn (`chosen`): Có **144 va chạm** = **144 tránh được (`avoidable`)** + **0 không tránh được** + **0 không quy được** (10 vụ ở $< 1$ m/s). $P(\text{lỗi planner} \mid \text{rắc rối}) = \mathbf{9.86 \times 10^{-2}}$ ($9.86\%$), khoảng Wilson $[8.43 \times 10^{-2}, 1.15 \times 10^{-1}]$. Tần suất mỗi chuyến đi: $\mathbf{2.96 \times 10^{-3}}$ lần/chuyến (cận trên $3.45 \times 10^{-3}$).
    * Bản sửa $T=2.0$ giúp giảm thiểu $13.8\%$ số ca lỗi va chạm với xe đạp lạng đường.
* **Vì sao "tránh được" (`avoidable`) là chuẩn lạc quan?**  
  - *Một mặt:* Người phanh chuẩn (oracle) biết ngay khi rắc rối bắt đầu xuất hiện trong tầm nhìn và lập tức đạp phanh kịch sàn ở giới hạn ma sát lốp xe ($\mu g$), không có độ trễ nhận thức hay thời gian tích áp phanh. Do đó, oracle tránh được nhiều hơn người lái/planner thật có thể làm $\implies$ `avoidable` **đếm nhiều hơn** số lỗi thực tế mà một hệ thống tự lái đời thực có thể cứu vãn.  
  - *Mặt khác:* Oracle chỉ thử nghiệm duy nhất hành vi phanh (không thử đánh lái né tránh). Trong thực tế, nhiều tình huống xe đạp lạng sang hoàn toàn có thể tránh được bằng cách lách nhẹ sang mép trái nếu làn đường còn trống. Việc không cho đánh lái làm nó đếm ít đi ở khía cạnh này.
* **Hình ảnh chụp từ chương 9/10:**  
  *Ảnh chụp màn hình:* `evidence/shots/hazard_mine.png`
* **Bạn tự lái thì sao:**  
  Khi tự lái thử ở Chương 10 bằng cách bấm "Bạn", tôi hoàn toàn tránh được va chạm. Khác biệt lớn nhất giữa phản xạ con người và planner là: Con người quan sát thấy xe đạp bắt đầu có xu hướng nghiêng bánh và lạng dần về bên trái từ cự ly xa nên đã chủ động nhả chân ga và rà phanh phòng ngừa từ trước, trong khi planner đợi xe đạp cắt hẳn vào giới hạn ngang tim làn mới bắt đầu tính toán phanh.
* **Cách sửa planner:**  
  - *Bằng núm chỉnh:* Không có núm chỉnh IDM đơn lẻ nào triệt tiêu hoàn toàn rắc rối này mà không làm tê liệt vận tốc của xe (ví dụ tăng khoảng đệm $T=2.0$ chỉ giảm từ $11.3\%$ xuống $9.86\%$).  
  - *Logic mới cần bổ sung:* Cần bổ sung module **Dự đoán ý định (Intent & Trajectory Prediction)** cho các đối tượng dễ bị tổn thương (VRU - Vulnerable Road Users). Khi phát hiện xe hai bánh có vận tốc ngang $v_{lat} > 0.3$ m/s hướng vào làn xe ta, hệ thống phải lập tức mở rộng hành lang giám sát an toàn (dynamic lateral buffer) và chủ động giảm tốc rà phanh (pre-braking) thay vì đợi đối tượng lọt vào tim đường mới phản ứng.
