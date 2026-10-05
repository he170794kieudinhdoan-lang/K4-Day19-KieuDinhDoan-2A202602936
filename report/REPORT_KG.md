# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Kiều Đình Đoàn  **MSSV:** 2A202602936  **Ngày:** 05/10/2026

> Dữ liệu được đo đạc và sinh ra thực tế từ lệnh `python bench_kg.py --judge` với Model Chat `gemini-3.5-flash-lite` và Embedding `gemini-embedding-001`. Toàn bộ số liệu bên dưới khớp 100% với file `ket_qua_benchmark_kg.txt` (bản ontology tự thiết kế xét bonus +15; bản đối chiếu gốc nộp kèm ở `ket_qua_benchmark_kg.hint.txt`).

## 1. Chi phí (10 điểm)

Hai bảng dữ liệu trích xuất nguyên văn từ `ket_qua_benchmark_kg.txt`:

```text
Chat model: gemini:gemini-3.5-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 208 nodes / 386 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000     98.2
graph       196     34619     5897   0.00582    137.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       72   0.00010     1.60
graph       0.91   1.67     4530      142   0.00051     2.35
```

Bảng so sánh tổng hợp chỉ số:

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00582 | +$0.00582 |
| Indexing giây | 98.2s | 137.3s | ×1.40 |
| Mỗi câu: USD | $0.00010 | $0.00051 | ×5.10 |
| Mỗi câu: giây | 1.60s | 2.35s | ×1.47 |
| Mỗi câu: in_tok | 696 | 4530 | ×6.51 |

**Chi phí tăng thêm đến từ đâu?**
1. **Ở pha Indexing:** GraphRAG phải gánh thêm chi phí trích xuất thực thể có cấu trúc từ 20 bài báo tin tức thông qua LLM (gọi thêm 20 lần LLM chat JSON mode, tiêu tốn 34.619 input tokens và 5.897 output tokens, mất $0.00582 và thêm ~39s), trong khi Flat RAG chỉ cần embed các chunk văn bản một chiều.
2. **Ở pha Querying:** Chi phí mỗi câu hỏi của GraphRAG cao gấp 5.1 lần và in_tok cao gấp 6.5 lần do prompt phải "nhồi" thêm danh sách facts phong phú lấy từ đồ thị multi-hop (các điều luật, khoản hình phạt định lượng theo tang vật, tóm tắt vụ việc, danh sách người liên quan).
3. **Đánh giá ROI:** Với khoản đầu tư ban đầu cực kỳ "hạt dẻ" (< $0.006 cho một lần build graph), GraphRAG đưa Recall nhảy vọt từ 0.51 lên 0.91 (gần như gấp đôi). Điểm hòa vốn về mặt giá trị thông tin đạt được ngay lập tức ở các bài toán thực tế đòi hỏi độ chuẩn xác pháp lý cao.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Câu hỏi định nghĩa luật đơn lẻ, vector search của Flat RAG tìm trúng ngay chunk chứa khoản 4 Điều 2 Luật PCMT 2021 nên cả hai đều trả lời trọn vẹn. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Câu hỏi thuần tin tức, ngữ cảnh nằm gọn trong 1 bài báo về vụ 36kg ma túy nên Flat RAG tìm đúng và trả về 2 án tử hình chuẩn xác. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ tìm được bài báo về Lê Minh Thành (36 tháng tù) và đầu hàng trước câu hỏi điều luật; GraphRAG lần theo node Crime sang Điều 251 khoản 1 chuẩn chỉ. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | **Graph** | Flat RAG chỉ tìm được bài báo, không biết điều luật; GraphRAG đi qua quan hệ CHARGED_WITH sang Điều 255 BLHS chỉ rõ tội tổ chức sử dụng ma túy. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 0.80 / 1 | **Graph** | Nhờ ontology tự thiết kế mô hình hóa ngưỡng khối lượng, GraphRAG xác định chuẩn xác khối lượng >100g MDMA thuộc Khoản 4 Điều 250 BLHS (Recall tăng từ 0.60 lên 0.80). |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | **Graph** | Flat RAG trích xuất các vụ ẩn danh không kèm tên người nên Recall = 0; GraphRAG duyệt từ node MDMA tìm trúng cả 3 thực thể bắt buộc (Cái Quang Huy, Lê Minh Thành, Viện Pháp y tâm thần) và đạt điểm tuyệt đối 2/2. |

## 3. Phân tích lỗi (20 điểm)

Dưới đây là 3 lỗi điển hình phản ánh chân thực những vấn đề thường gặp của hệ thống GraphRAG ngoài đời thực:

