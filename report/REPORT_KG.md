# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đinh Tiến Cảnh

**MSSV:** 2A202602918

**Ngày:** 05/10/2026

## 1. Chi phí (10 điểm)

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     84.4
graph       196     91958     4666   0.00930    168.9

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.85
graph       0.83   1.83     3910       74   0.00062     3.03
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0.00112 | 0.00930 | ×8.30 |
| Indexing giây | 84.4 | 168.9 | ×2.00 |
| Mỗi câu: USD | 0.00013 | 0.00062 | ×4.77 |
| Mỗi câu: giây | 1.85 | 3.03 | ×1.64 |
| Mỗi câu: in_tok | 694 | 3910 | ×5.63 |

**Chi phí tăng thêm đến từ đâu?** GraphRAG thực hiện thêm 20 lần gọi chat để trích xuất 20 bài báo khi indexing, làm tổng input tăng từ 56.072 lên 91.958 token và phát sinh 4.666 output token. Khi querying, prompt GraphRAG chứa cả vector chunks lẫn facts nhiều bước từ graph nên input trung bình cao gấp 5,63 lần.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm trong một đoạn luật nên Flat đã đủ; Graph bổ sung chính xác Điều 2 khoản 4 nhưng không tăng điểm. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Hai bị cáo và mức án cùng nằm trong một bài báo nên không cần đường đi nhiều bước. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat báo không đủ thông tin; Graph nối Lê Minh Thành → vụ → tội → Điều 251 và trả đủ 36 tháng cùng khung 02–07 năm. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph nối Hoàng Nato với tội tổ chức sử dụng trái phép chất ma túy, Điều 255 khoản 4 và mức tối đa chung thân. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Ontology định lượng nối 9,6 kg MDMA với đúng khoản 4 Điều 250 và khung 20 năm/chung thân/tử hình; Flat thiếu số điều và số khoản. |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai nêu đúng ba nội dung vụ MDMA nhưng không dùng đủ ba tên riêng mà `must_include` yêu cầu, nên keyword recall bằng 0 và judge chỉ cho 1. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E4: Keyword recall bằng 0 dù câu trả lời đúng một phần

- **Hiện tượng:** ở Q6, cả Flat và Graph đều liệt kê ba vụ có MDMA nhưng `recall=0.00`, trong khi LLM judge cho `1/2`. Điểm recall không phản ánh được phần nội dung đã trả lời đúng.
- **Bằng chứng:** `must_include` của Q6 yêu cầu chính xác ba chuỗi:

```json
["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
```

Câu trả lời Graph trong file benchmark là:

```text
1. Vụ vận chuyển ma túy từ Đức về Việt Nam: Tổng khối lượng hơn 9,6kg MDMA.
2. Vụ góp tiền mua ma túy tại Hà Nội: Có liên quan đến 5 viên MDMA.
3. Vụ tổ chức sử dụng ma túy tại Sầm Sơn: Có liên quan đến 0,686g MDMA.
```

Câu trả lời mô tả đúng nội dung ba vụ nhưng không chứa nguyên văn `Cái Quang Huy`, `Lê Minh Thành` và `Pháp y tâm thần`, nên keyword recall tính 0. LLM judge nhận ra câu đúng một phần và cho 1.

- **Nguyên nhân:** `keyword_recall` chỉ kiểm tra chuỗi bắt buộc, không hiểu rằng “vụ vận chuyển từ Đức”, “vụ góp tiền tại Hà Nội” và “vụ tại Sầm Sơn” đang nói đến các vụ trong gold. Đây là giới hạn của phép đo; đồng thời câu trả lời cũng chưa đủ định danh để người đọc đối chiếu chắc chắn.
- **Đề xuất sửa:** đánh giá Q6 theo tập `case_id`/thực thể chuẩn thay vì chuỗi tự do, hoặc thêm alias hợp lệ cho từng vụ. Vẫn giữ LLM judge để kiểm tra ý nghĩa. Đánh đổi là bộ đánh giá phức tạp hơn và LLM judge phát sinh thêm chi phí, nhưng tránh kết luận sai rằng một câu đúng một phần hoàn toàn không có recall.

### Lỗi E5: LLM không đưa tên người có sẵn trong graph vào câu aggregation

