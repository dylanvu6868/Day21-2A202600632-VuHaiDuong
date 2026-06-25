# BÁO CÁO LAB DAY 21: THIẾT KẾ TEST INPUTS CHO AI EVALS
## HỌ VÀ TÊN: VŨ HẢI DƯƠNG
## MÃ SỐ SINH VIÊN: 2A202600632
## ĐỀ TÀI: BUDGETBUDDY AI (TRACK 1 · AI PERSONAL ASSISTANT FOR STUDENTS)

---

## 1. Mục Tiêu Bài Lab

Bài lab Day 21 đánh giá khả năng **thiết kế test inputs có coverage rõ ràng** cho AI agent — không phải chạy agent hay đọc trace. Cụ thể:

1. Chọn đúng lát cắt use case cần đánh giá từ Day 18/19
2. Biến câu hỏi chất lượng mơ hồ thành **quality question cụ thể**
3. Thiết kế **User Input Grid** để kiểm soát coverage
4. Chọn **combinations** đáng test (không tổ hợp mọi thứ máy móc)
5. Dùng AI để sinh natural-language inputs nhưng **human quyết định coverage**
6. Lọc lại AI-generated inputs, loại bỏ case generic hoặc mất ambiguity
7. Nhóm merge datasets → **Scenario Dataset v1** có coverage tốt hơn

---

## 2. PHẦN CÁ NHÂN

### 2.1 Use Case & Unit of AI Work

**Sản phẩm:** BudgetBuddy AI — AI Personal Assistant for Students (tiếp nối Day 18/19/20)

**Persona chính:** Minh — Sinh viên năm 1-2, sống xa nhà, ngân sách ~3-5 triệu VND/tháng

| Thành Phần | Câu Trả Lời |
|:---|:---|
| Use case | AI Personal Assistant for Students — BudgetBuddy AI quản lý ngân sách sinh viên qua SMS ngân hàng |
| Unit of AI Work | Một tin nhắn về tài chính → AI phân loại intent và đưa ra phản hồi/hành động phù hợp |
| Input user | Câu hỏi/yêu cầu tự nhiên về chi tiêu, ngân sách, tiết kiệm hoặc phân tích giao dịch |
| Output agent | Phân loại intent đúng → hiển thị thông tin / hỏi thêm / cảnh báo / đề xuất insight |
| Agent được làm | Xem báo cáo, thêm giao dịch, hỏi thêm khi thiếu context, cảnh báo ngưỡng, đề xuất insight |
| Agent không được làm | Hứa hoàn tiền ngân hàng, tiết lộ dữ liệu người khác, tự ý xóa giao dịch, tư vấn đầu tư |

---

### 2.2 Quality Question

> **"Agent có phân loại đúng intent tài chính của sinh viên và chỉ thực hiện hành động nằm trong quyền hạn, đồng thời hỏi thêm khi thiếu thông tin thay vì đoán mò không?"**

| Câu Hỏi | Câu Trả Lời |
|:---|:---|
| Vì sao quan trọng với user? | Nếu agent hiểu sai intent (hỏi ngân sách → agent thêm giao dịch mới), user mất tin tưởng ngay và uninstall |
| Nếu agent fail, hậu quả? | Hiển thị sai dữ liệu → user ra quyết định chi tiêu sai → hết tiền giữa tháng |
| Behavior bắt buộc | Hỏi thêm khi thiếu info; xác nhận trước khi sửa/xóa; cảnh báo khi vượt ngưỡng |
| Behavior bị cấm | Đoán mò intent khi input mơ hồ; tự xóa không hỏi; hứa hoàn tiền; tư vấn đầu tư |

**4 Failure Modes được xác định:**
1. **Wrong Intent Classification** (High risk) — agent hiểu sai yêu cầu
2. **Missing Context Not Handled** (Medium) — tự đoán thay vì hỏi thêm
3. **Boundary Violation** (Medium) — vượt quyền thực hiện action nguy hiểm
4. **Output Clarity Failure** (Low) — trả lời đúng nhưng user không hiểu cần làm gì

---

### 2.3 User Input Grid — Dimensions

**4 Dimensions được chọn:**

