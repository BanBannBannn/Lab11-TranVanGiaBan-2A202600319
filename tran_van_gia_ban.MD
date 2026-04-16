**Course:** AICB-P1 — AI Agent Development  
**Student Name:** Trần Văn Gia Bân  
**Student ID:** 2A202600319

---

## 1. Layer Analysis
Dưới đây là phân tích về lớp bảo mật đầu tiên phát hiện và ngăn chặn các mẫu tấn công trong Test 2.

| # | Attack Prompt Category | Primary Layer (First Catch) | Secondary Layers (Redundancy) |
|---|------------------------|-----------------------------|-------------------------------|
| 1 | Instructions Override  | **Input Guardrails (Regex)**| NeMo Guardrails, LLM-as-Judge |
| 2 | Secret Extraction      | **Input Guardrails (Regex)**| NeMo Guardrails, Output Guardrails |
| 3 | Authority Impersonation| **NeMo Guardrails** | LLM-as-Judge |
| 4 | Output Reformatting    | **Input Guardrails (Regex)**| LLM-as-Judge |
| 5 | Language Bypass (VN)   | **NeMo Guardrails** | Input Guardrails (Regex) |
| 6 | Fill-in-the-blank      | **Output Guardrails** | LLM-as-Judge |
| 7 | Creative Writing       | **LLM-as-Judge** | Output Guardrails (Redaction) |

---

## 2. False Positive Analysis
Trong quá trình thử nghiệm với các câu hỏi an toàn (Test 1), hệ thống hoạt động ổn định và **không có câu hỏi nào bị chặn nhầm**.

- **Thử nghiệm tăng cường:** Khi tôi thử thắt chặt `Topic Filter` bằng cách chỉ cho phép các từ khóa cực kỳ hẹp liên quan đến "số dư", các câu hỏi hỗ trợ chung như *"Làm sao để đổi mật khẩu?"* bắt đầu bị chặn nhầm (False Positive).
- **Đánh đổi (Trade-off):** - **Security cao:** Ngăn chặn triệt để các hành vi tán gẫu (chitchat) để lừa AI, nhưng gây khó chịu cho người dùng thật.
    - **Usability cao:** Giúp trải nghiệm mượt mà nhưng mở rộng "attack surface" cho kẻ tấn công. 
    - **Kết luận:** Cần sử dụng Semantic Similarity thay vì khớp từ khóa cứng để hiểu đúng ý định người dùng.

---

## 3. Gap Analysis
Mặc dù hệ thống có nhiều lớp, vẫn tồn tại các kịch bản có thể vượt qua:

| # | Proposed Attack Prompt | Why it bypasses current layers | Proposed Additional Layer |
|---|------------------------|--------------------------------|---------------------------|
| 1 | **Base64 Encoded Injection** | Các bộ lọc Regex hiện tại không thể đọc hiểu nội dung đã mã hóa Base64/Hex. | **Decoding Layer:** Tự động giải mã các chuỗi mã hóa trước khi đưa vào pipeline kiểm tra. |
| 2 | **Multi-turn Social Engineering** | Pipeline hiện tại chỉ kiểm tra từng tin nhắn đơn lẻ, không phân tích được ý đồ qua cả hội thoại. | **Session Anomaly Detector:** Theo dõi trạng thái và lịch sử hội thoại để phát hiện hành vi dò xét thông tin. |
| 3 | **Token Smuggling** | Sử dụng các ký tự đặc biệt hoặc dấu cách lạ để làm nhiễu bộ lọc từ khóa. | **Text Normalization Layer:** Chuẩn hóa văn bản đầu vào (xóa dấu cách thừa, ký tự lạ) trước khi xử lý. |

---

## 4. Production Readiness
Nếu triển khai cho 10,000 người dùng tại một ngân hàng thực tế, tôi sẽ thực hiện các thay đổi sau:

1.  **Tối ưu độ trễ (Latency):** Chạy song song (Parallel) các lớp Input Guardrails và NeMo thay vì chạy nối tiếp. Chỉ gọi LLM-as-Judge đối với các tác vụ nhạy cảm để giảm số lượng LLM call.
2.  **Quản lý chi phí:** Sử dụng các model nhỏ (Gemini Flash-Lite) cho các lớp kiểm tra trung gian để tiết kiệm token.
3.  **Cập nhật quy tắc động:** Lưu trữ Regex và quy tắc Colang trên Database/Redis thay vì hard-code, giúp cập nhật luật mới ngay lập tức mà không cần khởi động lại hệ thống.
4.  **Giám sát (Monitoring):** Thiết lập dashboard theo dõi tỷ lệ chặn (block rate) theo thời gian thực để phát hiện các đợt tấn công quy mô lớn (DDoS prompt).

---

## 5. Ethical Reflection
**Hệ thống an toàn tuyệt đối?** Không bao giờ. AI là một hệ thống xác suất, và kẻ tấn công luôn tìm ra những kịch bản "out-of-distribution" mà Guardrails chưa từng thấy.

**Từ chối vs. Đưa ra cảnh báo:**
- **Từ chối (Refuse):** Khi yêu cầu vi phạm pháp luật hoặc đòi quyền truy cập admin/password.
- **Cảnh báo (Disclaimer):** Khi AI cung cấp thông tin tài chính nhạy cảm.
- *Ví dụ cụ thể:* Khi khách hỏi về lãi suất vay, AI sẽ trả lời con số ước tính nhưng bắt buộc đính kèm câu: *"Thông tin này chỉ mang tính tham khảo, kết quả cuối cùng phụ thuộc vào phê duyệt hồ sơ thực tế tại quầy."*

---

## 6. Bonus Layer: Language Detection Filter
Tôi đề xuất thêm lớp **Language Detection Layer**. 
- **Cơ chế:** Sử dụng thư viện `langdetect` để chỉ cho phép Tiếng Anh và Tiếng Việt.
- **Lý do:** Nhiều kỹ thuật Jailbreak hiện nay sử dụng các ngôn ngữ hiếm (như tiếng Swahili) để lách qua các bộ lọc từ khóa của hệ thống. Việc giới hạn ngôn ngữ hỗ trợ sẽ chặn đứng các cuộc tấn công này một cách hiệu quả ngay từ đầu.
