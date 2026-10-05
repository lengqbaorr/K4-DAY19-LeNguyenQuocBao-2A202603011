# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Lê Nguyễn Quốc Bảo  **MSSV:** 2A202603011  **Ngày:** 2026-10-05

Kết quả chính ở `ket_qua_benchmark_kg.txt`, mốc ontology gợi ý ở `ket_qua_benchmark_kg.hint.txt`. Cả hai chạy chat OpenAI `gpt-4o-mini`, embedding OpenAI `text-embedding-3-small`, `top_k=3`, `chunk_size=800`, 176 chunk. Graph cuối có 201 node/384 cạnh. `judge` là điểm LLM 0–2; `recall` là tỉ lệ từ khóa nên không đồng nghĩa độ đúng pháp lý. USD là ước tính từ bảng giá trong `src/llm.py`.

## 1. Chi phí

**Indexing (one-off)** — sao từ file kết quả cuối:

```text
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     43.7
graph       196     91958     4756   0.00936    173.7
```

**Querying (mean per question)** — sao từ file kết quả cuối:

```text
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.59
graph       0.89   1.83     5546       82   0.00087     2.32
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0.00112 | 0.00936 | 8.36× |
| Indexing giây | 43.7 | 173.7 | 3.97× |
| Mỗi câu: USD | 0.00013 | 0.00087 | 6.69× |
| Mỗi câu: giây | 1.59 | 2.32 | 1.46× |
| Mỗi câu: input tokens | 694 | 5546 | 7.99× |

Indexing Graph tăng 20 cuộc gọi và khoảng 35.886 input token vì dùng LLM trích tin (luật trích bằng regex). Khi hỏi, Graph giữ cùng top-k vector rồi thêm các dòng graph, nên input token và chi phí tăng. Với N câu hỏi, chi phí ước tính Flat = `0.00112 + 0.00013N` USD; Graph = `0.00936 + 0.00087N` USD. Hai đường không có điểm hòa vốn theo **USD thuần** vì Graph tốn thêm `0.00824 + 0.00074N` USD với mọi N≥0. Đổi lại, Graph có 5/6 câu judge=2 so với Flat 2/6; cần định giá lợi ích của câu đúng mới quyết định được ngưỡng kinh tế.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Lý do |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Một chunk đủ định nghĩa tiền chất. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Một bài đủ hai tên bị cáo tử hình. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Qua Crime tới Điều 251 khoản 1; Flat trả “Không đủ thông tin.” |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph đưa khoản nặng nhất của Điều 255 nên trả đúng chung thân. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Graph nối MDMA và Điều 250 khoản 4; Flat thiếu Điều/khoản. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph về recall; hòa judge | Thiếu tên Cái Quang Huy, Lê Minh Thành và gán MDMA cho vụ 36kg không có cạnh MDMA trong graph. |

Quy luật ở mẫu nhỏ này: câu một nguồn Q1–Q2 không cần graph; câu xuyên tin–luật Q3–Q5 hưởng lợi rõ. Q6 cho thấy có đường graph chưa đủ bảo đảm phần tổng hợp cuối cùng nêu đúng thực thể bắt buộc.

## 3. Phân tích lỗi có bằng chứng

### E2 — Thiếu khoản phạt cao nhất trong ngữ cảnh

- **Hiện tượng:** Bản gợi ý ở Q4 GraphRAG trả “tối đa 7 năm theo Điều 255”, sai so với khoản 4; recall 0.67/judge 1. Bản mới trả “tối đa 20 năm hoặc tù chung thân theo Điều 255 BLHS khoản 4”, recall 1.00/judge 2.
- **Bằng chứng Cypher trên graph cuối:**

```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number AS number, cl.max_years AS max_years ORDER BY number;
```

```text
number  max_years
1       7
2       15
3       20
4       99
5       5
```

- **Nguyên nhân:** Quy tắc gợi ý lấy khoản 1 và khoản nhắc chất trong vụ. Điều 255 khoản 4 quy định chung thân nhưng không nhắc tên chất, nên bị bỏ.
- **Đề xuất sửa/đã làm:** Parse `max_years` (99 biểu diễn chung thân/tử hình để xếp thứ tự) và lấy thêm khoản có mức cao nhất cùng Điều. Phải đọc nguyên văn `penalty` khi trả lời; 99 là sentinel kỹ thuật, không phải “99 năm tù”.

### E3 — Tên chất đồng nghĩa thành nhiều node

- **Hiện tượng:** Bản gợi ý có 17 `Substance`; bản mới còn 14 dù cùng 18 Điều và 20 bài. Tên viết hoa/viết thường và tên đường phố từng chia đôi node.
- **Bằng chứng Cypher chạy trên từng graph sau `--judge`:**

```cypher
MATCH (s:Substance) RETURN s.name AS name ORDER BY toLower(name);
```

```text
Trước (17): Amphetamine, chất ma túy, Cocaine, côca, cần sa, etomidate,
Heroine, Ketamine, ketamine, ma túy, ma túy tổng hợp, MDMA,
methamphetamine, Methamphetamine, thuốc lắc, thuốc phiện, XLR-11
Sau (14): Amphetamine, chất ma túy, Cocaine, côca, cần sa, etomidate,
Heroine, Ketamine, ma túy, ma túy tổng hợp, MDMA,
Methamphetamine, thuốc phiện, XLR-11
```

- **Nguyên nhân:** `MERGE` theo chuỗi `name` thô, trong khi regex luật và LLM tin dùng cách viết khác nhau.
- **Đề xuất sửa/đã làm:** Chuẩn hóa alias rồi `link_entity` tới danh sách chất chuẩn trước khi ghi `INVOLVES`. Tên chung như “ma túy” chưa đủ thông tin để ép vào một chất cụ thể, nên vẫn giữ riêng.

### E1 — Vụ không có cầu Crime

- **Hiện tượng:** Cả graph trước và sau có một vụ không có `CHARGED_WITH`.
- **Bằng chứng:**

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->(:Crime)
RETURN k.name AS name, k.doc_id AS doc_id;
```