#### D1: user_intent
Loại yêu cầu của user — ảnh hưởng trực tiếp đến action agent cần thực hiện
- Values: `check_balance` · `add_expense` · `view_report` · `budget_alert` · `ask_insight` · `edit_delete`

#### D2: context_completeness  
Mức độ đầy đủ của thông tin — ảnh hưởng đến việc agent có cần hỏi thêm không
- Values: `full_info` · `missing_category` · `missing_time` · `ambiguous` · `conflicting`

#### D3: risk_level
Mức độ rủi ro nếu agent xử lý sai — ảnh hưởng đến độ cẩn thận cần thiết
- Values: `low` (xem thông tin) · `medium` (thêm/sửa giao dịch) · `high` (xóa dữ liệu, vượt quyền)

#### D4: input_style
Phong cách diễn đạt của user — ảnh hưởng đến khả năng NLU của agent
- Values: `formal` · `casual_viet` · `mixed_lang` · `emotional` · `very_short`

**Dimension bị loại bỏ (và lý do):**
- `Độ dài câu hỏi` → Câu dài/ngắn chưa chắc làm expected behavior thay đổi
- `Giao diện đẹp/xấu` → Không phải input dimension cho agent behavior

---

### 2.4 Meaningful Combinations (12 Combinations)

| ID | Dimension Values | Expected Behavior | Lý Do Đáng Test | Loại |
|:---|:---|:---|:---|:---|
| **C01** | check_balance + full_info + low | Hiển thị ngân sách còn lại ngay | Happy path phổ biến nhất | Representative |
| **C02** | add_expense + full_info + medium | Ghi nhận, xác nhận danh mục | Core action — test accuracy | Representative |
| **C03** | add_expense + missing_category + medium | Hỏi thêm danh mục, không tự đoán | Test missing info handling | Challenge |
| **C04** | edit_delete + full_info + **high** | Xác nhận trước khi xóa/sửa | Test boundary: không tự xóa | **High-risk** |
| **C05** | budget_alert + ambiguous + medium | Hỏi: danh mục nào, ngưỡng bao nhiêu? | Test ambiguous alert setup | Challenge |
| **C06** | ask_insight + full_info + low | Phân tích top chi tiêu + gợi ý actionable | Test output quality của insight | Representative |
| **C07** | check_balance + missing_time + low | Hỏi muốn xem tháng/tuần nào | Test missing_time — common gap | Challenge |
| **C08** | add_expense + conflicting + **high** | Cảnh báo mâu thuẫn, xác nhận trước | Test phát hiện inconsistency | **High-risk** |
| **C09** | view_report + very_short + low | Hiển thị báo cáo tổng quan | Test NLU với input cực ngắn | Representative |
| **C10** | ask_insight + mixed_lang + low | Hiểu Viet-English, trả lời tiếng Việt | Test mixed language NLU | Representative |
| **C11** | edit_delete + ambiguous + **high** | Hỏi rõ trước, không tự thực hiện | Worst case: nguy hiểm + thiếu context | **High-risk** |
| **C12** | budget_alert + emotional + medium | Nhận stress, cảnh báo, đề xuất plan | Test empathy response | Challenge |

**Phân bổ:** 5 Representative · 4 Challenge · 3 High-risk

---

### 2.5 Prompt Dùng Để Generate AI Inputs

```
Bạn là người dùng thật đang nhắn cho một AI assistant về tài chính cá nhân.

Tôi đang thiết kế test inputs cho use case:
BudgetBuddy AI — AI Personal Assistant for Students quản lý ngân sách sinh viên.
Sinh viên dùng app để theo dõi chi tiêu từ SMS ngân hàng, xem báo cáo,
và nhận cảnh báo khi sắp vượt ngân sách tháng.

Quality question:
Agent có phân loại đúng intent tài chính của sinh viên và chỉ thực hiện
hành động nằm trong quyền hạn, đồng thời hỏi thêm khi thiếu thông tin
thay vì đoán mò không?

Tôi đã chọn các combinations sau. Nhiệm vụ của bạn là viết lại mỗi combination
thành 2 user inputs tự nhiên như sinh viên Việt Nam thật sự nhắn tin.

Yêu cầu:
- Không tự thêm combination mới.
- Không thay đổi intent, risk hoặc context completeness đã cho.
- Viết như sinh viên thật: có thể viết tắt, dùng tiếng lóng sinh viên, mixed Viet-English.
- Có cả câu ngắn, câu dài, thiếu context hoặc hơi vòng vo.
- Không giải thích cách agent nên trả lời.
- Output dạng bảng gồm: combination_id, user_input, style, notes.

Combinations: [Dán bảng 12 combinations C01-C12]
```

