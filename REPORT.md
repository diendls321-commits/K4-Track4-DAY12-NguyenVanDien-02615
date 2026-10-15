# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Văn Diện |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2202602615 |
| GitHub | diendls321-commits |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 5706 / [7508, 7509, 7510] |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.279 m | 20.6 % (CI95 [19.2, 22.2] %) | BLOCK |
| IDM | 1.573 m | 0.1 % (CI95 [0.05, 0.36] %) | RELEASE (baseline) |
| EgoMLP | 0.772 ±0.003 m (0.769..0.777 m) | 7.0..27.4 % (worst seed CI95 [25.8, 29.0] %) | BLOCK |
| PercMLP | 0.628 ±0.001 m (0.627..0.630 m) | 1.1..1.3 % (worst seed CI95 [1.0, 1.8] %) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.823 ±0.008 m (0.814..0.830 m) | 0.2..0.3 % (worst seed CI95 [0.16, 0.59] %) | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Quá trình kiểm toán dữ liệu ở Bước 0 từ 992.679 dòng phát hiện ba vấn đề trọng yếu: (1) Trùng lặp dữ liệu: có 383/9.993 quỹ đạo thô bị trùng lặp hoàn toàn nội dung dù nằm ở các ID cách xa nhau, sau khi khử trùng còn 9.610 quỹ đạo sạch; (2) Cột gia tốc xe dẫn `Acc_LV` bị lỗi cảm biến hoặc sai số tính toán nghiêm trọng: độ tương quan tuyến tính giữa `Acc_LV` và đạo hàm vận tốc d(Speed_LV)/dt chỉ đạt 0.045 (gần bằng 0, trong khi xe theo sau đạt 0.929). Do đó không thể dùng `Acc_LV` để khai thác ca phanh hay giám sát, mà phải thay bằng gia tốc đạo hàm từ vận tốc xe dẫn đã làm mượt `v_lead_s` qua lọc tín hiệu số lưu trong `a_lead`; (3) Khoảng cách không gian `Spatial_Gap` thực tế là khoảng cách bumper-to-bumper lệch đúng bằng chiều dài thân xe 4.50 m (độ lệch chuẩn cực đại 3.64e-11 m). Việc phân chia train/test bắt buộc phải chia theo quỹ đạo (mỗi quỹ đạo dài 70-109 bước tương đương 7-11 giây ở tần số 10 Hz) thay vì chia theo khung hình, bởi vì các khung hình kề nhau trong cùng một quỹ đạo có tính tự tương quan thời gian cực kỳ mạnh; nếu chia ngẫu nhiên theo khung hình sẽ gây rò rỉ dữ liệu (data leakage) nghiêm trọng giữa tập huấn luyện và kiểm thử.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Khi phát lại 600 đoạn log (9.4 MB) ở Bước 1: bản refactor float32 khác bit ở 600/600 đoạn (100% không khớp từng bit do tính chất không kết hợp của số thực: (0.1 + 0.2) + 0.3 = 0.6000000000000001 != 0.1 + (0.2 + 0.3) = 0.6 với độ lệch tương đối 6.94e-06), nhưng có 0/600 đoạn vượt ngưỡng dung sai (atol=0.001, rtol=0.001, sai lệch cực đại chỉ 1.84e-05 m/s²), chứng minh việc đòi hỏi bit-identical là sai lầm mà cần so sánh có dung sai. Khi bổ sung tính năng phanh khẩn cấp AEB (ngưỡng TTC < 1.5 s), có đúng 9/600 đoạn bị thay đổi hành vi (các đoạn 60, 78, 140, 187, 194, 246, 249, 503, 569, độ lệch cực đại 1.84 m/s²). Mở chi tiết đoạn 60 cho thấy chỉ có đúng 4 khung hình (khung 98-101) bị lệch: tại khung 98, khoảng cách gap = 4.30 m, closing speed = 2.92 m/s dẫn tới TTC = 1.47 s (< 1.5 s), khiến AEB kích hoạt phanh tối đa -6.00 m/s² thay vì mức -5.44 m/s² của IDM nguyên bản. Cuối cùng, chế độ phát nhanh nhất (fastest replay) hoàn toàn bỏ qua trục thời gian và hạn định phần cứng nên che giấu toàn bộ lỗi (0 khung trễ, 0 khung rớt), trong khi chế độ thời gian thật (realtime replay) làm lộ ra 582 khung trễ và 581 khung rớt trên tổng số 59.688 khung hình, khiến 377/600 đoạn bị thay đổi hành vi điều khiển với độ lệch gia tốc lên tới 11.25 m/s².

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cắt vào được thiết lập ở vận tốc cruise 12.0 m/s với 60 mốc khoảng cách (2.76 m đến 59.31 m) và 45 mốc vận tốc tiếp cận (0.0 m/s đến 12.0 m/s), tạo thành 2.700 ô thử nghiệm. Theo vật lý phanh khẩn cấp ở mức tối đa b_max = 9.0 m/s², khi độ trễ phản ứng t_react = 0.0 s có 2.2% số ô (59/2.700 ô) là bất khả thi vật lý (không một bộ điều khiển nào có thể tránh đâm); khi độ trễ tăng lên t_react = 0.6 s, vùng bất khả thi tăng vọt lên 7.4% (201/2.700 ô, tăng thêm 5.2% diện tích lưới do xe mất thêm quãng đường co lại dv * 0.6 s trước khi bắt đầu phanh). Vùng va chạm của IDM tăng tương ứng từ 2.2% lên 7.4%, nghĩa là diện tích va chạm quy trách nhiệm (attributable collision area) của IDM duy trì ở mức tuyệt đối 0.0% với 0 ô sai lệch so với biên vật lý ở cả hai độ trễ. Về độ bao phủ, dữ liệu log thật hoàn toàn không phủ vùng nguy hiểm: mức phanh gắt nhất của con người trong log (phân vị 99.9%) chỉ đạt 3.62 m/s² (trong 614 sự kiện khai thác, trung vị phanh p50 chỉ là -2.53 m/s² và p99 là -4.21 m/s²), trong khi có tới 8.56% diện tích lưới kịch bản đòi hỏi gia tốc giảm tốc lớn hơn 3.62 m/s² để không đâm.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh hiệu năng giữa đánh giá vòng hở (L2 position error) và vòng kín (tỷ lệ va chạm trên 2.883 quỹ đạo kiểm thử):
| Bộ điều khiển | L2 vòng hở (m) | Hạng vòng hở | Va chạm vòng kín (%) | Hạng vòng kín |
|---|---|---|---|---|
| PercMLP | 0.628 m | 1 | 1.2 % (1.1..1.3 %) | 2 |
| EgoMLP | 0.772 m | 2 | 20.3 % (7.0..27.4 %) | 3 |
| constant-velocity | 1.279 m | 3 | 20.6 % | 4 |
| IDM | 1.573 m | 4 | 0.1 % (0.1..0.4 %) | 1 |