- **Hiện tượng:** graph đã lưu các quan hệ từ người tới những vụ có MDMA, nhưng câu trả lời Q6 chỉ dùng tên vụ chung chung. Vì vậy GraphRAG không nêu `Cái Quang Huy`, `Lê Minh Thành` hoặc cụm `Viện Pháp y tâm thần Trung ương` dù dữ liệu tương ứng tồn tại trong graph.
- **Bằng chứng bằng Cypher:**

```cypher
MATCH (p:Person)-[:INVOLVED_IN]->(k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})
WHERE p.name IN ['Cái Quang Huy', 'Lê Minh Thành', 'Lê Văn Đông']
RETURN DISTINCT p.name AS person, k.name AS case_name
ORDER BY person, case_name;
```

Kết quả quan sát được trên graph đầy đủ:

```text
Cái Quang Huy | Vụ vận chuyển ma túy từ Đức về Việt Nam
Lê Minh Thành  | Vụ góp tiền mua ma túy tại Hà Nội
Lê Văn Đông    | Vụ án tại Viện Pháp y tâm thần Trung ương
Lê Văn Đông    | Vụ tổ chức sử dụng ma túy tại Sầm Sơn
```

Trong khi đó, câu trả lời Graph Q6 chỉ ghi tên ba vụ, không ghi tên người hay tên cơ quan như gold; `recall=0.00`, `judge=1`.

- **Nguyên nhân:** `Neo4jGraph.context()` tìm các `Case` kề node `Substance`, nhưng phần mở rộng aggregation chủ yếu đưa cạnh `Case-[:INVOLVES]->Substance` và tóm tắt vụ vào prompt. Quan hệ `Person-[:INVOLVED_IN]->Case` nằm thêm một bước nên có thể không xuất hiện trong tập facts giới hạn; LLM không thể nhắc lại tên không có trong context nổi bật. Lỗi nằm ở Cypher/context selection và sau đó biểu hiện ở câu trả lời LLM.
- **Đề xuất sửa:** khi câu hỏi mang tính tổng hợp “những vụ nào”, truy vấn thêm người đại diện cho từng case và đưa fact đọc được như `Vụ X — người: Y — chất: MDMA` vào prompt. Có thể ưu tiên người có vai trò bị cáo/bị can để không đưa toàn bộ người liên quan. Đánh đổi là prompt dài hơn; cần `DISTINCT`, giới hạn số người mỗi vụ và sắp xếp ổn định để kiểm soát token.

## 4. Kết luận (5 điểm)

Flat RAG phù hợp khi đáp án nằm gọn trong một tài liệu: Q1 và Q2 đều đạt recall 1.00, judge 2 với chi phí trung bình 0,00013 USD và 1,85 giây/câu. Knowledge Graph đáng dùng khi phải nối nhiều nguồn: Q3–Q4 chuyển từ `0.00/0` ở Flat thành `1.00/2` ở Graph; Q5 cũng tăng từ `0.60/1` lên `1.00/2` nhờ node `QuantityThreshold`. Đổi lại, Graph tốn 8,30 lần chi phí indexing, 4,77 lần chi phí mỗi câu và 1,64 lần độ trễ. Q6 cho thấy KG không tự bảo đảm câu trả lời tốt: nếu context bỏ tên người thì LLM vẫn trả thiếu và phép đo chuỗi có thể đánh giá lệch. Vì vậy nên dùng KG khi dữ liệu có thực thể chung ổn định và nhu cầu cross-KB/multi-hop đủ lớn; với tra cứu định nghĩa hoặc một bài báo đơn lẻ, Flat RAG đơn giản và kinh tế hơn.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 368 node / 564 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 10 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064.
```

Ảnh Neo4j:

- `report/img/kg_count.png`: Q-A, thấy đủ 8 label của ontology và tổng 419 node / 645 cạnh.
- `report/img/kg_cross_kb.png`: Q-B, thấy đường `Person → Case → Crime ← Article`, ô truy vấn và Results overview.
- `report/img/kg_my_case.png`: Q-D với **Phan Kim Nhi**, thấy đường từ người qua các vụ án, tội danh tới điều luật, kèm địa điểm.

**Người đã chọn cho `kg_my_case.png`:** Phan Kim Nhi.

Truy vấn đã dùng:

```cypher
MATCH p=(:Person {name:'Phan Kim Nhi'})-[:INVOLVED_IN]->(k:Case)
        -[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article)
OPTIONAL MATCH q=(k)-[:INVOLVES|LOCATED_IN]->()
RETURN p, q;
```

## Vấn đề gặp phải (không tính điểm)