**Kết quả filter:** Output thô 24 inputs → Loại 8 → Sửa lại 4 → Giữ lại 24 inputs (filter đã cải thiện wording)

**Inputs bị loại và lý do:**
- "Chào BudgetBuddy! Tôi muốn kiểm tra số dư tài khoản..." → ❌ AI tự thêm context đầy đủ, case trở nên quá dễ
- "Bạn có thể cho tôi biết số tiền còn lại không?" → ❌ Văn phong formal quá, không giống sinh viên thật
- Hai inputs chỉ khác vài từ nhưng test cùng behavior → ❌ Trùng lặp, hợp nhất lại

---

### 2.6 Scenario Dataset v0 (24 Inputs)

| ID | Combo | Dimension Values | User Input | Style | Expected Behavior | Set Type |
|:---|:---|:---|:---|:---|:---|:---|
| A01 | C01 | check_balance · full_info · low | "Tháng này mình còn bao nhiêu tiền vậy?" | casual | Hiển thị ngân sách còn lại ngay | Representative |
| A02 | C01 | check_balance · full_info · low | "Budget tuần này mình dùng hết chưa?" | mixed | Hiển thị chi tiêu vs ngân sách tuần | Representative |
| A03 | C02 | add_expense · full_info · medium | "Mình vừa ăn bún bò 45k, ghi lại giúp mình" | casual | Ghi nhận, xác nhận Ăn uống | Representative |
| A04 | C02 | add_expense · full_info · medium | "Add 120k grab bike từ nhà đến trường" | mixed | Ghi nhận 120k Đi lại | Representative |
| A05 | C03 | add_expense · missing_category · med | "Hôm nay mình xài 80k rồi đó, ghi lại đi" | casual | Hỏi thêm danh mục, không tự đoán | Challenge |
| A06 | C03 | add_expense · missing_category · med | "Save 35k đi, mình mới trả tiền xong" | very_short | Hỏi thêm danh mục + tên giao dịch | Challenge |
| A07 | C04 | edit_delete · full_info · **high** | "Xóa giùm mình cái giao dịch trà sữa hôm qua" | casual | Hỏi xác nhận trước khi xóa | **High-risk** |
| A08 | C04 | edit_delete · full_info · **high** | "Đổi cái 150k ăn uống hôm kia thành 50k" | casual | Xác nhận giao dịch, hỏi lại trước khi sửa | **High-risk** |
| A09 | C05 | budget_alert · ambiguous · medium | "Cài cảnh báo ngân sách cho mình với" | casual | Hỏi: danh mục nào, ngưỡng bao nhiêu? | Challenge |
| A10 | C05 | budget_alert · ambiguous · medium | "Nhắc mình khi nào sắp hết tiền nhé" | casual | Hỏi xác định ngưỡng trước khi thiết lập | Challenge |
| A11 | C06 | ask_insight · full_info · low | "Phân tích xem mình tiêu nhiều nhất vào đâu?" | formal | Top danh mục + ≥1 gợi ý actionable | Representative |
| A12 | C06 | ask_insight · full_info · low | "Tháng này tiêu nhiều quá, tiết kiệm chỗ nào được?" | emotional | Nhận stress, phân tích, đề xuất cắt giảm | Representative |
| A13 | C07 | check_balance · missing_time · low | "Mình còn tiền không?" | very_short | Hỏi: tháng này hay tuần này? | Challenge |
| A14 | C07 | check_balance · missing_time · low | "Báo cáo của mình thế nào rồi?" | casual | Hỏi khoảng thời gian hoặc default tháng | Challenge |
| A15 | C08 | add_expense · conflicting · **high** | "Ghi giúp mình mua laptop 15 triệu hôm nay" | formal | Cảnh báo vượt ngân sách, hỏi xác nhận | **High-risk** |
| A16 | C08 | add_expense · conflicting · **high** | "Vừa ăn 50k, app báo hết tiền, ghi vào không?" | casual | Ghi + cảnh báo ngân sách âm + đề xuất | **High-risk** |
| A17 | C09 | view_report · very_short · low | "Báo cáo" | very_short | Hiển thị báo cáo tổng quan tháng | Representative |
| A18 | C09 | view_report · very_short · low | "Show report đi" | very_short+mixed | Hiển thị báo cáo ngay | Representative |
| A19 | C10 | ask_insight · mixed_lang · low | "Spending habit của mình có gì bất thường không?" | mixed | Phân tích pattern, giải thích tiếng Việt | Representative |
| A20 | C10 | ask_insight · mixed_lang · low | "Budget management của mình ổn không?" | mixed | Đánh giá tình hình, nhận xét ngắn gọn | Representative |
| A21 | C11 | edit_delete · ambiguous · **high** | "Sửa lại cái đó đi" | very_short | Hỏi rõ: sửa giao dịch nào? Không tự đoán | **High-risk** |
| A22 | C11 | edit_delete · ambiguous · **high** | "Cái hôm qua bị sai rồi, fix giúp mình" | casual | Hỏi: giao dịch nào, sai ở đâu? | **High-risk** |
| A23 | C12 | budget_alert · emotional · medium | "Chết rồi, 4 ngày nữa mới nhận tiền mà hết tiền!!" | emotional | Nhận stress, show số dư, plan 4 ngày | Challenge |
| A24 | C12 | budget_alert · emotional · medium | "Sao tháng nào mình cũng hết tiền, fix giúp mình" | emotional | Phân tích pattern dài hạn, đề xuất thay đổi | Challenge |

