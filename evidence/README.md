# Báo Cáo Phân Tích Thực Nghiệm: Prompt Versioning & Đánh Giá RAGAS (Day 22)

**Học viên:** Nguyễn Nguyên Phong  
**MSSV:** 2A202602691  
**Project:** LLMOps - LangSmith Tracing, Prompt Hub & RAGAS Evaluation  

---

## 1. Giới thiệu & Mục tiêu

Lab Day 22 tập trung vào việc áp dụng các kỹ thuật LLMOps hiện đại nhằm quản trị vòng đời prompt, theo dõi luồng xử lý ứng dụng RAG (Retrieval-Augmented Generation) và đánh giá định lượng chất lượng câu trả lời:
- **Tracing & Observability:** Tích hợp LangSmith (`@traceable`) để theo dõi các run, latency, token usage và context retrieval.
- **Prompt Hub & A/B Routing:** Quản trị phiên bản prompt độc lập với mã nguồn trên LangSmith Prompt Hub (`nguyen-nguyen-phong-rag-prompt-v1` và `nguyen-nguyen-phong-rag-prompt-v2`), định tuyến tất định dựa trên hash MD5 của `request_id`.
- **RAGAS Evaluation:** Đánh giá định lượng qua 50 cặp câu hỏi - đáp án chuẩn (Ground Truth) theo 4 chỉ số cốt lõi: Faithfulness, Answer Relevancy, Context Recall, và Context Precision.
- **Guardrails AI:** Kiểm soát đầu ra LLM với custom validator `PIIDetector` (redact PII tự động với `OnFailAction.FIX`) và `JSONFormatter` (tự động sửa lỗi JSON phổ biến và cơ chế fallback).

---

## 2. Thiết kế Prompts (V1 vs V2)

### Phiên bản V1: `nguyen-nguyen-phong-rag-prompt-v1` (Ngắn gọn, Thân thiện)
- **Mục tiêu thiết kế:** Tối ưu hóa tính trung thực (`faithfulness`) và tốc độ phản hồi bằng cách giới hạn câu trả lời trong phạm vi 2–4 câu, chỉ tập trung vào sự kiện trực tiếp từ ngữ cảnh.
- **System Prompt:**
  ```text
  Bạn là trợ lý AI thân thiện. Trả lời ngắn gọn (2-4 câu), chỉ dựa trên context sau. Nếu không có thông tin, hãy nói thẳng là không biết.

  Context:
  {context}
  ```

### Phiên bản V2: `nguyen-nguyen-phong-rag-prompt-v2` (Chuyên gia, Có cấu trúc)
- **Mục tiêu thiết kế:** Tối ưu hóa tính đầy đủ và mạch lạc (`answer_relevancy` & giải thích rõ ràng) bằng cách yêu cầu phân tích các facts liên quan và cấu trúc câu trả lời logic trong 3–5 câu.
- **System Prompt:**
  ```text
  Bạn là chuyên gia phân tích thông tin. Đọc kỹ context, xác định các facts liên quan, rồi viết câu trả lời rõ ràng, có cấu trúc logic (3-5 câu). Tuyệt đối không suy đoán ngoài context.

  Context:
  {context}
  ```

---

## 3. Phân Tích So Sánh Kết Quả RAGAS (V1 vs V2)

### 3.1. Bảng tóm tắt chỉ số RAGAS

| Chỉ số (Metric) | Prompt V1 (Ngắn gọn) | Prompt V2 (Cấu trúc) | Xu hướng / Winner |
|---|---|---|---|
| **Faithfulness** | **0.9472** ⭐ | 0.9145 | **← V1** (V1 chiếm ưu thế nhờ trả lời súc tích, bám sát từng từ trong context) |
| **Answer Relevancy** | 0.9661 | **0.9782** | **← V2** (V2 chiếm ưu thế do phân tích toàn diện, có cấu trúc logic) |
| **Context Recall** | **1.0000** | **1.0000** | Hòa (Cả hai đều trích xuất và bao phủ trọn vẹn Ground Truth) |
| **Context Precision** | **0.9300** | **0.9300** | Hòa (Cùng sử dụng chung retriever FAISS với k=3 tài liệu) |

### 3.2. Giải thích vì sao có sự khác biệt giữa hai phiên bản

1. **Về chỉ số Faithfulness:**
   - **V1 đạt điểm Faithfulness cao hơn (thường vượt trội ≥ 0.90):** Do chỉ thị "ngắn gọn (2-4 câu)" và "nói thẳng là không biết nếu không có thông tin", LLM hầu như chỉ trích xuất các mệnh đề trực tiếp từ `context`, giảm thiểu tối đa hiện tượng "hallucination" hoặc suy diễn bắc cầu.
   - **V2 có thể gặp rủi ro nhỏ về Faithfulness:** Khi được yêu cầu "phân tích" và "có cấu trúc", LLM có xu hướng sử dụng thêm các từ nối logic hoặc diễn giải lại ý tưởng, khiến evaluator của RAGAS đôi khi nhận định là có thông tin bổ sung không được nêu tường minh trong đoạn trích.

2. **Về chỉ số Answer Relevancy:**
   - **V2 đạt điểm Answer Relevancy cao hơn:** Với phong cách phân tích chuyên sâu, V2 trả lời trọn vẹn và đa chiều hơn đối với các câu hỏi phức tạp đòi hỏi giải thích nguyên nhân hoặc quy trình. Người dùng nhận được câu trả lời có tính hoàn thiện và ngữ nghĩa phong phú hơn.

---

## 4. Quản Trị Prompt Hub & A/B Routing

- **Deterministic Routing:** Hệ thống sử dụng thuật toán hash MD5:
  ```python
  hash_int = int(hashlib.md5(request_id.encode()).hexdigest(), 16)
  return PROMPT_V1_NAME if hash_int % 2 == 0 else PROMPT_V2_NAME
  ```
  Nhờ đó, một `request_id` cụ thể luôn luôn nhận cùng một phiên bản prompt (tất định), đảm bảo tính nhất quán trong trải nghiệm người dùng và việc thu thập metrics phân tích A/B test.
- **Prompt Registry:** Việc tách prompt ra khỏi code và quản lý trên LangSmith Prompt Hub cho phép cập nhật, rollback phiên bản mà không cần deploy lại ứng dụng.

---

## 5. Danh Sách Minh Chứng (Evidence Deliverables)

1. `01_langsmith_traces.png`: Ảnh chụp giao diện LangSmith hiển thị ≥ 50 traces truy vấn RAG.
2. `02_prompt_hub.png`: Ảnh chụp giao diện Prompt Hub hiển thị 2 prompts V1 và V2.
3. `02_ab_routing_log.txt`: Console log của quá trình định tuyến A/B với các tag `[prompt-v1]` và `[prompt-v2]`.
4. `03_ragas_scores.png`: Ảnh chụp bảng điểm so sánh V1 vs V2 từ RAGAS evaluation.
5. `03_ragas_report.json`: File dữ liệu chi tiết kết quả đánh giá 4 metrics.
6. `04_pii_demo_log.txt`: Log kiểm thử che thông tin cá nhân (Email, Phone, SSN, Credit Card) bằng `PIIDetector`.
7. `04_json_demo_log.txt`: Log kiểm thử tự động sửa lỗi cú pháp JSON và fallback bằng `JSONFormatter`.