```text
Vụ tông cảnh sát giao thông ở An Giang | news-100260926112415229
```

- **Nguyên nhân:** Bài gốc ghi khởi tố Nguyễn Minh Nhân về “chống người thi hành công vụ”; sử dụng ma túy là tình tiết, không phải tội ma túy đang truy tố. Tập luật chỉ có các Điều về ma túy, nên không có `Crime` phù hợp.
- **Đề xuất sửa:** Nếu muốn trả lời vụ này xuyên KB, thêm Điều luật tương ứng và giữ nguyên quy tắc chỉ nối khi tên tội đủ giống. Không ép vụ vào một tội ma túy khác.

### E6 — `charge` ở người còn rỗng

- **Hiện tượng:** Graph cuối còn 12 quan hệ `INVOLVED_IN` có `charge=''`.
- **Bằng chứng:**

```cypher
MATCH (p:Person)-[r:INVOLVED_IN]->(k:Case)
WHERE r.charge = '' RETURN count(r) AS n;
```

```text
n = 12
```

- **Nguyên nhân:** Người liên quan/cán bộ có thể không bị truy tố (rỗng hợp lý); với bị can như Bùi Thị Thanh Thủy hoặc Lê Văn Đông trong vụ Viện Pháp y tâm thần, cần đối chiếu bài để biết LLM bỏ sót hay bài không nêu tội riêng.
- **Đề xuất sửa:** Tách “không áp dụng” khỏi “không rõ”; chỉ bổ sung `charge` sau khi đối chiếu nguồn, không tự suy từ `Case.charges` cho tất cả người.

## 4. Kết luận

Flat đủ cho định nghĩa hoặc một bài báo: Q1–Q2 đều recall 1.00/judge 2, nhanh 1.59 giây/câu trung bình. Với câu phải nối người–vụ–tội–Điều, Graph cho Q3–Q5 recall 1.00/judge 2 trong khi Flat lần lượt 0.00/0, 0.00/0, 0.60/1; cái giá là indexing 0.00936 so với 0.00112 USD và hỏi 0.00087 so với 0.00013 USD/câu. Q6 vẫn judge 1 vì tóm tắt thiếu tên dù graph có các vụ; Graph cần kiểm soát câu trả lời tổng hợp và chất lượng trích xuất, không chỉ thêm cạnh.

## 5. Tự kiểm

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j Browser: `report/img/kg_count.png` (Q-A), `report/img/kg_cross_kb.png` (Q-B), `report/img/kg_my_case.png` (Q-D với **Cái Quang Huy**, khác Lê Minh Thành). Cả ba ảnh có ô truy vấn và kết quả; Q-B/Q-D có Graph cùng Results overview.

## Vấn đề gặp phải

| Vấn đề | Hiện tượng | Nguyên nhân | Cách xử lý / bài học |
| --- | --- | --- | --- |
| Key không được đọc | `--check` báo `[LỖI SETUP-1]` dù `.env` có `OPENAI_API_KEY`. | `bench_kg.py` gọi `load_dotenv(override=False)`; phiên PowerShell đã có `OPENAI_API_KEY` rỗng nên che giá trị trong `.env`. | Xóa biến rỗng bằng `Remove-Item Env:OPENAI_API_KEY -EA SilentlyContinue` trong phiên lệnh. Với `override=False`, biến môi trường hiện có được ưu tiên. |
| `--judge` tưởng đóng băng | Qua `OPENAI_BASE_URL` tương thích OpenAI, chat khoảng 2 giây nhưng mỗi embedding mất 3–22 giây; batch 50 input timeout sau hơn 30 giây. Neo4j vẫn ở 146 node của `--check`. | 176 chunk được embed tuần tự, không in tiến độ; SDK mặc định timeout 600 giây và retry 2 lần nên một request chậm có thể giữ lệnh nhiều phút. Graph 146 node cho thấy quá trình chưa qua bước embedding để dựng graph đầy đủ. | Kiểm tra trạng thái graph để xác định bước đang chậm; theo dõi thời gian từng embedding và đặt timeout phù hợp khi triển khai thực tế. |
| Đổi embedding giữa các lần thử | `gemini-embedding-001` chạy khoảng 0,5 giây/lần nhưng tạo ra kết quả không cùng cấu hình với lần so sánh cuối. | Khác provider embedding có thể đổi top-k và câu trả lời, làm sai lệch phép so sánh ontology. | Bỏ kết quả thử đó, chạy lại **từ đầu** cả `.hint.txt` và bản cuối bằng OpenAI trực tiếp: chat `gpt-4o-mini`, embedding `text-embedding-3-small`. Flat indexing còn 43,7 giây. |
| Biến thiên LLM | Q6 khác nhau giữa các lần chạy cùng code. | Trích xuất và sinh câu trả lời bằng LLM không hoàn toàn ổn định. | Chỉ so sánh trước/sau khi cùng provider; dùng Cypher và kết quả graph làm bằng chứng chính, không dựa riêng vào điểm judge. |