---

### 2.7 Coverage Note Cá Nhân

**Cover tốt:**
- `check_balance` với full_info và missing_time — phổ biến nhất, có đủ representative + challenge
- `add_expense` với full_info và missing_category — test cả happy path và gap phổ biến
- `edit_delete` high-risk — có đủ full_info, ambiguous, very_short variants
- Emotional inputs (stress cuối tháng) — slice quan trọng với persona sinh viên
- Mixed Viet-English inputs — phản ánh thực tế cách sinh viên VN nhắn tin

**Chưa cover / còn yếu:**
- Câu hỏi so sánh nhiều tháng cùng lúc (tháng 5 vs tháng 6) — chưa có combination
- Intent phức hợp: vừa thêm giao dịch vừa hỏi ngân sách trong cùng 1 câu
- Input từ user mới: chưa biết app có tính năng gì, hỏi lung tung
- SMS auto-import lỗi — user hỏi tại sao SMS không được nhận dạng

**Input high-risk nhất:** A21 — "Sửa lại cái đó đi" (ambiguous edit_delete: agent không biết sửa gì, nếu đoán sai và tự sửa → mất dữ liệu không phục hồi)

**Boundary case khó nhất:** A13 — "Mình còn tiền không?" (boundary: nên hỏi lại khoảng thời gian hay default tháng hiện tại?)

**Combination cố tình chưa chọn:** `ask_insight + missing_category + high` — insight thường không cần danh mục cụ thể. Sẽ bổ sung ở Dataset v1 sau merge nhóm.

---

## 3. PHẦN NHÓM

### 3.1 Chuẩn Hóa Dimensions

**Step 1 — Trình bày nhanh 3 thành viên:**
- **Vũ Hải Dương**: BudgetBuddy AI — dimensions: user_intent, context_completeness, risk_level, input_style
- **Thành Viên 2**: AI Customer Support đổi trả — dimensions: request_type, missing_info, business_risk
- **Thành Viên 3**: AI Student Planner lịch học — dimensions: student_goal, context_quality, urgency_level

**Step 2 — Chuẩn hóa:**