Có 3 cặp đổi đảo thứ hạng hoàn toàn giữa hai cách chấm: (const-vel, IDM), (IDM, EgoMLP), và (IDM, PercMLP). Nguyên nhân IDM có L2 vòng hở đội sổ (1.573 m) nhưng vòng kín lại dẫn đầu tuyệt đối (chỉ đâm 0.1%) là vì IDM là thuật toán điều khiển phản hồi theo quy luật vật lý với khoảng cách an toàn mục tiêu, nó không học giám sát để khớp từng milimet quỹ đạo người lái bình thường (vốn có độ trễ và nhiễu), nhưng khi chạy vòng kín nó liên tục điều chỉnh gia tốc để triệt tiêu sai số và bảo toàn khoảng cách an toàn. Ngược lại, EgoMLP đạt L2 vòng hở rất đẹp (0.772 m, tốt hơn nhiều so với const-vel 1.279 m) nhờ ghi nhớ được phân bố gia tốc mượt mà của xe chủ, nhưng trong vòng kín lại đâm tới 20.3% (và đâm 62.6% trên lát cắt xe dẫn phanh) vì EgoMLP bị mù hoàn toàn trước ngoại cảnh (không nhận tín hiệu gap và v_lead); khi gặp tình huống xe dẫn giảm tốc, EgoMLP không hề nhận biết được chướng ngại vật phía trước và tích lũy sai số vị trí dẫn đến tai nạn tất yếu.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Khi huấn luyện EgoMLP trên 3 seed khác nhau (7508, 7509, 7510), tỷ lệ va chạm vòng kín dao động cực kỳ lớn: từ mức 7.0% (202/2.883 ca ở seed 7510) lên đến 26.5% (765 ca ở seed 7508) và đỉnh điểm là 27.4% (789 ca ở seed 7509), biên độ dao động lên tới 20.4 điểm phần trăm (vượt xa giới hạn ổn định 1.0% của quy tắc R5). Khoảng tin cậy Wilson 95% chỉ ra rằng: seed 7510 có khoảng CI95 là [6.1, 8.0] %, hoàn toàn tách rời và không hề giao thoa với khoảng CI95 của seed 7509 là [25.8, 29.0] %. Điều này khẳng định sự chênh lệch này là do tính ngẫu nhiên trong khởi tạo trọng số và hội tụ cực trị địa phương của mạng nơ-ron chứ không phải do phương sai mẫu kiểm thử. Nếu kỹ sư chỉ chạy đúng một lần và rơi vào seed 7510, họ sẽ lầm tưởng mô hình có tỷ lệ va chạm chấp nhận được (7.0%) và vội vã đưa ra quyết định phát hành; trong khi thực tế mô hình hoàn toàn có thể hội tụ vào điểm kỳ dị nguy hiểm với tỷ lệ đâm lên tới 27.4% (gấp gần 4 lần). Báo cáo một seed duy nhất sẽ che giấu bản chất bất ổn định của thuật toán học sâu và tạo ra rủi ro an toàn chết người trong hệ thống tự hành.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh tỷ lệ va chạm trên lưới kịch bản cắt vào ở độ trễ phản ứng t_react = 0.6 s:
| Bộ điều khiển | Xe sau phát lại (%) | Xe sau phản ứng (%) | Va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| constant-velocity | 77.6 % | 77.6 % | 0.0 % / 0.0 % |
| IDM | 75.7 % | 7.4 % | 68.3 % / 0.0 % |
| EgoMLP | 78.2 % | 78.2 % | 0.0 % / 0.0 % |
| PercMLP | 78.1 % | 26.0 % | 52.7 % / 0.0 % |
| PercMLP+ | 75.6 % | 8.1 % | 67.4 % / 0.0 % |