---

### Lỗi 1: E2 — Thiếu ngữ cảnh luật (Missing Legal Context) ở câu hỏi suy luận định lượng Q5 & Cách khắc phục thành công

- **Hiện tượng ban đầu:** Ở ontology gợi ý gốc, GraphRAG chỉ trích xuất được tội danh "vận chuyển trái phép chất ma túy" và chất "MDMA" nhưng lúng túng khi xác định khoản áp dụng cụ thể cho khối lượng 9,6kg, do đồ thị không có dữ kiện so sánh số học (Recall chỉ đạt 0.60, xem file `ket_qua_benchmark_kg.hint.txt`).
- **Bằng chứng sau khi nâng cấp ontology:**
  - Trích nguyên văn kết quả Q5 của GraphRAG trong `ket_qua_benchmark_kg.txt`:
    > *"Dựa trên ngữ cảnh và dữ kiện knowledge graph: ... Khối lượng MDMA: Hơn 9,6kg (tương đương 9.600 gam) MDMA. Điều luật và khoản áp dụng: Với khối lượng MDMA từ 100 gam trở lên (khoản 4), các điều luật tương ứng đề cập đến hành vi này bao gồm Điều 250 BLHS khoản 4 (Tội vận chuyển trái phép chất ma túy)..."*
  - Kiểm tra đồ thị bằng Cypher:
    ```cypher
    MATCH (k:Case)-[inv:INVOLVES]->(s:Substance {name: 'MDMA'})<-[m:MENTIONS]-(cl:Clause)<-[:HAS_CLAUSE]-(a:Article)
    WHERE inv.amount_grams >= m.min_grams AND (m.max_grams IS NULL OR inv.amount_grams < m.max_grams)
    RETURN a.id, cl.number, inv.amount, m.desc;
    ```
    *Kết quả trên Neo4j:*
    ```text
    a.id: "Điều 250 BLHS"
    cl.number: 4
    inv.amount: "hơn 9,6kg"
    m.desc: "từ 100 gam trở lên"
    ```
- **Nguyên nhân gốc rễ:** Đồ thị cũ chỉ lưu text thô `amount: "hơn 9,6kg"` nên không có quan hệ kết nối số học với quy định pháp lý.
- **Cách khắc phục:** Mô hình hóa `min_grams` trên quan hệ `MENTIONS` và `amount_grams` trên quan hệ `INVOLVES`, giúp Cypher so sánh số học trực tiếp và chỉ điểm chính xác Khoản 4 Điều 250 BLHS.

---

### Lỗi 2: E4 — Phép đo mâu thuẫn giữa Recall và LLM Judge ở câu hỏi gom cụm Q6

- **Hiện tượng:** Ở câu Q6, Flat RAG có `recall = 0.00` nhưng `judge = 2` (điểm tuyệt đối); trong khi GraphRAG có `recall = 1.00` và `judge = 2`.
- **Bằng chứng:**
  - Trích nguyên văn so sánh câu Q6 trong `ket_qua_benchmark_kg.txt`:
    > **Flat RAG (Recall = 0.00, Judge = 2):**
    > *"Dựa trên ngữ cảnh, cả 3 vụ việc đều có liên quan đến ma túy MDMA:*
    > *1. Vụ việc [1]: Lực lượng chức năng phát hiện bên trong thùng hàng có các viên nén màu xanh là MDMA...*
    > *2. Vụ việc [2]: Thành bị bắt quả tang khi mang 5 viên nén màu trắng đi bán...*
    > *3. Vụ việc [3]: Kết quả giám định xác định số viên nén hình tam giác... MDMA..."*
    > 
    > **GraphRAG (Recall = 1.00, Judge = 2):**
    > *"Dựa trên ngữ cảnh và dữ kiện từ knowledge graph, các vụ việc liên quan đến ma túy MDMA bao gồm:*
    > *1. Vụ vận chuyển ma túy qua sân bay Nội Bài do Cái Quang Huy thực hiện...*
    > *2. Vụ mua bán trái phép chất ma túy do Lê Minh Thành và đồng phạm...*
    > *3. Vụ việc tại Viện Pháp y tâm thần Trung ương..."*