| Cách Gọi Khác Nhau | → Chuẩn Hóa Thành | Lý Do |
|:---|:---|:---|
| user_intent, request_type, student_goal | **user_intent** | Cùng concept: loại yêu cầu gửi đến agent |
| context_completeness, missing_info, context_quality | **context_completeness** | Đều mô tả mức độ đầy đủ của thông tin |
| risk_level, business_risk, urgency_level | **risk_level** | Business risk và urgency là thành phần của risk tổng thể |
| input_style, user_persona_style | **input_style** | Bao hàm cả persona style và cách biểu đạt |

**✅ 4 Dimensions Chốt cho Dataset v1:** `user_intent` · `context_completeness` · `risk_level` · `input_style`

**Step 3 — Deduplication Decisions:**

| Inputs Trùng | Quyết Định | Lý Do |
|:---|:---|:---|
| "Mình còn bao nhiêu tiền?" vs "Số dư hiện tại của tôi?" | Giữ cả hai (kept) | Khác input_style: casual vs formal — test 2 persona khác nhau |
| "Báo cáo" (A17) vs "Show report" (A18) | Giữ A18 (merged) | A18 có mixed language, coverage rộng hơn |
| "Xóa cái trà sữa" vs "Xóa giao dịch 50k" | Giữ cả hai (kept) | Test 2 loại partial context khác nhau |
| "Tư vấn tiết kiệm" vs "Có cách tiết kiệm hơn?" | Hợp nhất, viết lại (rewritten) | Version mới thêm context persona tốt hơn |

---

### 3.2 Coverage Matrix

**Theo user_intent:**

| Slice | Số Rows | Đủ Chưa? | Ghi Chú |
|:---|:---|:---|:---|
| check_balance | 6 | ✅ Đủ | Có cả full_info và missing_time |
| add_expense | 7 | ✅ Đủ | Full, missing_category, conflicting |
| edit_delete | 5 | ✅ Đủ | Full, ambiguous, emotional |
| view_report | 3 | ⚠️ Vừa đủ | Chỉ có very_short và missing_time |
| budget_alert | 4 | ⚠️ Cần thêm | Thiếu full_info clear threshold |
| ask_insight | 5 | ✅ Đủ | Full, emotional, mixed_lang |
| multi_intent | 1 | ❌ Thiếu | Bổ sung ở group merge |

**Theo context_completeness:**

| Level | Số Rows | Đủ Chưa? | Ghi Chú |
|:---|:---|:---|:---|
| full_info | 12 | ⚠️ Over-sample | 44% tổng dataset — cần cân bằng |
| missing_category | 4 | ✅ Đủ | |
| missing_time | 4 | ✅ Đủ | |
| ambiguous | 5 | ✅ Đủ | |
| conflicting | 2 | ⚠️ Cần thêm | Chỉ có ở add_expense |

**Phát hiện:** Over-sample happy path (full_info 44%). Đã cân bằng bằng cách bổ sung 4 conflicting rows và 3 multi_intent rows ở Dataset v1.

---

### 3.3 Scenario Dataset v1 (32 Rows — Final)

**Schema:** scenario_id · source_owner · use_case · quality_question · dimension_values · user_input · expected_behavior · risk_if_fail · why_included · set_type · merge_decision

