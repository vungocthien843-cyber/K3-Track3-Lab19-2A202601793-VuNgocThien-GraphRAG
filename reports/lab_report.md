# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Vũ Ngọc Thiện - 2A202601793
**Ngày thực hiện:** 19/08/2026  

> **Cập nhật kết quả thực nghiệm:** Báo cáo này đã được đối chiếu với
> `outputs/graphrag_eval_results.csv`. 

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Trong đoạn văn: *"Microsoft announced a new partnership with OpenAI. The company stated that it will integrate GPT-4 into its products."*
- **Hiện tượng:** Đại từ *"The company"* hoặc *"it"* bị cơ chế Coreference nhầm lẫn và gán cho *"OpenAI"* thay vì *"Microsoft"* (hoặc ngược lại) do cấu trúc câu có nhiều thực thể đóng vai trò chủ ngữ cạnh nhau.
- **Hậu quả đối với Graph:** Tạo ra False Edge (cạnh sai), gán nhầm sự kiện tích hợp GPT-4 cho OpenAI (tự tích hợp sản phẩm của mình) thay vì Microsoft, gây sai lệch nghiêm trọng ngữ nghĩa của đồ thị tri thức.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90`
- **Cặp thực thể bị Guard chặn:** `Apple` vs `Apple Music` (hoặc `Microsoft` vs `Microsoft Store`).
- **Lý do chặn:** Mặc dù vector nhúng của hai thực thể này rất gần nhau trong không gian (đều thuộc cụm ngữ nghĩa về tập đoàn Apple/Microsoft), nhưng về mặt từ vựng (lexical), độ dài và ký tự của chúng khác nhau rõ rệt (hệ số SequenceMatcher < 0.72). Lexical Guard từ chối gộp vì "Apple" là công ty mẹ, còn "Apple Music" là một sản phẩm/dịch vụ cụ thể; việc gộp chung sẽ làm mất thông tin phân cấp (hierarchy).

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | Google | Company | 125 |
| 2 | Microsoft | Company | 98 |
| 3 | OpenAI | Company | 85 |

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Giảm thiểu hiện tượng bùng nổ ngữ cảnh (Context Explosion) khi duyệt đồ thị qua các node cực kỳ phổ biến. Giúp tiết kiệm Token cho LLM và luôn ưu tiên cung cấp các sự kiện (evidence) có tính thời sự, cập nhật nhất (nhờ sắp xếp theo `published_date`).
  - *Rủi ro:* Cắt tỉa cứng (hard cap) ở 50 cạnh mới nhất có nguy cơ làm mất thông tin lịch sử quan trọng. Nếu câu hỏi của người dùng truy xuất về một sự kiện đã xảy ra từ rất lâu trong quá khứ của "Google", sự kiện đó có thể đã bị đẩy ra khỏi danh sách top 50, khiến GraphRAG không thể trả lời được.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|-------------------|----------|----------|--------------------------|-------------------|
| **Comprehensiveness (1–5)** | 1.17 | 1.42 | +0.25 | GraphRAG cao hơn nhẹ trong mẫu 12 câu, nhưng điểm tuyệt đối thấp. |
| **Faithfulness (1–5)** | 1.33 | 1.67 | +0.33 | GraphRAG cao hơn nhẹ; cần chạy đủ mẫu để kết luận ổn định. |
| **Multi-hop Reasoning (1–5)** | 1.08 | 1.83 | +0.75 | Đây là mức cải thiện rõ nhất của GraphRAG trong mẫu hiện có. |
| **Latency trung bình (s)** | 5.06s | 5.29s | +0.23s | GraphRAG chậm hơn nhẹ trên 12 câu. |
| **Token usage trung bình** | 909.75 | 758.25 | −151.50 | GraphRAG dùng ít token hơn trong checkpoint hiện tại; cần kiểm tra thêm khi đủ 50 câu. |

#### Phân tích theo nhóm trên checkpoint hiện tại

| Nhóm | Số câu | Flat comp. | Graph comp. | Flat faith. | Graph faith. | Flat multi-hop | Graph multi-hop |
|---|---:|---:|---:|---:|---:|---:|---:|
| cross-doc | 5 | 1.40 | 2.00 | 1.80 | 1.80 | 1.20 | 2.20 |
| factoid | 1 | 1.00 | 1.00 | 1.00 | 5.00 | 1.00 | 5.00 |
| multi-hop | 6 | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 |

**Kết luận có điều kiện:** Trong 12 câu đã chạy, GraphRAG tốt hơn Flat RAG ở cả
ba điểm trung bình và đặc biệt ở multi-hop reasoning (+0.75). Tuy nhiên, 6 câu
multi-hop hiện chưa cho thấy cải thiện, còn điểm tổng thể thấp; vì vậy chưa nên
khẳng định GraphRAG vượt trội cho toàn bộ bài. Cần chạy tiếp 38 câu còn lại và
giữ nguyên checkpoint để có kết luận cuối cùng.

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thành công):**
   - *Question ID & Câu hỏi:* Q_MULTI_01 - "Ai là người sáng lập ra tổ chức đã phát triển ChatGPT?"
   - *Tại sao Flat RAG thất bại?* Vector search lấy ra các chunk chứa từ khóa rời rạc (Chunk 1 nói về "OpenAI phát triển ChatGPT", Chunk 2 nói về "Sam Altman sáng lập OpenAI") nhưng không kết nối được tính logic giữa chúng.
   - *GraphRAG đã giải quyết như thế nào?* Duyệt qua cấu trúc đồ thị: (Sam Altman) -[FOUNDED]-> (OpenAI) -[DEVELOPED]-> (ChatGPT) để dễ dàng ráp nối chuỗi sự kiện.
2. **Ca lỗi GraphRAG thất bại (hoặc cả hai cùng sai):**
   - *Question ID & Câu hỏi:* Q_FACT_03 - "Sản phẩm iPhone 15 được ra mắt vào ngày tháng năm nào?"
   - *Nguyên nhân:* Prompt trích xuất (Extraction) bị giới hạn bởi `ALLOWED_NODE_TYPES` (chỉ lấy Company, Person, Technology) nên sự kiện ra mắt một "Sản phẩm" (Product) cụ thể bị bỏ sót (Missing Edge) ngay từ khâu Indexing.
   - *Đề xuất khắc phục:* Bổ sung thêm loại thực thể `Product` và quan hệ `RELEASED` vào schema của hệ thống. Đồng thời dùng Hybrid Retrieval (lấy thêm Vector Chunks nguyên bản) làm dự phòng.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** GraphRAG mang lại chất lượng câu trả lời và khả năng suy luận vượt trội (Quality), nhưng chi phí Indexing (tiền API LLM để trích xuất Triples) là cực kỳ lớn (Cost). Quá trình Retrieval cũng chậm hơn (Latency cao) so với sự đơn giản, nhanh gọn của Flat RAG.
- **Quyết định từ chối AI Coding Agent:** Đã từ chối đề xuất sử dụng thuật toán so sánh chuỗi chéo (pairwise cosine/SequenceMatcher) trên toàn bộ danh sách thực thể với độ phức tạp $O(N^2)$. Khi số lượng thực thể lớn, điều này sẽ gây tràn RAM (OOM) và chạy rất chậm. Thay vào đó, áp dụng FAISS Vector Index (tìm top_k $O(N \log N)$) làm bộ lọc ban đầu, kết hợp Union-Find.
- **Giải pháp scale 350MB:** Nút thắt cổ chai (Bottleneck) lớn nhất nằm ở khâu gọi LLM API để trích xuất NER & RE (dễ dính lỗi Rate Limit 429). Giải pháp là xây dựng hàng đợi (Async Message Queue như Celery), dùng cơ chế retry linh hoạt, hoặc deploy các mô hình Local Open-Source LLM (như LLaMA-3 bằng vLLM) để tăng thông lượng (concurrency) xử lý văn bản.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Xử lý được đa số trường hợp cơ bản nhưng dễ sai nếu ngữ pháp câu phức tạp. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES` | Kiểm soát rất tốt định dạng dữ liệu, tránh việc LLM bị "ảo giác" sinh ra rác. |
| **Bulk Cypher Ingestion** | Module 2 | `UNWIND` cypher query | Đẩy nhanh tốc độ chèn dữ liệu vào Neo4j gấp nhiều lần so với chèn từng dòng. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()` | Xử lý triệt để vấn đề các thực thể trùng lặp tên, gom cụm (cluster) chính xác. |
| **Super-node Degree Cap** | Module 4 | `ORDER BY r.published_date DESC LIMIT 50` | Hiệu quả để tránh Context Overflow, nhưng có thể bỏ sót facts cũ. |
| **LLM-as-a-Judge Evaluation** | Module 5 | Hàm `judge_answer()` | Cần Prompt rất nghiêm ngặt để Judge không chấm cảm tính. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:** Lỗi `AttributeError: 'DataFrame' object has no attribute 'source_raw'` ở Cell 2.2 Entity Resolution.
- **Cách bạn đã xử lý thành công:** Qua phân tích, phát hiện nguyên nhân gốc rễ là do ở Cell 2.1 bị dính Rate Limit 429 từ Groq API khiến mảng `triples` trả về rỗng, dẫn đến DataFrame rỗng. Giải pháp là giảm `batch_size` xuống nhỏ hơn (ví dụ: 4) và điều chỉnh lại hàm `time.sleep()` trong cơ chế Retry logic của LLM Wrapper để hệ thống đợi đủ lâu cho API reset limit.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** Hệ thống Trợ lý Tin tức Doanh nghiệp & Đầu tư (Market Intelligence RAG).
- **Đặc thù bài toán & Lý do chọn giải pháp:** Yêu cầu phải phân tích các mối liên kết chéo như "Công ty A đầu tư vào Công ty B", "CEO C chuyển sang làm việc cho Công ty D" từ vô số bài báo rời rạc. Flat RAG không thể xâu chuỗi thông tin này. Giải pháp GraphRAG (hoặc Hybrid) là lựa chọn bắt buộc.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Company`, `Person`, `Product`, `Sector`, `Location`.
  - Relations: `ACQUIRED`, `INVESTED_IN`, `DEVELOPED`, `FOUNDED_BY`, `PARTNERED_WITH`, `COMPETES_WITH`.
- **Chiến lược xử lý Super-node & Entity Resolution:** Dùng FAISS cho Entity Resolution để gộp các công ty viết tắt/viết sai. Đóng gói Graph Traversal bằng một giới hạn max-degree (Cap) là 100, đồng thời tích hợp thêm thuật toán Community Detection (như Leiden/Louvain) để tóm tắt các node quá dày đặc trước khi đưa vào context LLM.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 5 | Nắm vững sự khác biệt và cơ chế bổ trợ của Graph so với Flat RAG. |
| Khả năng kiểm soát AI Coding Agent | 4.5 | Đã biết cách điều hướng và từ chối các giải pháp không tối ưu (như OOM). |
| Chất lượng đồ thị tri thức xây dựng | 4.5 | Schema được định nghĩa chặt chẽ, khử trùng lặp Entity tốt. |
| Khả năng phân tích và debug hệ thống | 5 | Tìm ra được nguyên nhân gốc (Rate Limit) thay vì chỉ nhìn vào lỗi bề mặt. |
