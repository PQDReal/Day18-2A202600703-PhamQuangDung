# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Cá nhân  
**Thành viên:** Phạm Quang Dũng → M1, M2, M3, M4, M5

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.0000 | 0.5333 | +0.5333 |
| Answer Relevancy | 0.0000 | 0.5537 | +0.5537 |
| Context Precision | 0.0000 | 0.8750 | +0.8750 |
| Context Recall | 0.0000 | 0.7500 | +0.7500 |

**Nhận xét nhanh:** Production pipeline cải thiện rõ ở retrieval: `context_precision` đạt 0.8750 và `context_recall` đạt 0.7500. Điểm yếu còn lại là generation: nhiều câu trả lời trả về "Không tìm thấy." dù ground truth có trong corpus, làm `faithfulness` và `answer_relevancy` thấp hơn.

## Bottom-5 Failures

### #1
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng. Lương thử việc = 85% x 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness = 0.0000
- **Error Tree:** Output sai → Context có thể thiếu hoặc không đủ rõ về cả bảng lương Junior và quy tắc 85% thử việc → Query cần multi-hop giữa chính sách lương và thử việc → Root cause: retrieval/rerank chưa ghép đúng hai tài liệu cần thiết, prompt cũng quá bảo thủ nên trả "Không tìm thấy."
- **Suggested fix:** Tăng `top_k`, dùng parent context khi child match, thêm query expansion cho "lương thử việc", "Junior", "85%", và yêu cầu LLM tính toán từ nhiều context nếu có đủ dữ kiện.

### #2
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness = 0.0000
- **Error Tree:** Output sai → Context cần đồng thời nghỉ phép v2024 và bảng lương Senior → Query là multi-hop + numeric reasoning → Root cause: pipeline chưa đảm bảo recall đa tài liệu cho câu hỏi ghép chính sách nghỉ phép và lương.
- **Suggested fix:** Multi-query retrieval: tách query thành "ngày phép Senior 9 năm thâm niên" và "lương Senior P3 P4"; sau đó merge contexts trước rerank.

### #3
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected:** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Ngoài ra, mua sắm thiết bị CNTT cần có xác nhận cấu hình kỹ thuật từ phòng CNTT trước khi đề xuất. Cần đính kèm ít nhất 3 báo giá vì trên 10 triệu.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness = 0.0000
- **Error Tree:** Output sai → Context cần cả ngưỡng phê duyệt mua sắm và yêu cầu riêng cho thiết bị CNTT → Query OK nhưng nhiều điều kiện trong một câu → Root cause: retrieval/rerank có thể lấy thiếu một trong các policy con.
- **Suggested fix:** Thêm metadata/category filter cho `procurement`/`IT`, giữ parent section khi child liên quan đến "laptop", "30 triệu", "CNTT", và tăng số context đưa vào LLM cho câu hỏi nhiều điều kiện.

### #4
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Theo chính sách v2024 hiện hành, nhân viên có thâm niên từ 3 năm trở lên được cộng thêm 1 ngày phép cho mỗi 3 năm. Chính sách cũ v2023 yêu cầu 5 năm.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness = 0.0000
- **Error Tree:** Output sai → Context có thể bị nhiễu giữa v2023 và v2024 hoặc thiếu dấu hiệu "hiện hành" → Query OK nhưng cần version awareness → Root cause: pipeline chưa ưu tiên version mới khi có tài liệu superseded.
- **Suggested fix:** Extract metadata `version`, `effective_date`, `superseded`; rerank ưu tiên chính sách hiện hành và prompt yêu cầu nêu rõ nếu có bản cũ/bản mới.

### #5
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness = 0.0000
- **Error Tree:** Output sai → Context cần match cả "thử việc" và "bảo hiểm sức khỏe PVI" → Query OK, dạng negation/eligibility → Root cause: retrieval có thể ưu tiên tài liệu PVI chung, nhưng thiếu đoạn ngoại lệ cho nhân viên thử việc.
- **Suggested fix:** Thêm query expansion cho phủ định/đối tượng: "thử việc", "không được hưởng", "điều kiện tham gia"; dùng reranker mạnh hơn hoặc rule boost cho chunks chứa cả entity `PVI` và `thử việc`.

## Case Study (cho presentation)

**Question chọn phân tích:** Nhân viên được nghỉ bao nhiêu ngày phép năm?

**Observed answer:** Mỗi nhân viên chính thức được hưởng 12 ngày phép năm có lương.

**Expected:** Theo chính sách hiện hành v2024, nhân viên được nghỉ 15 ngày phép năm có lương. Chính sách cũ v2023 là 12 ngày nhưng đã bị thay thế.

**Error Tree walkthrough:**
1. Output đúng? → Không. Output lấy số 12 ngày từ chính sách cũ v2023.
2. Context đúng? → Chưa đủ/chưa đúng thứ tự ưu tiên. RAGAS ghi `context_precision = 0.0` cho case này, cho thấy context retrieved/reranked có nhiễu hoặc ưu tiên sai bản.
3. Query rewrite OK? → Query ngắn và mơ hồ về version; hệ thống phải tự hiểu "hiện hành" nhưng chưa có metadata version/effective-date đủ mạnh.
4. Fix ở bước: M2/M3/M5. Cần metadata versioning trong enrichment, boost tài liệu `v2024`, down-rank tài liệu `v2023/superseded`, và prompt yêu cầu chọn chính sách hiện hành khi nhiều phiên bản mâu thuẫn.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Thêm metadata `version`, `is_current`, `supersedes` trong M5.
- Trong M2/M3, boost chunks có `v2024`, `hiện hành`, `có hiệu lực`; giảm điểm chunks có `v2023`, `cũ`, `thay thế`.
- Tăng `RERANK_TOP_K` hoặc truyền parent chunk thay vì chỉ child chunk cho câu hỏi cần bối cảnh.
- Thêm query decomposition cho câu hỏi multi-hop/numeric.