- **Nguyên nhân:**
  1. *Điểm yếu của Keyword Recall:* Tập từ khóa bắt buộc `must_include` là `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`. Flat RAG trích xuất các đoạn văn bản nhưng dùng cách gọi ẩn danh theo nguồn (`Vụ việc [1]`, `Vụ việc [2]`) mà không nêu đích danh họ tên đầy đủ, khiến `keyword_recall` đánh giá trượt toàn bộ (0.00). Tuy nhiên, LLM judge khi chấm lại đọc hiểu ngữ nghĩa thấy có 3 vụ án nói về MDMA nên hào phóng cho điểm 2.
  2. *Sức mạnh của GraphRAG:* Duyệt theo đồ thị từ node `MDMA` tìm ra đầy đủ cả 3 đối tượng cụ thể bằng họ tên thật, đạt điểm tuyệt đối ở cả 2 phép đo (Recall 1.00 & Judge 2).
- **Đề xuất sửa:** Cần tinh chỉnh bộ benchmark test suite để kết hợp cả phép đo thực thể (Entity Coverage) lẫn phép đo ngữ nghĩa ngữ cảnh.

---

### Lỗi 3: E5 — LLM lệch với Graph (Attention Bleeding) trong phân bổ quan hệ

- **Hiện tượng:** Ở các câu hỏi tổng hợp nhiều vụ án, việc gom tất cả facts vào một danh sách phẳng đôi khi khiến LLM bị hiện tượng trôi chú ý, dễ liên tưởng các Điều luật thuộc vụ án này sang vụ án khác nếu các facts không được phân nhóm theo Case ID.
- **Bằng chứng:** Trong các câu trả lời tổng hợp diện rộng, LLM có xu hướng gộp chung các điều luật liên quan của cả nhóm vụ án thay vì bóc tách rạch ròi từng vụ.
- **Nguyên nhân:** Cấu trúc facts trong KG-3 trả về danh sách 1 chiều, thiếu tính đóng gói phân cấp (Hierarchical Encapsulation).
- **Đề xuất sửa:** Nhóm facts theo cấu trúc:
  ```text
  Vụ án A: [Điều luật tương ứng]
  Vụ án B: [Điều luật tương ứng]
  ```
  giúp LLM tập trung sự chú ý chính xác vào từng thực thể.

## 4. Kết luận (5 điểm)

Từ thực nghiệm đối đầu trực tiếp trên cùng một tập dữ liệu và cùng một mô hình ngôn ngữ:

1. **Khi nào NÊN dùng Knowledge Graph & GraphRAG:**
   - **Khi bài toán đòi hỏi suy luận nhiều bước (Multi-hop) và liên kết liên miền (Cross-domain):** Minh chứng thép là các câu hỏi Q3, Q4 và Q5. Khi cần nối từ một vụ việc đời thực sang khung điều luật pháp lý và định lượng mức phạt, Flat RAG hoàn toàn thất bại (Recall 0.33 - 0.40). GraphRAG giải quyết xuất sắc với Recall 0.80 - 1.00 nhờ các đường đi quan hệ tường minh (`Case` → `Crime` → `Article` → `Clause`).
   - **Khi cần tổng hợp, gom cụm thực thể (Aggregation):** Ở câu Q6, GraphRAG lần theo cạnh đồ thị từ node `MDMA` để gom trọn vẹn mọi vụ án và nhân vật liên quan mà vector search bị bỏ sót do giới hạn `top_k`.
2. **Khi nào Flat RAG là ĐỦ:**
   - **Khi câu hỏi là đơn bước (Single-hop) nằm trọn trong một tài liệu:** Minh chứng ở Q1 (định nghĩa tiền chất trong luật) và Q2 (tên bị cáo tử hình trong tin tức). Ở các tác vụ này, Flat RAG đạt điểm tuyệt đối (Recall 1.00, Judge 2) mà không cần đến đồ thị.
   - **Khi ưu tiên tốc độ và chi phí:** Flat RAG nhanh hơn 1.47 lần, rẻ hơn 5.1 lần trên mỗi câu hỏi và hoàn toàn không tốn chi phí trích xuất ban đầu.
3. **Chiến lược tối ưu cho sản phẩm thực tế:** Mô hình lai **Hybrid RAG** (kết hợp Vector Search cho ngữ cảnh ngữ nghĩa thô và Graph Traversal cho các mối liên kết có cấu trúc) là con đường chuẩn mực nhất, tận dụng ưu điểm của cả hai thế giới: độ phủ rộng của vector và độ chuẩn xác logic của đồ thị tri thức.

## 5. Tự kiểm (Checklist hoàn thành)

Output kiểm tra tự động trước khi đóng gói nộp bài:

### Output `pytest tests/ -q`
```text
................................................                         [100%]
48 passed in 0.28s
```

### Output `python bench_kg.py --check`
```text
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 33 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00055. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```
