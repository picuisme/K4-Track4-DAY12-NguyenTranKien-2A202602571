# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Nguyễn Trần Kiên` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602571` |
| GitHub | `picuisme` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `7325 / 5312 (5 seed: 5312..5316)` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.256 | 20.2 (CI95 18.8–21.7) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.541 | 0.14 (CI95 0.05–0.36) | (baseline) |
| EgoMLP | 0.754 .. 0.761 (TB 0.757 ±0.003) | 18.1 .. 32.9 (TB 25.9) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.615 .. 0.622 (TB 0.618 ±0.003) | 0.35 .. 3.0 (TB 1.2) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.815 ±0.020 | 0.24 .. 0.35 (TB 0.30) | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 cho thấy ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan tới đạo hàm của `Speed_LV`: tương quan chỉ 0.045, trong khi cột tương ứng của xe sau (`Acc_FAV`) đạt 0.929. Vì vậy mình không dùng `Acc_LV` mà dùng gia tốc xe dẫn tính từ tốc độ đã làm mượt (`a_lead`); với gia tốc này p0.1 / p1 là −2.77 / −1.95 m/s², đủ để khai thác ca phanh. (2) Có 383 quỹ đạo trùng hệt nhau nằm ở các id cách xa nhau, nên 9993 quỹ đạo thô còn 9610 sau khi loại trùng. Nếu không loại, một quỹ đạo có thể rơi cả vào train lẫn test, làm điểm test bị "rò" và đẹp giả. (3) Các khung hình liền nhau (10 Hz, mỗi quỹ đạo 70–109 bước) gần như giống hệt nhau, nên chia theo khung thì test chỉ là bản sao gần của train. Vì thế phải chia theo **quỹ đạo** (seed chia 7325) để test thật sự là những lần lái chưa từng thấy.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Log gồm 600 đoạn (9.4 MB). Chạy lại cùng mã: 0/600 đoạn khác, 0 đoạn khác bit. Bản refactor float32 khác bit ở **600/600** đoạn nhưng **0** đoạn vượt dung sai (atol = rtol = 1e-3, sai lệch lớn nhất 9.7e-05 m/s²). Phép cộng số thực không kết hợp, (0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6, nên so khớp từng bit là phép thử sai; dung sai cỡ 1e-3 m/s² lớn hơn nhiều so với nhiễu làm tròn nhưng nhỏ hơn nhiều so với một thay đổi hành vi thật. Thêm AEB chỉ làm lệch **11/600** đoạn (59, 122, 162, …; sai lệch tối đa 1.84 m/s²). Mở đoạn 59 và 122: chỉ vài khung cuối bị lệch, đúng những khung khoảng cách còn vài mét và TTC tụt dưới ngưỡng AEB 1,5 s, nơi lệnh IDM bị ép xuống −6 m/s²; ở mọi khung khác AEB không kích hoạt nên lệnh giống hệt. Phát nhanh nhất có 0 khung trễ, 0 khung rớt vì không có hạn chót; phát theo thời gian thật lộ ra 584 khung trễ + 577 khung rớt trên 59917 khung, làm 380/600 đoạn đổi lệnh (lệch tới 10.6 m/s²) — lỗi thời gian xử lý chỉ thấy được khi tôn trọng thời gian.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của mình: tốc độ cruise 14 m/s, khoảng cách 2.76–59.3 m (p2–p98 của log), tốc độ tiến gần 0–14 m/s, 60 × 45 = 2700 ô. Biên bất khả thi (phanh 9 m/s² sau độ trễ) chiếm **3.6 %** lưới với độ trễ 0 s và **10.2 %** với 0.6 s; bộ phanh-hết-cỡ đâm đúng ở đó, 0 ô lệch ngoài biên ở cả hai độ trễ, nên mô phỏng và công thức `unavoidable` khớp nhau. IDM đâm 3.6 % → 10.2 % khi độ trễ tăng 0 → 0.6 s, tức là **đúng bằng** vùng bất khả thi: diện tích va chạm do lỗi của IDM là 0.0 %. Đó là cách tách "ai cũng đâm" khỏi "bộ điều khiển đâm": const-vel đâm 80.6 % lưới, trong đó 70.4 % (ở 0.6 s) là vùng tránh được. Log không phủ vùng khó: khai thác được 614 ca xe dẫn phanh (6.4 % quỹ đạo), phanh đỉnh p99 chỉ −4.2 m/s², người lái trong log phanh mạnh nhất cỡ 3.6 m/s² (p99.9), trong khi **12.7 %** ô lưới đòi phanh mạnh hơn thế — một mô hình học từ log chưa từng thấy vùng này.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?


| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|
| PercMLP | 0.618 | 1 | 1.2 % | 2 |
| EgoMLP | 0.757 | 2 | 25.9 % | 4 |
| const-vel | 1.256 | 3 | 20.2 % | 3 |
| IDM | 1.541 | 4 | 0.14 % | 1 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất (1.541 m) vì nó không bắt chước người: nó giữ khoảng cách theo luật riêng (T = 1.2 s) nên quỹ đạo lệch khỏi người lái; nhưng trong vòng kín nó **phản hồi** với khoảng cách và tốc độ xe dẫn ở mọi bước, tự sửa sai, nên chỉ đâm 0.14 % (lát cắt phanh 1.6 %). EgoMLP có L2 0.757 m tốt hơn const-vel vì trên log người lái chủ yếu giữ đà, nên đoán theo lịch sử tốc độ của chính mình là đủ; nhưng nó **không thấy** khoảng cách và tốc độ xe dẫn, nên khi xe dẫn phanh nó không biết để phanh (59.9 % va chạm trên lát cắt phanh). Trong vòng kín, sai số của nó còn tích luỹ: trạng thái do chính nó tạo ra khác log, nên điểm vòng hở không đo được điều quan trọng.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động **18.1 % → 32.9 %** va chạm qua 5 seed (5315 tốt nhất, 5314 tệ nhất) dù L2 vòng hở gần như bằng nhau (0.754–0.761 m). Khoảng tin cậy Wilson 95 % của seed tốt nhất là [16.7, 19.5] % và của seed tệ nhất là [31.2, 34.6] % — **không chồng nhau**, nên khác biệt là do seed huấn luyện chứ không phải do may rủi khi lấy mẫu 2883 quỹ đạo. PercMLP cũng vậy: 0.35 % [0.2, 0.6] đến 3.0 % [2.5, 3.7]. Nếu chỉ chạy một seed, có thể báo "PercMLP 0.35 %" — gần IDM 0.14 % — và kết luận gần đủ để phát hành, trong khi seed 5312 cho 3.0 %, hơn 20 lần IDM. CI chỉ đo nhiễu lấy mẫu trong một lần chạy, không đo phương sai giữa các lần huấn luyện, nên phải báo dải qua seed và lấy seed xấu nhất để quyết định (cổng R2 và R5 làm đúng như vậy).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Hiệu chỉnh xe sau IDM trên phân phối khoảng cách thời gian của người lái (trung vị 1.87 s): T tốt nhất là **1.4 s** với khoảng cách Wasserstein 0.349, so với 0.865 của một xe không bao giờ phản ứng — gần gấp 2.5 lần gần hơn với người thật. Bảng ở độ trễ 0.6 s (tỷ lệ lưới có va chạm bất kỳ):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 80.6 % | 80.6 % | 0.0 % / 0.0 % |
| IDM | 80.9 % | 10.2 % | 70.6 % / 0.0 % |
| EgoMLP | 82.8 % | 82.8 % | 0.0 % / 0.0 % |
| PercMLP | 78.4 % | 36.1 % | 43.3 % / 0.0 % |

Với xe sau phát lại, IDM phanh đúng để tránh xe cắt vào, nhưng xe sau vẫn chạy theo tốc độ trong log nên đâm vào đuôi: 70.6 % va chạm phía sau là **va chạm giả**, sinh ra vì thế giới không phản ứng. Kết quả là IDM (80.9 %) trông còn tệ hơn const-vel (80.6 %) — bộ không bao giờ phanh. Khi xe sau phản ứng, va chạm phía sau về 0.0 % và IDM chỉ còn 10.2 %, đúng bằng vùng bất khả thi; kết luận đảo ngược hoàn toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi R2, R3, R4, R5, R6. Hai nguyên nhân gốc: (a) **kịch bản log không chứa** — trên lưới cắt vào ở 0.6 s, 25.9 % diện tích tránh được vẫn đâm (R4, IDM 0.0 %), thế giới phản ứng 36.1 % so với 10.2 % (R6); ở tốc độ cruise 20 m/s chưa từng huấn luyện nó đâm 23.0–37.5 % lưới; (b) **sai số tích luỹ** khi tự lái — trên log nó đâm 0.35–3.0 % qua seed (cận trên seed xấu nhất 3.71 % > 1.14 %, R2; dải 2.7 điểm > 1 điểm, R5), dù L2 vòng hở tốt nhất. Sửa: thêm 197030 cửa sổ từ 20000 kịch bản cắt vào mô phỏng (chữa (a): dạy mô hình vùng log không có, nhãn là IDM) và 2 vòng DAgger (chữa (b): gắn nhãn IDM cho chính các trạng thái mô hình tự đi vào). PercMLP+ qua cả 7 quy tắc: va chạm log 0.24–0.35 % (cận trên 0.64 %), lát cắt phanh 1.6 %, lưới 0.9 %, thế giới phản ứng 11.1 %, ở 20 m/s chỉ 0.1–2.7 %. Nhưng PercMLP+ **không tốt hơn** IDM: mọi con số đều bằng hoặc hơi kém IDM (0.35 % vs 0.14 %, 0.9 % vs 0.0 %, 11.1 % vs 10.2 %); cổng chỉ chứng minh "không kém hơn quá biên". Nó học từ IDM nên khó vượt thầy. Lưu ý L2 vòng hở của PercMLP+ **tăng** 0.618 → 0.815 m vì nó bắt chước IDM thay vì người, trong khi vòng kín tốt hơn hẳn — thêm bằng chứng rằng L2 không phải tiêu chí phát hành. Mình không đề xuất thay IDM ngay: trước hết cần lý do để thay (một lợi ích đo được, ví dụ êm hơn), thử thêm tác nhân phản ứng đa dạng, nhiễu cảm biến và độ trễ thật, rồi chạy bóng (shadow mode) trên xe thật.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Chỉ một chiều dọc, không đổi làn, không cảm biến thật**: mô hình nhận khoảng cách và tốc độ xe dẫn hoàn hảo (thử nhiễu ±1 m / ±0.5 m/s chỉ là một mức), nên 0.24–0.35 % va chạm không nói gì về lỗi nhận thức, che khuất hay xe cắt vào từ bên cạnh — những ca nguy hiểm nhất ngoài đời. (2) **Xe dẫn trong log không phản ứng với xe thử**, và xe sau phản ứng chỉ là một IDM hiệu chỉnh trên một tham số T = 1.4 s; câu 6 cho thấy mô hình tác nhân làm đổi kết luận từ 80.9 % xuống 10.2 %, nên mọi con số phụ thuộc mạnh vào giả định về người khác. (3) **Thế giới mô phỏng là giả định của chính ta**: bộ chấp hành lý tưởng (phanh 9 m/s² tức thời sau độ trễ cố định), không có đường trơn, dốc, tải; vùng bất khả thi 10.2 % đúng trong mô hình này chứ không phải trên xe thật. (4) **Phân phối dữ liệu hẹp**: 9610 quỹ đạo Argoverse 7–11 s, tốc độ p99 ~15.6 m/s, phanh mạnh nhất của người 3.6 m/s²; PercMLP còn đâm 23–37.5 % lưới ở 20 m/s, nên không thể ngoại suy sang cao tốc. Để thu hẹp khoảng cách: mô phỏng nhiều chiều với cảm biến và tác nhân phản ứng đa dạng, thêm dữ liệu cao tốc và phanh gấp, chạy chế độ bóng trên xe thật trước khi cho mô hình điều khiển.
