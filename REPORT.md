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
| PercMLP+ (sau khi sửa) | 0.822 ± 0.009 (trung bình ± độ lệch chuẩn qua 5 seed) | 0.31 .. 0.35 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ở Bước 0, quá trình kiểm tra phát hiện ba vấn đề nghiêm trọng trong bộ dữ liệu OpenPAV: (1) Cột gia tốc xe dẫn `Acc_LV` bị lỗi cảm biến hoặc tính toán sai, độ tương quan tuyến tính với đạo hàm vận tốc d(Speed_LV)/dt gần như bằng 0 (chỉ đạt 0.0453), trong khi cột của xe theo sau `Acc_FAV` có độ tương quan rất cao là 0.9291. Do đó, ta tuyệt đối không dùng `Acc_LV` mà thay thế bằng gia tốc tính từ đạo hàm của vận tốc xe dẫn đã làm mượt `v_lead_s` (`a_lead`). (2) Có 383 quỹ đạo trùng lặp hoàn toàn (exact duplicates) dù mang Trajectory_ID cách xa nhau, khiến số quỹ đạo giảm từ 9993 xuống 9610; nếu không loại bỏ, các mẫu trùng có thể rơi vào cả tập huấn luyện và kiểm thử gây rò rỉ dữ liệu. (3) Dữ liệu bắt buộc phải chia theo từng quỹ đạo (trajectory) với tỷ lệ 70/30 (2883 quỹ đạo test trên tổng 9610) thay vì chia ngẫu nhiên theo từng khung hình (frame), nhằm tránh hiện tượng rò rỉ thông tin thời gian (temporal data leakage) do các khung hình liền kề trong cùng chuỗi có tương quan tự thân rất cao.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 khác bit ở toàn bộ 600/600 đoạn vì hai nguyên nhân kết hợp: vừa chuyển từ float64 sang float32 với độ chính xác biểu diễn thấp hơn nhiều, vừa thay đổi thứ tự các phép tính trong khi phép cộng dấu phẩy động không có tính kết hợp (ví dụ: `(0.1+0.2)+0.3 = 0.6000000000000001` khác `0.1+(0.2+0.3) = 0.6`). Tuy nhiên, số đoạn khác biệt vượt dung sai là 0/600 đoạn vì sai số tuyệt đối cực đại qua mọi lệnh chỉ là 1.22e-04 m/s², hoàn toàn nằm dưới ngưỡng dung sai atol = 1e-3. Bản AEB làm lệch 6/600 đoạn (gồm các đoạn 74, 290, 296, 450, 515, 575). Mở đoạn 74 bằng đoạn mã quan sát cho thấy nó chỉ lệch đúng 1 khung hình duy nhất là khung 82: tại đây khoảng cách gap = 6.16 m, vận tốc tiến gần closing = 4.20 m/s, dẫn đến TTC = 1.47 s < 1.5 s (ngưỡng AEB kích hoạt), khiến lệnh điều khiển chuyển từ -4.56 m/s² sang phanh khẩn cấp -6.00 m/s²; ở các khung khác TTC >= 1.5 s nên lệnh giữ nguyên. Chế độ phát nhanh nhất (fastest) gửi thông điệp kế tiếp ngay khi thông điệp trước xử lý xong, nên không tồn tại hạn chót: một khung xử lý chậm chỉ làm lần chạy lâu hơn chứ không làm đổi lệnh (0 khung trễ, 0 khung rớt). Chế độ thời gian thật (realtime) giao thông điệp đúng dấu thời gian trong log, mỗi khung cách nhau 0.1 s. Thỉnh thoảng bộ điều khiển xử lý một khung lâu hơn chu kỳ đó: khung ấy bị tính trễ (late) và lệnh cũ được giữ, rồi khung kế tiếp đến khi bộ điều khiển còn bận nên bị rớt (dropped). Vì vậy số khung trễ và rớt gần bằng nhau (582 và 578 trên 59816 khung). Chỉ một phần rất nhỏ khung gặp sự cố nhưng 379/600 đoạn đã đổi lệnh vượt dung sai, tức lỗi trễ hạn ảnh hưởng thật lên hành vi mà chế độ fastest không bao giờ thấy được.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cắt vào có tốc độ cruise v_ego = 14.0 m/s, khoảng cách từ 2.76 m đến 59.31 m, chênh lệch vận tốc dv từ 0.0 m/s đến 14.0 m/s, tổng cộng gồm 2700 ô (grid_cells). Tại độ trễ phản ứng t_react = 0.0 s, có 3.63% số ô là bất khả thi về mặt vật lý (unavoidable). Khi độ trễ phản ứng tăng lên t_react = 0.6 s, tỷ lệ ô không ai tránh được tăng lên 10.22%, tức tăng thêm 6.59 điểm phần trăm. Tỷ lệ va chạm của IDM tăng tương ứng từ 3.63% lên 10.22% (đúng bằng phần diện tích bất khả thi, diện tích lỗi quy trách nhiệm cho IDM attributable_area là 0.0%). Trong log thực tế, người lái xe sau phanh mạnh nhất (phân vị 99.9) chỉ đạt 3.62 m/s² (human_brake_limit_p99.9), dù xe dẫn trong 614 sự kiện khai thác được có lúc phanh mạnh hơn (gia tốc đỉnh của xe dẫn: trung vị -2.53 m/s², phân vị 99 là -4.21 m/s²); trong khi đó bộ chấp hành xe có thể phanh tới 9.0 m/s². Có tới 12.67% diện tích lưới kịch bản đòi hỏi gia tốc giảm tốc lớn hơn 3.62 m/s² (share_beyond_human_brake_limit = 0.1267). Toàn bộ 614 sự kiện xe dẫn phanh khai thác được từ 9610 quỹ đạo (chiếm 6.39%) đều phản ánh hành vi lái thông thường của con người, không thể bao phủ được 12.67% vùng kịch bản phanh khẩn cấp mà lưới đòi hỏi.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh bốn bộ điều khiển:
- const-vel: L2 = 1.273 m (hạng 3) | Tỷ lệ va chạm = 19.91% (hạng 3)
- IDM: L2 = 1.534 m (hạng 4) | Tỷ lệ va chạm = 0.10% (hạng 1)
- EgoMLP: L2 = 0.759 m (hạng 2) | Tỷ lệ va chạm = 25.72% (hạng 4, dải 18.07..31.39%)
- PercMLP: L2 = 0.626 m (hạng 1) | Tỷ lệ va chạm = 3.27% (hạng 2, dải 0.35..11.24%)
Có 4 cặp đảo thứ tự giữa vòng hở và vòng kín: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), và (IDM, PercMLP). IDM có sai số vòng hở L2 tệ nhất (1.534 m) vì nó tuân theo công thức động học lý tưởng thay vì bắt chước từng sai lệch ngẫu nhiên của con người, nhưng trong vòng kín nó phản hồi liên tục với khoảng cách và tốc độ tương đối nên giữ khoảng cách an toàn xuất sắc (chỉ đâm 0.10%). Ngược lại, EgoMLP đạt L2 tốt hơn const-vel (0.759 m vs 1.273 m) nhờ học phân phối vận tốc mượt mà, nhưng nó hoàn toàn không có thông tin về khoảng cách và xe dẫn; trong vòng kín khi xe dẫn phanh, EgoMLP không hề biết để giảm tốc nên đâm tới 25.72% (thậm chí 56.1% trên lát cắt xe phanh). Cặp (IDM, PercMLP) đảo vì một cơ chế khác: PercMLP có nhìn thấy gap và vận tốc xe dẫn nên có L2 tốt nhất (0.626 m, kể cả trên lát cắt xe dẫn phanh là 0.861 m so với 1.642 m của IDM), nhưng L2 chỉ đo sai số trên những trạng thái người lái đã đi qua. Khi tự lái, mỗi sai số nhỏ đẩy xe sang trạng thái hơi khác log, sai số sau cộng dồn lên sai số trước, và mô hình rơi vào vùng chưa từng được huấn luyện: ở seed xấu nhất, khoảng cách nhỏ nhất trung vị chỉ còn 6.66 m so với 9.36 m của IDM và 9.45 m của người lái, còn va chạm trên lát cắt phanh lên 28.65%. IDM là luật phản hồi theo trạng thái hiện tại nên không bị cộng dồn sai số như vậy.

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

