# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Ngô Đức Chung |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602985 |
| GitHub | chung170703 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split_seed: 2657 / model_seed: 5286 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.273 | 19.91 | BLOCK |
| IDM | 1.534 | 0.10 | (baseline) |
| EgoMLP | 0.754 .. 0.764 | 18.07 .. 31.39 | BLOCK |
| PercMLP | 0.616 .. 0.636 | 0.35 .. 11.24 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.813 .. 0.831 | 0.31 .. 0.35 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ở Bước 0, quá trình kiểm tra phát hiện ba vấn đề nghiêm trọng trong bộ dữ liệu OpenPAV: (1) Cột gia tốc xe dẫn `Acc_LV` bị lỗi cảm biến hoặc tính toán sai, độ tương quan tuyến tính với đạo hàm vận tốc d(Speed_LV)/dt gần như bằng 0 (chỉ đạt 0.0453), trong khi cột của xe theo sau `Acc_FAV` có độ tương quan rất cao là 0.9291. Do đó, ta tuyệt đối không dùng `Acc_LV` mà thay thế bằng gia tốc tính từ đạo hàm của vận tốc xe dẫn đã làm mượt `v_lead_s` (`a_lead`). (2) Có 383 quỹ đạo trùng lặp hoàn toàn (exact duplicates) dù mang Trajectory_ID cách xa nhau, khiến số quỹ đạo giảm từ 9993 xuống 9610; nếu không loại bỏ, các mẫu trùng có thể rơi vào cả tập huấn luyện và kiểm thử gây rò rỉ dữ liệu. (3) Dữ liệu bắt buộc phải chia theo từng quỹ đạo (trajectory) với tỷ lệ 70/30 (6727 train / 2883 test) thay vì chia ngẫu nhiên theo từng khung hình (frame), nhằm tránh hiện tượng rò rỉ thông tin thời gian (temporal data leakage) do các khung hình liền kề trong cùng chuỗi có tương quan tự thân rất cao.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 khác bit ở toàn bộ 600/600 đoạn do phép cộng dấu phẩy động không có tính kết hợp (ví dụ: `(0.1+0.2)+0.3 = 0.6000000000000001` khác `0.1+(0.2+0.3) = 0.6`), nhưng khác biệt vượt dung sai là 0/600 đoạn vì sai số tuyệt đối cực đại chỉ là 1.22e-04 m/s², nằm hoàn toàn dưới ngưỡng atol = 1e-3. Bản AEB làm lệch 6/600 đoạn (gồm các đoạn 74, 290, 296, 450, 515, 575). Mở đoạn 74 bằng đoạn mã quan sát cho thấy nó chỉ lệch đúng 1 khung hình duy nhất là khung 82: tại đây khoảng cách gap = 6.16 m, vận tốc tiến gần closing = 4.20 m/s, dẫn đến TTC = 1.47 s < 1.5 s (ngưỡng AEB kích hoạt), khiến lệnh điều khiển chuyển từ -4.56 m/s² sang phanh khẩn cấp -6.00 m/s²; ở các khung khác TTC >= 1.5 s nên lệnh giữ nguyên. Chế độ phát nhanh nhất (fastest) che giấu toàn bộ vấn đề vi phạm hạn chót (0 khung trễ, 0 khung rớt), trong khi phát theo thời gian thực (realtime) làm lộ ra 582 khung trễ (late) và 578 khung bị rớt (dropped) trên tổng số 59816 khung hình, khiến 379/600 đoạn bị thay đổi hành vi điều khiển.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cắt vào có tốc độ cruise v_ego = 14.0 m/s, khoảng cách từ 2.76 m đến 59.31 m, chênh lệch vận tốc dv từ 0.0 m/s đến 14.0 m/s, tổng cộng gồm 60 x 45 = 2700 ô. Tại độ trễ phản ứng t_react = 0.0 s, có 3.63% số ô (98 ô) là bất khả thi về mặt vật lý (unavoidable). Khi độ trễ phản ứng tăng lên t_react = 0.6 s, tỷ lệ ô không ai tránh được tăng lên 10.22% (276 ô), tức tăng thêm 6.59 điểm phần trăm. Tỷ lệ va chạm của IDM tăng tương ứng từ 3.63% lên 10.22% (đúng bằng phần diện tích bất khả thi, diện tích lỗi quy trách nhiệm cho IDM là 0.0%). Trong log thực tế, người lái phanh mạnh nhất (phân vị 99.9) chỉ đạt 3.62 m/s², trong khi bộ chấp hành có thể đạt 9.0 m/s². Có tới 12.67% diện tích lưới kịch bản đòi hỏi gia tốc giảm tốc lớn hơn 3.62 m/s², và 614 sự kiện xe dẫn phanh khai thác được từ 9610 quỹ đạo (chiếm 6.4%) chỉ phân bố ở vùng khoảng cách an toàn, hoàn toàn không bao phủ được các tình huống cắt vào khẩn cấp ở cự ly gần.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh bốn bộ điều khiển:
- const-vel: L2 = 1.273 m (hạng 3) | Tỷ lệ va chạm = 19.91% (hạng 3)
- IDM: L2 = 1.534 m (hạng 4) | Tỷ lệ va chạm = 0.10% (hạng 1)
- EgoMLP: L2 = 0.759 m (hạng 2) | Tỷ lệ va chạm = 25.72% (hạng 4, dải 18.07..31.39%)
- PercMLP: L2 = 0.626 m (hạng 1) | Tỷ lệ va chạm = 3.27% (hạng 2, dải 0.35..11.24%)
Có 4 cặp đảo thứ tự giữa vòng hở và vòng kín: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), và (IDM, PercMLP). IDM có sai số vòng hở L2 tệ nhất (1.534 m) vì nó tuân theo công thức động học lý tưởng thay vì bắt chước từng sai lệch ngẫu nhiên của con người, nhưng trong vòng kín nó phản hồi liên tục với khoảng cách và tốc độ tương đối nên giữ khoảng cách an toàn xuất sắc (chỉ đâm 0.10%). Ngược lại, EgoMLP đạt L2 tốt hơn const-vel (0.759 m vs 1.273 m) nhờ học phân phối vận tốc mượt mà, nhưng nó hoàn toàn không có thông tin về khoảng cách và xe dẫn; trong vòng kín khi xe dẫn phanh, EgoMLP không hề biết để giảm tốc nên đâm tới 25.72% (thậm chí 56.1% trên lát cắt xe phanh).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm vòng kín của EgoMLP dao động rất mạnh qua 5 seed, trải dài từ 18.07% (seed 5290) đến 31.39% (seed 5289), tức biên độ chênh lệch lên tới 13.32 điểm phần trăm. Tương tự, PercMLP dao động từ 0.35% (seed 5290) đến 11.24% (seed 5288), chênh lệch 10.89 điểm phần trăm. Khoảng tin cậy Wilson 95% cho seed tốt nhất của EgoMLP là [16.71%, 19.52%] và seed xấu nhất là [29.72%, 33.11%]; hai khoảng này hoàn toàn tách rời nhau (disjoint). Nếu chỉ chạy một seed duy nhất, người đánh giá có thể may mắn gặp seed tốt (như PercMLP 0.35%) rồi kết luận sai lầm rằng mô hình học sâu an toàn tương đương baseline IDM (0.10%), hoặc rơi vào seed xấu (11.24%) và kết luận mô hình hoàn toàn phế thải. Việc chỉ dựa vào một lần chạy duy nhất là cực kỳ nguy hiểm trong kỹ thuật an toàn vì nó che giấu sự bất ổn định ngẫu nhiên trong khởi tạo tham số và quá trình tối ưu hóa.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh trên lưới kịch bản tại t_react = 0.6 s (tỷ lệ va chạm chung và tỷ lệ va chạm đuôi):
- const-vel: xe sau phát lại 80.6% (rear 0.0%) | xe sau phản ứng 80.6% (rear 0.0%)
- IDM: xe sau phát lại 80.9% (rear 70.6%) | xe sau phản ứng 10.2% (rear 0.0%)
- EgoMLP: xe sau phát lại 81.1% (rear 0.0%) | xe sau phản ứng 81.1% (rear 0.0%)
- PercMLP: xe sau phát lại 75.3% (rear 36.9%) | xe sau phản ứng 44.9% (rear 0.0%)
- PercMLP+: xe sau phát lại 81.1% (rear 70.6%) | xe sau phản ứng 10.7% (rear 0.0%)
Tham số thời gian giãn cách được hiệu chỉnh là T = 1.4 s với khoảng cách Wasserstein W1 = 0.35 so với phân phối người thật (trung vị 1.87 s), vượt trội so với tác nhân không phản ứng (W1 = 0.86). Khi xe sau chỉ phát lại log, nó duy trì tốc độ cũ bất chấp xe thử nghiệm phía trước đang phanh gấp để tránh xe cắt đầu; điều này tạo ra 70.6% "va chạm giả" từ phía sau cho IDM, khiến IDM bị tính tổng va chạm lên tới 80.9% (tệ hơn cả const-vel 80.6%). Khi chuyển sang xe sau biết phản ứng, 70.6% va chạm giả biến mất hoàn toàn, giúp IDM lộ rõ bản chất an toàn với tỷ lệ va chạm thực chỉ là 10.2% (bằng đúng giới hạn bất khả thi vật lý).

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi 5 quy tắc: R2 (cận trên CI95 trên log là 12.44% > baseline 0.10% + 1.0%), R3 (va chạm lát cắt phanh là 28.65%, cận trên 35.84% > 0.6% + 5.0%), R4 (diện tích va chạm tránh được trên lưới là 34.63% > baseline 0.0% + 2.0%), R5 (độ phân tán qua 5 seed là 0.35..11.24% chênh lệch 10.89% > 1.0%), và R6 (lưới phản ứng đâm 44.85% > baseline 10.22% + 2.0%). Có hai nguyên nhân gốc: (1) Thiếu hụt kịch bản: log thực tế không chứa các ca cắt đầu gấp ở cự ly gần (gây lỗi R4, R6); (2) Sai số tích lũy (compounding error) trong vòng kín khiến mô hình trôi dạt vào trạng thái lạ chưa từng thấy trong log (gây lỗi R2, R3, R5). Ta sửa bằng cách bổ sung 197080 cửa sổ từ 20000 kịch bản cắt vào mô phỏng (chữa nguyên nhân 1) kết hợp 2 vòng DAgger ghi nhận trạng thái mô hình tự lái và gán nhãn bằng luật IDM (chữa nguyên nhân 2). Sau sửa, PercMLP+ đạt RELEASE với va chạm log chỉ 0.31..0.35% và diện tích lưới 0.44%. Tuy nhiên PercMLP+ KHÔNG tốt hơn IDM (va chạm log 0.35% vs 0.10%, diện tích lưới 0.44% vs 0.00%) vì nó học từ chính IDM. Chưa nên phát hành ngay mà cần bổ sung hành lang an toàn (safety fallback filter) và thử nghiệm thêm trên xe thật.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn quan trọng nhất của bài lab làm suy giảm độ tin cậy của quyết định phát hành gồm: (1) Giới hạn chuyển động một chiều dọc (1D longitudinal dynamics): Mô hình hoàn toàn bỏ qua tương tác ngang (lateral control), góc lái, bám đường của lốp và khả năng đánh lái né tránh chướng ngại vật; trong thực tế việc phanh gấp ở gia tốc 9.0 m/s² trên đường trơn trượt có thể gây lật xe hoặc mất lái nghiêm trọng. (2) Chuỗi cảm biến lý tưởng hóa (no real perception stack): Hệ thống giả định biết trước khoảng cách gap và vận tốc xe dẫn một cách hoàn hảo hoặc chỉ thêm nhiễu Gaussian đơn giản, bỏ qua các vấn đề cốt tử như phát hiện nhầm chướng ngại vật (ghost braking), mất dấu mục tiêu (occlusion/false negatives), nhiễu ánh sáng ngược hay camera bị mờ do thời tiết. (3) Tác nhân giao thông đơn giản hóa (oversimplified agent behaviors): Xe dẫn trong log không hề phản ứng với xe theo sau, và xe sau phản ứng chỉ tuân theo một luật IDM 1D cố định với T = 1.4 s, chưa bao quát được các hành vi con người phức tạp như xe máy luồn lách, tài xế mất tập trung hoặc tài xế hung hăng. Để thu hẹp khoảng cách, cần kiểm thử trên mô phỏng 3D đa làn (như CARLA), đưa mô hình vào môi trường phần cứng thật (Hardware-in-the-Loop), và thử nghiệm thực địa trên đường thử khép kín với các bia mục tiêu mềm trước khi triển khai thương mại.