Thời khoảng cách hiệu chỉnh tối ưu cho tác nhân xe sau IDM là `T = 1.4 s` (đạt khoảng cách Wasserstein 1-D W1 = 0.34 so với phân bố headway trung vị 1.86 s của người lái thật trong log). Giá trị này chân thực hơn hẳn tác nhân không phản ứng (never-reacting agent có W1 = 0.89, tức giảm sai lệch phân bố hơn 2.6 lần). Hiện tượng "va chạm giả" xảy ra khi xe sau chỉ phát lại một cách thụ động: do xe sau di chuyển mù quáng theo đúng vận tốc cũ trong log mà không biết xe chủ (IDM) đã phanh khẩn cấp để tránh va chạm với xe cắt vào phía trước, xe sau đâm sầm vào đuôi xe chủ trong 68.3% diện tích lưới (khiến tổng va chạm của IDM bị đội lên 75.7%, nhìn qua tưởng chừng nguy hiểm ngang const-vel 77.6%). Khi chuyển sang môi trường xe sau phản ứng thông minh, số vụ đâm đuôi của IDM giảm ngay về 0.0%, và tổng va chạm chỉ còn đúng 7.4% (bằng đúng biên bất khả thi vật lý). Điều này đảo ngược kết luận: IDM thực tế không hề nguy hiểm mà là bộ điều khiển phanh hoàn hảo; các vụ va chạm trước đó hoàn toàn là lỗi giả do mô hình hóa môi trường thiếu tính phản ứng.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi 4 quy tắc nghiêm ngặt tại cổng kiểm thử: R2 (va chạm log 1.3 %, cận trên CI95 là 1.8 % vượt ngưỡng baseline 0.1 % + 1.0 % = 1.1 %); R3 (va chạm lát cắt phanh 4.4 %, cận trên 8.2 % vượt ngưỡng 1.0 % + 5.0 % = 6.0 %); R4 (diện tích va chạm tránh được trên lưới cắt vào là 18.5 % vượt xa 0.0 % + 2.0 %); R6 (va chạm với xe sau phản ứng là 26.0 % vượt mức 7.4 % + 2.0 % = 9.4 %). Có hai nguyên nhân gốc: (1) Lệch phân bố dữ liệu (Covariate Shift/OOD) do log thật hoàn toàn thiếu các ca cắt vào đột ngột hay phanh sát sườn (gây rớt R4, R6); (2) Tích lũy sai số khi tự lái vòng kín khiến mô hình trôi dần vào các trạng thái chưa từng thấy (gây rớt R2, R3). Phương pháp sửa gồm hai kỹ thuật: bổ sung 197.220 cửa sổ từ 20.000 ca cắt vào mô phỏng gắn nhãn bởi IDM để bao phủ vùng OOD, kết hợp 2 vòng thuật toán DAgger ghi nhận 894.600 cửa sổ tại các trạng thái mô hình tự đi vào để học cách phục hồi sai số. Sau khi sửa, PercMLP+ vượt qua cả 7 quy tắc và được RELEASE (R2 giảm còn 0.3 % với cận trên 0.6 % <= 1.1 %; R3 giảm còn 1.0 %; R4 giảm ngoạn mục từ 18.5 % xuống 0.7 %; R6 giảm còn 8.1 %). Tuy nhiên, PercMLP+ KHÔNG HỀ tốt hơn IDM (IDM đạt 0.1 % trên log và 0.0 % va chạm tránh được trên lưới, trong khi PercMLP+ là 0.2..0.3 % trên log và 0.7 % trên lưới); mô hình học chỉ đạt mức "không kém hơn" (non-inferiority). PercMLP+ học hoàn toàn từ nhãn của IDM nên không thể vượt qua chuyên gia đã dạy nó. Do đó KHÔNG NÊN phát hành ngay: khi kiểm tra ở tốc độ cruise chưa từng học (20 m/s), PercMLP+ vẫn đâm 0.0..2.6% trong khi IDM là 0.0%. Cần tiếp tục thử nghiệm thêm các kịch bản thời tiết, cảm biến có độ trễ lớn và chạy kiểm thử thực địa (proving ground) trước khi xem xét phát hành.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn cốt lõi làm giảm độ tin cậy của bài lab đối với quyết định phát hành sản phẩm thật gồm:
1. Mô hình động học một chiều dọc giản lược (1-D longitudinal simplification): Hệ thống chỉ mô phỏng chuyển động tiến/lùi trên một trục thẳng, bỏ qua hoàn toàn động lực học quay vòng, gia tốc ngang, góc nghiêng mặt đường, sự suy giảm độ bám lốp (friction coefficient) và khả năng đánh lái tránh chướng ngại vật (evasive steering). Trong thực tế, phanh ở mức 9.0 m/s² trên đường trơn có thể gây bó cứng phanh, trượt bánh hoặc lật xe mà mô hình 1-D không thể phát hiện.
2. Khoảng cách hiện thực về hệ thống cảm biến và nhận thức (Perception & Sensor Reality Gap): Mô hình nhận đầu vào là trạng thái lý tưởng được trích xuất từ log hoặc mô phỏng với nhiễu Gaussian đơn giản (gap ±1.0 m, lead speed ±0.5 m/s). Trên xe tự hành thực tế, pipeline nhận thức từ Camera/LiDAR/Radar đối mặt với hiện tượng mất dấu mục tiêu (false negatives), phát hiện ảo (ghost braking), trễ truyền thông CAN bus, vật cản bị che khuất và sự suy giảm thị giác do mưa, sương mù hay chói sáng ngược chiều.
3. Hành vi tác nhân xung quanh chưa đủ phức tạp (Over-simplified Traffic Interactions): Trong bài lab, xe dẫn chỉ phát lại thụ động từ log quá khứ mà không có phản ứng tương hỗ, còn xe sau chỉ là một luật IDM đơn thuần với một độ trễ cố định (0.6 s). Giao thông thực tế là một hệ thống xã hội phức hợp với các tác nhân phi tuyến tính như xe máy luồn lách, người đi bộ băng qua đường ẩu, xe tải lớn và các phản xạ cảm xúc/bất ngờ của tài xế con người.

Để thu hẹp khoảng cách đến thế giới thật trước khi phát hành, nhóm phát triển cần: (1) Đưa mô hình vào các bộ mô phỏng 3D vật lý chính xác cao (như CARLA hoặc SVL) tích hợp mô hình lốp Pacejka và cảm biến ảo chân thực; (2) Chạy kiểm thử chế độ bóng (Shadow Mode) trên đội xe thương mại thực tế hàng triệu km để so sánh quyết định của mô hình với tài xế an toàn; (3) Thực hiện kiểm thử kịch bản vật lý trên đường thử đóng (closed proving ground) theo các tiêu chuẩn quốc tế như Euro NCAP trước khi triển khai phần mềm thương mại.