PercMLP bị chặn bởi 5 quy tắc: R2 (cận trên CI95 trên log là 12.44% > baseline 0.10% + 1.0%), R3 (va chạm lát cắt phanh là 28.65%, cận trên 35.84% > 0.6% + 5.0%), R4 (diện tích va chạm tránh được trên lưới là 34.63% > baseline 0.0% + 2.0%), R5 (độ phân tán qua 5 seed là 0.35..11.24% chênh lệch 10.89% > 1.0%), và R6 (lưới phản ứng đâm 44.85% > baseline 10.22% + 2.0%). Có hai nguyên nhân gốc: (1) Thiếu hụt kịch bản: log thực tế không chứa các ca cắt đầu gấp ở cự ly gần (gây lỗi R4, R6); (2) Sai số tích lũy (compounding error) trong vòng kín khiến mô hình trôi dạt vào trạng thái lạ chưa từng thấy trong log (gây lỗi R2, R3, R5). Ta sửa bằng cách bổ sung cửa sổ huấn luyện từ 20000 kịch bản cắt vào mô phỏng (chữa nguyên nhân 1) kết hợp 2 vòng DAgger ghi nhận trạng thái mô hình tự lái và gán nhãn bằng luật IDM (chữa nguyên nhân 2). Sau sửa, PercMLP+ đạt RELEASE với va chạm log chỉ 0.31..0.35% và diện tích lưới 0.44%. Đáng chú ý, L2 vòng hở của PercMLP+ lại tệ hơn PercMLP (0.822 m so với 0.626 m) trong khi va chạm vòng kín giảm từ 0.35..11.24% xuống 0.31..0.35%: mô hình được dạy làm theo IDM ở các trạng thái lạ nên bắt chước người kém đi. Điểm vòng hở ở đây đi ngược với độ an toàn, nên không thể dùng nó để chọn mô hình. PercMLP+ KHÔNG tốt hơn IDM, nó chỉ "không kém hơn" trong biên cho phép của cổng (cận trên 0.64% ≤ 0.10% + 1.0%): va chạm log 0.35% so với 0.10%, diện tích lưới 0.44% so với 0.00%, và ở tốc độ cruise 20 m/s chưa dùng khi đánh giá (`fix.json`), diện tích va chạm do lỗi của PercMLP+ là 0.22..1.07% qua 5 seed so với 0.0% của IDM (PercMLP là 17.70..44.85%). Nó học nhãn từ chính IDM nên giỏi lắm chỉ bằng thầy, và mọi số đều cho thấy nó vẫn kém thầy một chút. Vì vậy ta nên giữ IDM làm bộ điều khiển chính; nếu muốn tiếp tục với PercMLP+, chỉ nên chạy ở chế độ bóng (shadow mode) song song với IDM để thu thêm dữ liệu, kèm bộ lọc an toàn dự phòng (safety fallback filter), trước khi cân nhắc phát hành lại.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Bốn giới hạn quan trọng nhất đối với quyết định RELEASE của PercMLP+: (1) Chuyển động chỉ một chiều dọc (1D longitudinal): mô hình bỏ qua điều khiển ngang, góc lái, độ bám lốp và khả năng đánh lái né; trong thực tế phanh tới 9.0 m/s² trên đường trơn có thể làm xe trượt bánh, mất lái. Cả biên bất khả thi lẫn diện tích va chạm 0.44% đều được tính trong một thế giới dễ hơn thật, nên kết quả RELEASE chỉ đúng cho bài toán bám xe trên một làn. (2) Cảm biến lý tưởng hóa: gap và vận tốc xe dẫn được cho đúng, hoặc chỉ cộng nhiễu Gaussian (độ lệch 1.0 m và 0.5 m/s); không có phát hiện nhầm (ghost braking), mất dấu do che khuất hay ảnh hưởng thời tiết. PercMLP+ dựa hoàn toàn vào hai đầu vào này nên rủi ro thật của nó lớn hơn số đo được. (3) Tác nhân đơn giản hóa: xe dẫn chỉ phát lại log, xe sau phản ứng theo một luật IDM duy nhất với T = 1.4 s; chưa có xe máy luồn lách, tài xế mất tập trung hay hung hăng. Câu 6 cho thấy chỉ riêng việc đổi mô hình xe sau đã làm va chạm của IDM đổi từ 80.9% xuống 10.2%, nên kết luận của cổng còn rất nhạy với giả định về tác nhân. (4) Phép đo vòng tròn và hẹp: baseline của cổng (IDM) cũng chính là thầy gán nhãn cho PercMLP+, nên cổng chỉ hỏi "bắt chước IDM có đủ sát không" chứ không phát hiện được điểm mù chung của cả hai; lưới kịch bản cũng chỉ ở v_ego = 14 m/s, và khi thử ở 20 m/s PercMLP+ đã có 0.22..1.07% diện tích va chạm do lỗi so với 0.0% của IDM. Để thu hẹp khoảng cách: kiểm thử trên lưới nhiều tốc độ và nhiều họ kịch bản hơn, dùng mô phỏng 3D đa làn (như CARLA) với chuỗi cảm biến thật, đưa vào Hardware-in-the-Loop, chạy chế độ bóng trên xe thật song song với IDM, rồi thử nghiệm trên đường thử khép kín với bia mềm trước khi phát hành.