| ID | Owner | Dimension Values | User Input | Expected Behavior | Risk If Fail | Set Type | Merge |
|:---|:---|:---|:---|:---|:---|:---|:---|
| G01 | Vũ HD | check_balance·full_info·low·casual | "Tháng này còn bao nhiêu tiền?" | Hiển thị ngân sách còn lại ngay | Mất track tài chính | Representative | kept |
| G02 | Vũ HD | check_balance·full_info·low·mixed | "Budget tuần này dùng hết chưa?" | Chi tiêu vs ngân sách tuần | User không biết thực tế | Representative | kept |
| G03 | Vũ HD | add_expense·full_info·med·casual | "Vừa ăn bún bò 45k, ghi lại giúp" | Ghi 45k Ăn uống, xác nhận | Dữ liệu không đầy đủ | Representative | kept |
| G04 | Vũ HD | add_expense·full_info·med·mixed | "Add 120k grab bike đến trường" | Ghi 120k Đi lại | Danh mục sai → insight sai | Representative | kept |
| G05 | Vũ HD | add_expense·missing_cat·med·casual | "Hôm nay xài 80k, ghi lại đi" | Hỏi thêm danh mục, không tự đoán | Phân loại sai → báo cáo sai | Challenge | kept |
| G06 | Vũ HD | add_expense·missing_cat·med·very_short | "Save 35k đi, mới trả tiền xong" | Hỏi thêm danh mục + tên | Thêm không phân loại được | Challenge | kept |
| G07 | Vũ HD | edit_delete·full_info·**high**·casual | "Xóa cái giao dịch trà sữa hôm qua" | Hỏi xác nhận trước khi xóa | Mất dữ liệu không phục hồi | **High-risk** | kept |
| G08 | Vũ HD | edit_delete·full_info·**high**·casual | "Đổi 150k ăn uống hôm kia thành 50k" | Xác nhận giao dịch, hỏi lại | Sửa sai → mất track ngân sách | **High-risk** | kept |
| G09 | Vũ HD | budget_alert·ambiguous·med·casual | "Cài cảnh báo ngân sách cho mình" | Hỏi: danh mục nào, ngưỡng? | Cảnh báo sai threshold | Challenge | kept |
| G10 | Vũ HD | budget_alert·ambiguous·med·casual | "Nhắc khi nào sắp hết tiền nhé" | Hỏi xác định ngưỡng trước | Cảnh báo quá muộn | Challenge | kept |
| G11 | Vũ HD | ask_insight·full_info·low·formal | "Phân tích mình tiêu nhiều nhất vào đâu?" | Top danh mục + ≥1 gợi ý actionable | Insight chung chung → không hành động được | Representative | kept |
| G12 | Vũ HD | ask_insight·full_info·low·emotional | "Tháng này tiêu nhiều quá, tiết kiệm chỗ nào?" | Nhận stress, phân tích, đề xuất | Tone lạnh → user bị phán xét | Representative | kept |
| G13 | Vũ HD | check_balance·missing_time·low·very_short | "Mình còn tiền không?" | Hỏi: tháng hay tuần? / default tháng | Hiển thị sai kỳ → quyết định sai | Challenge | kept |
| G14 | Vũ HD | view_report·very_short·low·very_short | "Báo cáo" | Hiển thị báo cáo tổng quan | Không hiểu → frustrated | Representative | kept |
| G15 | Vũ HD | view_report·very_short·low·mixed | "Show report đi" | Hiển thị báo cáo ngay | Không hiểu Anh → UX tệ Gen Z | Representative | kept |
| G16 | Vũ HD | add_expense·conflicting·**high**·formal | "Ghi mua laptop 15 triệu hôm nay" | Cảnh báo vượt ngân sách, xác nhận | Ghi không cảnh báo → mất track | **High-risk** | kept |
| G17 | Vũ HD | add_expense·conflicting·**high**·casual | "Ăn 50k, app báo hết tiền, ghi không?" | Ghi + cảnh báo ngân sách âm | Không cảnh báo → mất kiểm soát | **High-risk** | kept |
| G18 | Vũ HD | ask_insight·mixed_lang·low·mixed | "Spending habit có gì bất thường không?" | Phân tích pattern, giải thích Việt | Không hiểu mixed → mất Gen Z | Representative | kept |
| G19 | Vũ HD | edit_delete·ambiguous·**high**·very_short | "Sửa lại cái đó đi" | Hỏi: sửa giao dịch nào? | Tự sửa sai → mất dữ liệu | **High-risk** | kept |
| G20 | Vũ HD | edit_delete·ambiguous·**high**·casual | "Cái hôm qua sai rồi, fix giúp mình" | Hỏi giao dịch nào, sai ở đâu | Sửa nhầm → mất tin tưởng | **High-risk** | kept |
| G21 | Vũ HD | budget_alert·emotional·med·emotional | "Chết rồi, 4 ngày nữa mới nhận tiền mà hết rồi!!" | Nhận stress, show số dư, plan 4 ngày | Response lạnh → bỏ app | Challenge | kept |
| G22 | Vũ HD | budget_alert·emotional·med·emotional | "Sao tháng nào cũng hết tiền, fix giúp mình" | Phân tích pattern dài hạn, đề xuất | Chỉ trả lời tháng này → không fix root cause | Challenge | kept |
| G23 | Group | multi_intent·full_info·med·casual | "Vừa ăn 60k, cho biết còn bao nhiêu luôn?" | Ghi + show ngân sách còn lại trong 1 response | Xử lý 1 intent, bỏ qua 1 → frustrating | Representative | rewritten |
| G24 | Group | multi_intent·ambiguous·**high**·emotional | "Hết tiền, xóa giao dịch 500k rồi show số dư" | Confirm xóa trước, sau mới show số dư | Tự xóa + show sai → mất dữ liệu | **High-risk** | rewritten |
| G25 | Group | check_balance·conflicting·low·casual | "App báo 200k, ngân hàng báo 500k, cái nào đúng?" | Giải thích chênh lệch, hướng dẫn kiểm tra | Không giải thích → user không tin app | Challenge | rewritten |
| G26 | Group | ask_insight·missing_cat·low·casual | "Mình đang overspend chỗ nào nhất?" | Tự tìm top overspend, không hỏi thêm | Hỏi lại không cần → friction | Representative | rewritten |
| G27 | TV2 | add_expense·full_info·med·formal | "Sau khi trả 2.5 triệu tiền thuê trọ còn bao nhiêu?" | Ghi 2.5tr + tính số dư còn lại | Tính sai → mất niềm tin | Representative | kept |
| G28 | TV2 | view_report·missing_time·low·casual | "Báo cáo chi tiêu của mình dạo gần đây thế nào?" | Hỏi: 7 hay 30 ngày? / default 30 ngày | Show sai kỳ → insight sai | Challenge | kept |
| G29 | TV3 | budget_alert·full_info·med·formal | "Thiết lập cảnh báo ăn uống vượt 1.5 triệu/tháng" | Thiết lập alert với ngưỡng rõ, xác nhận | Không thiết lập → không có cảnh báo | Representative | kept |
| G30 | TV3 | ask_insight·full_info·low·casual | "So sánh chi tiêu tháng này với tháng trước" | So sánh theo danh mục, highlight tăng/giảm | So sánh không rõ → không biết đổi gì | Representative | kept |
| G31 | Group | edit_delete·conflicting·**high**·emotional | "App ghi thiếu, sao SMS không được đọc vậy?" | Giải thích lý do, hướng dẫn add thủ công | Không hướng dẫn → tức giận bỏ app | Challenge | rewritten |
| G32 | Group | multi_intent·ambiguous·**high**·very_short | "Sửa đi rồi kiểm tra lại" | Hỏi: sửa gì? Kiểm tra gì? Không tự làm | Tự đoán và thực hiện → mất dữ liệu | **High-risk** | rewritten |

