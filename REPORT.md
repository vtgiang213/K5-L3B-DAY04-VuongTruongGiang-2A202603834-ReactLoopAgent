# Báo cáo phân tích và cải tiến agent

Báo cáo tập trung vào ba case H1, H6 và H8. Evidence được đối chiếu trong `eval_before.json` và `eval_after.json`.

## 1. Case được chọn và evidence lỗi

**H1 - Nhiều bước và dữ liệu dài: agent dừng trước khi đọc nguồn liên quan.** Tool trả liên kết tới bản tổng kết GitHub tại [eval_before.json, dòng 2981](eval_before.json#L2981), nhưng agent kết thúc ở step 2 mà chưa gọi đọc bản tổng kết, theo [trace tại dòng 3047](eval_before.json#L3047). Câu trả lời vì vậy thiếu bằng chứng về nguyên nhân sự cố GitHub.

**H6 - Nhiều bước và dữ liệu dài: agent dừng sớm khi status có dữ liệu nhiễu.** Update của incident ban đầu dẫn tới bản tổng kết tại [dòng 3913](eval_before.json#L3913), nhưng agent vẫn kết thúc mà không đọc tiếp, theo [trace tại dòng 3934](eval_before.json#L3934).

**H8 - Yêu cầu ngoài phạm vi: agent tự tạo URL nguồn.** [Trace từ dòng 4196](eval_before.json#L4196) chỉ có hai lần gọi `get_status`, trả URL trang chủ GitHub và Cloudflare. Tuy nhiên, [câu trả lời tại dòng 4188](eval_before.json#L4188) tự ghép các URL incident dạng `/incident/<id>` mà tool chưa trả về.

## 2. Giả thuyết nguyên nhân

- **H1/H6:** vòng lặp chưa đối chiếu nguồn được tham chiếu với nguồn đã thử đọc trước khi nhận câu trả lời cuối. Một giả thuyết khác là câu chống injection trong system prompt có thể khiến model hiểu nhầm rằng không nên theo tham chiếu trong dữ liệu ngoài; trace không đủ để xác nhận nguyên nhân này.
- **H8:** model suy ra URL từ tên miền và ID. Bản dựng lại ban đầu kiểm tra có URL incident khi đọc chi tiết thành công, nhưng chưa kiểm tra link tự tạo trong final.

## 3. Những thay đổi đã thực hiện trong `agent.py`

### 3.1. Sửa cho H1 và H6

**Theo dõi và kiểm tra incident liên quan.** [Đoạn từ dòng 153](agent.py#L153) lưu incident đã thử đọc và tìm các liên kết incident được tham chiếu trong update của tool. Trước khi chấp nhận final, [đoạn từ dòng 215](agent.py#L215) so sánh hai tập này; nếu còn nguồn chưa thử đọc và còn step, agent ghi `validation_completion_review` rồi nhắc model đối chiếu yêu cầu với bằng chứng. Model vẫn tự quyết định có gọi tool tiếp hay không.

**Tổng hợp câu trả lời và báo giới hạn.** [Đoạn từ dòng 110](agent.py#L110) loại bản trả lời nháp khỏi context lượt sau, giữ tool calls và tool results. [Đoạn từ dòng 194](agent.py#L194) yêu cầu tổng hợp lại câu trả lời; [hàm `format_evidence_limits` tại dòng 23](agent.py#L23) và [đoạn tại dòng 292](agent.py#L292) thêm số cập nhật nhận được và thông tin bị cắt vào final. [Hàm `prepare_observation` tại dòng 43](agent.py#L43) bỏ mục nhiễu khỏi dữ liệu gửi model nhưng giữ tool result gốc trong trace.

### 3.2. Sửa cho H8

**Kiểm tra URL.** [Hàm `extract_urls` tại dòng 15](agent.py#L15) và [đoạn từ dòng 145](agent.py#L145) lưu URL đã quan sát trong kết quả tool. [Đoạn từ dòng 244](agent.py#L244) phát hiện URL chưa quan sát trong final, ghi `validation_unobserved_source` và yêu cầu model sửa trong `max_steps`.

## 4. Điểm mới và đánh giá

**H1: 60 → 75:** agent đã gọi đọc incident gộp trước khi trả lời, theo [trace sau sửa từ dòng 2829](eval_after.json#L2829); lỗi dừng sớm không còn trong lần chạy này.

**H6: 60 → 80:** agent đã đọc tiếp incident gộp và thêm giới hạn dữ liệu, theo [trace sau sửa từ dòng 3859](eval_after.json#L3859), nhưng [final](eval_after.json#L3579) vẫn diễn đạt biện pháp rollback chưa chính xác.

**H8: 75 → 75:** [final sau sửa](eval_after.json#L4207) không còn URL incident tự ghép, nhưng trace không có event `validation_unobserved_source`, nên chưa thể kết luận kiểm tra URL đã trực tiếp sửa lỗi trong lần chạy này.

## 5. So sánh chi phí hai lần eval

| Chỉ số | Before | After |
|---|---:|---:|
| Số lần gọi model agent | 51 | 54 |
| Số lần gọi model judge | 22 | 22 |
| Token input agent, trong đó cached | 34.127 (1.152 cached) | 47.171 (8.832 cached) |
| Token output agent | 10.348 | 11.003 |
| Token input judge, trong đó cached | 39.187 (0 cached) | 39.146 (0 cached) |
| Token output judge | 3.074 | 3.006 |
| **Tổng token agent và judge** | **86.736** | **100.326** |
| Chi phí agent ước tính | 0,01124145 USD | 0,01301505 USD |
| Chi phí judge ước tính | 0,00772245 USD | 0,00767550 USD |
| **Tổng chi phí ước tính** | **0,01896390 USD** | **0,02069055 USD** |

**Cách tính:** theo [giá GPT-4o mini của OpenAI](https://developers.openai.com/api/docs/models/gpt-4o-mini), input là 0,15 USD, cached input là 0,075 USD và output là 0,60 USD cho mỗi triệu token. 

Lần after dùng thêm **13.590 token** và chi phí ước tính tăng **0,00172665 USD** (khoảng **9,1%**). 