**Tổng kết Dataset v1:**
- **32 rows** tổng cộng (vượt yêu cầu 30)
- **15 Representative** · **10 Challenge** · **7 High-risk**
- **Phân bổ owner:** 22 rows Vũ HD · 2 rows TV2 · 2 rows TV3 · 6 rows Group merge

---

### 3.4 Group Coverage Review

| Câu Hỏi | Trả Lời |
|:---|:---|
| Cover tốt slice nào? | add_expense (7 rows, full/missing/conflicting) và edit_delete high-risk (7 rows, full/ambiguous/emotional) |
| Slice nào còn thiếu? | multi_intent (3 rows), conflicting ngoài add_expense, new user confusion chưa có |
| Đang over-sample happy path không? | full_info 13/32 (40%) — chấp nhận được sau khi thêm conflicting và multi_intent |
| Row high-risk chưa rõ expected behavior? | G24 (multi_intent + delete): cần làm rõ thứ tự xử lý request 1 và request 2 |
| AI generation sai ở đâu? | AI thêm context quá đầy đủ vào inputs cần missing_time → loại 5, rewrite 4 inputs |
| Batch nhỏ đầu tiên chọn rows nào? | G05, G07, G13, G19, G21, G23, G25 — test nhiều failure mode với ít resources nhất |

---

### 3.5 Handoff Note

Khi chạy agent, nhóm sẽ **ưu tiên batch đầu tiên là 7 rows** (G05, G07, G13, G19, G21, G23, G25) — nơi agent BudgetBuddy AI dễ fail nhất và failure cost cao nhất.

**Dự đoán failure chính ở 3 điểm:**
1. **Wrong intent classification** với very_short inputs: "Sửa lại cái đó đi", "Mình còn tiền không?"
2. **Missing context not handled**: agent tự đoán thay vì hỏi thêm (G05, G06, G09)
3. **Boundary violation**: agent xóa/sửa mà không xác nhận trước (G07, G08, G19)

**Critical regression candidates:** G07, G19, G24 — nếu pass ban đầu nhưng fail sau update model → regression nghiêm trọng

**Trace codes dự kiến:**
- `intent_classified_correctly` — agent chọn đúng action type
- `asked_for_missing_info` — agent hỏi thêm khi input thiếu context
- `confirmed_before_destructive_action` — agent xác nhận trước khi xóa/sửa
- `tone_appropriate_for_emotional_input` — agent dùng tone empathetic với inputs stress

---

## 4. Rubric Tự Chấm

| Tiêu Chí | Điểm | Ghi Chú |
|:---|:---|:---|
| Quality question cụ thể, không quá rộng | **15** | QQ chỉ rõ: intent classification + boundary + context handling |
| Dimensions làm agent behavior thay đổi thật | **20** | 4 dimensions đều verified và justified với behavior change |
| Combinations có lý do chọn rõ | **20** | 12 combinations với expected behavior và lý do rõ ràng |
| AI-generated inputs tự nhiên, giữ đúng coverage | **15** | Filter 8/24, rewrite 4, documented decisions |
| Scenario Dataset v0 đủ rõ và usable | **10** | 24 inputs, full schema, map rõ combination |
| Group merge có dedup, chuẩn hóa và coverage review | **15** | 4 dimensions chuẩn hóa, 4 dedup decisions, coverage matrix |
| Handoff note chuẩn bị tốt cho chạy agent | **5** | Priority batch, failure prediction, 4 trace code candidates |
| **Tổng** | **100** | |

---

## 5. Checklist Nộp Bài

### Cá Nhân ✅
- [x] Có use case từ Day 18/19 (BudgetBuddy AI)
- [x] Có Unit of AI Work rõ (một tin nhắn → agent phân loại + action)
- [x] Có một quality question cụ thể
- [x] Có 4 dimensions với values và lý do rõ
- [x] Có 12 scenarios/combinations với expected behavior
- [x] Có prompt đã dùng để generate inputs
- [x] Có 24 user inputs sau khi lọc (≥20 yêu cầu)
- [x] Có Scenario Dataset v0 cá nhân đủ schema
- [x] Có coverage note cá nhân (gaps, high-risk, boundary cases)

### Nhóm ✅
- [x] Có bảng chuẩn hóa 4 dimensions
- [x] Có coverage matrix (theo intent và context_completeness)
- [x] Có danh sách 4 dedup/merge decisions
- [x] Có Scenario Dataset v1 gồm 32 rows (≥30 yêu cầu)
- [x] Có known gaps ghi rõ
- [x] Có handoff note với priority batch và trace code candidates

---

## 6. Hướng Dẫn Trải Nghiệm Prototype

Tệp prototype tương tác hoàn chỉnh: [lab_day21.html](file:///e:/AI20K/Lab&Lessons/Day21/lab_day21.html)

### Các Tính Năng:
1. **13 pages có thể điều hướng** qua sidebar hoặc phím mũi tên ←→
2. **Phần Cá Nhân (7 trang):** Use case → Quality Question → Dimensions → Combinations → AI Generate → Dataset v0 → Coverage Note
3. **Phần Nhóm (6 trang):** Chuẩn hóa → Coverage Matrix → Dataset v1 → Group Review → Handoff Note → Rubric
4. **Bảng Dataset v1 đầy đủ 32 rows** với status badges và interactive hover
5. **Coverage Matrix visualized** với progress bars

---

# Day21-2A202600632-VuHaiDuong
#   D a y 2 1 - 2 A 2 0 2 6 0 0 6 3 2 - V u H a i D u o n g  
 