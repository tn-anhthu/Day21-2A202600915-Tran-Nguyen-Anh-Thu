# Day 21 Lab — Thiết kế Test Inputs cho AI Evals

**Họ và tên:** Trần Nguyễn Anh Thư

**Mã học viên:** 2A202600915

**Tên thư mục nộp bài:** Day21-2A202600915-Tran Nguyen Anh Thu

---

# PHẦN CÁ NHÂN

## Bài 1 — Use Case & Unit of AI Work

### 1.1 Bảng tổng quan

| Thành phần | Câu trả lời |
|---|---|
| Use case từ Day 18/19 | Academic Research Assistant — Claim Verification & Routing (ClaimVerifier feature) |
| Persona chính | Researcher đang viết literature review |
| Unit of AI Work | Một câu claim đơn lẻ → agent phân loại verification status + routing decision |
| Input user đưa vào | Câu claim (text) + source đã được retrieved từ Semantic Scholar / arXiv |
| Output agent cần tạo | Verification status (Supported / Partially Supported / Unsupported / Case C) + Action (Act auto-keep / Ask user / Act auto-delete / Priority Review) |
| Agent được phép làm gì? | Auto-keep claim Supported với nguồn mạnh (snippet/arxiv); Auto-delete + log claim Unsupported; Flag Priority Review cho contrasting intent |
| Agent không được phép làm gì? | Báo Supported khi source chỉ là abstract-only; Xóa dữ liệu gốc không có undo; Tự quyết khi claim không rõ ràng |

---

## Bài 2 — Quality Question

| Câu hỏi | Câu trả lời |
|---|---|
| **Quality question chính** | Agent có phân loại đúng verification status của claim và route đến đúng action, để không vi phạm tính học thuật và không gây nhiễu với user bằng cách hỏi quá nhiều về claim không cần thiết không? |
| Vì sao câu hỏi này quan trọng với user? | User là researcher — một claim sai được giữ lại ảnh hưởng trực tiếp đến tính học thuẩ của bài viết. Ngược lại, agent Ask quá nhiều sẽ làm feature vô dụng, không khác nào researcher tự làm. |
| Nếu agent fail ở đây, hậu quả là gì? | Claim không có nguồn vững bị giữ lại (risk cao nhất) hoặc user mất nội dung vì agent auto-delete nhầm |
| Behavior nào là bắt buộc? | Abstract-only KHÔNG BAO GIỜ được báo Supported; mọi auto-delete phải có log + undo; contrasting intent phải trigger Priority Review bất kể source quality |
| Behavior nào bị cấm? | Auto-delete không có undo; báo Supported khi thiếu evidence, chỉ có abstract; Ask hết mọi claim kể cả case rõ ràng |

---

## Bài 3 — User Input Grid

### 3.1 Bảng dimensions

| Dimension | Values | Vì sao làm agent phải đổi behavior? |
|---|---|---|
| **Source tier** | snippet / arxiv fulltext / abstract-only / no source found | Tier xác định độ tin cậy — abstract-only không bao giờ được Supported theo thiết kế 3-tier |
| **Claim-source alignment** | exact match / overgeneralized / contradicted / multi-source conflict | Alignment quyết định trực tiếp verification status (Supported / Partial / Unsupported) |
| **Claim intent signal** | neutral claim / contrasting intent ("however", "limitations") / ambiguous scope | Contrasting intent phải trigger Priority Review dù source quality cao — đây là dimension dễ làm agent nhầm nhất |
| **Context completeness** | claim đủ rõ / claim ambiguous / claim reference nhiều nguồn cùng lúc | Ảnh hưởng khả năng agent phân loại đúng hay cần hỏi thêm |

### 3.2 Kiểm tra dimension

| Câu hỏi kiểm tra | Source tier | Claim-source alignment | Claim intent signal | Context completeness |
|---|---|---|---|---|
| Nếu đổi value, expected behavior có đổi không? | Có | Có | Có | Có |
| Dimension này gắn với risk hoặc user outcome không? | Tính học thuật | Trực tiếp | Ảnh hưởng nội dung nghiên cứu | Ảnh hưởng routing |
| Giúp tìm failure mà happy path không thấy không? | abstract-only edge case | multi-source conflict | contrasting = failure mode đặc biệt | không rõ ràng = tricky case |

---

## Bài 4 — Meaningful Combinations (≥10)

| Combination ID | Dimension values | Expected behavior | Vì sao đáng test? | Loại |
|---|---|---|---|---|
| C01 | abstract-only + exact match + neutral | Case C → Ask (KHÔNG được Supported) | Failure mode nguy hiểm nhất: agent báo Supported dù chỉ có abstract | high-risk |
| C02 | snippet + exact match + neutral + claim rõ | Act auto-keep (Supported) | Happy path — agent xử lý đúng và không làm phiền user | representative |
| C03 | arxiv fulltext + overgeneralized + neutral + claim rõ | Partially Supported → Ask | Agent có nhận ra scope mismatch không? | representative |
| C04 | arxiv fulltext + contradicted + neutral + claim rõ | Act auto-delete + log + undo | Agent có log đúng + cho phép undo không? | high-risk |
| C05 | no source found + neutral + claim rõ | Ask / escalate review | Fallback khi pipeline không có data — agent không được tự quyết | representative |
| C06 | snippet + exact match + contrasting intent + claim rõ | Priority Review (dù source tốt) | Source tốt nhưng intent signal ghi đè — agent có detect "however/limitations" không? | high-risk |
| C07 | abstract-only + contradicted + contrasting intent | Case C + Priority Review | Double ambiguity — agent phải handle 2 signals cùng lúc | challenge |
| C08 | arxiv fulltext + multi-source conflict + neutral + claim rõ | Ask — không tự quyết | Agent có chọn source thuận tiện để auto-keep không? | high-risk |
| C09 | snippet + overgeneralized + ambiguous scope | Partially Supported → Ask | Claim mơ hồ dù source rõ — agent có nhận ra scope issue không? | challenge |
| C10 | abstract-only + contrasting intent + claim rõ | Priority Review (vẫn Ask) | Intent override tier — không phải vì source tệ mà vì intent cần review | challenge |
| C11 | snippet + exact match + ambiguous (multi-reference) | Partially Supported hoặc Ask | Claim ghép 2 ý từ nhiều nguồn — agent xử lý multi-claim thế nào? | challenge |
| C12 | arxiv fulltext + text looks contradicted but is nuance + neutral | KHÔNG auto-delete — Ask | "Nghe có vẻ sai nhưng thực ra đúng" — agent có trigger auto-delete oan không? | high-risk |

---

## Bài 5 — AI-Generated Inputs

### 5.1 Prompt đã dùng

```
Bạn là researcher đang viết literature review, nhắn cho một AI assistant.

Tôi đang thiết kế test inputs cho use case:
AI Academic Research Assistant — Claim Verification & Routing.
Agent nhận một câu claim học thuật (đơn lẻ) + thông tin về source đã retrieved, và phải phân loại verification status + quyết định routing action.

Quality question:
Agent có phân loại đúng verification status và route đến đúng action, để không vi phạm academic integrity và không overload user với review không cần thiết không?

Tôi đã chọn các combinations sau. Nhiệm vụ của bạn là viết lại mỗi combination thành 2 user inputs tự nhiên (câu claim như researcher thật viết trong bài).

Yêu cầu:
- Không tự thêm combination mới.
- Không thay đổi intent, source tier hoặc alignment đã cho.
- Viết như claim thật trong bài viết học thuật — có thể ngắn, dài, mơ hồ, hoặc dùng hedging language.
- Có cả claim rõ ràng và claim thiếu context.
- Không giải thích cách agent nên trả lời.
- Output dạng bảng gồm: combination_id, user_input (câu claim), style, notes.

Combinations:
[dán bảng combinations C01–C12 vào đây]
```

### 5.2 Output thô của AI

| combination_id | user_input | style | notes |
|---|---|---|---|
| C01 | "Smith et al. (2021) report that transformer-based encoders outperform LSTM baselines on long-document summarization." | rõ, ngắn | abstract-only + exact match + neutral — claim khớp y abstract, nhưng chưa có fulltext để xác minh chi tiết |
| C01 | "Recent findings indicate that low-rank adaptation reduces GPU memory usage by over 60% during fine-tuning (Chen et al., 2022)." | rõ, có số liệu cụ thể | trap case: match chính xác abstract nhưng vẫn chỉ là abstract-level evidence |
| C02 | "According to the dataset documentation, the corpus contains 1.2 million annotated sentence pairs across eight languages." | rõ, factual | snippet trùng khớp hoàn toàn, không có tín hiệu mơ hồ — happy path |
| C02 | "The original paper states that the proposed algorithm runs in O(n log n) time in the worst case." | rõ, ngắn | snippet + exact match, claim đơn giản dễ verify |
| C03 | "Our results, consistent with Lee et al. (2023), show that diffusion models are superior to GANs for all image generation tasks." | rõ nhưng overclaim | fulltext có thể chỉ test trên 1 domain (vd. faces), claim lại generalize "all" |
| C03 | "Reinforcement learning from human feedback universally improves alignment across all language model architectures (Ouyang et al., 2022)." | rõ, overgeneralize bằng từ "universally" | scope mismatch giữa fulltext thực tế và phát ngôn "all/universally" |
| C04 | "As demonstrated by Park et al. (2020), increasing model size always leads to better generalization performance." | rõ, dùng "always" | fulltext thực ra có chỉ ra giới hạn/phản ví dụ — claim bị contradicted |
| C04 | "The study confirms that batch normalization eliminates the need for careful weight initialization in deep networks." | rõ, tuyên bố mạnh | claim trái với nội dung fulltext gốc — test auto-delete + log + undo |
| C05 | "It has been established that quantum annealing provides an exponential speedup for combinatorial optimization problems." | rõ nhưng không có citation cụ thể | no source found — claim nghe "chắc chắn" nhưng pipeline không retrieve được gì |
| C05 | "Earlier research has shown that synthetic data alone can fully replace real-world data in NLP training pipelines." | rõ, dùng "has shown" mơ hồ về nguồn | không có source → fallback, agent không được tự quyết |
| C06 | "Although the benchmark report confirms a 15% accuracy improvement, this result does not hold under domain shift conditions." | rõ, có "although" | snippet match tốt nhưng có "however/limitations" signal → contrasting intent |
| C06 | "The documentation states the system processes 10,000 requests/second; however, this figure excludes network latency overhead." | rõ, hai mệnh đề đối lập | source tốt (snippet exact match) nhưng intent ghi đè → Priority Review |
| C07 | "While the abstract claims state-of-the-art results, several follow-up studies suggest the reported gains may not generalize." | hedging, dài | abstract-only + contradicted + contrasting intent — double ambiguity |
| C07 | "It is often assumed that larger context windows improve reasoning, though this might not actually be the case based on the abstract." | hedging mạnh, dùng "might not actually be" | claim thiếu context rõ, vừa contradicted vừa contrasting intent |
| C08 | "Across the literature, there is consensus that pretraining on multilingual corpora improves zero-shot cross-lingual transfer." | rõ, dùng từ "consensus" | claim giả định đồng thuận nhưng multi-source thực ra conflict — agent không được chọn nguồn thuận tiện |
| C08 | "Multiple studies agree that knowledge distillation preserves over 95% of teacher model performance in all settings." | rõ, số liệu cụ thể | "all settings" + multi-source conflict — test agent có tự average/chọn nguồn tiện không |
| C09 | "These methods tend to work well in practice." | mơ hồ, rất ngắn | claim thiếu scope rõ (work well — ở task/domain nào?), source là snippet overgeneralized |
| C09 | "It has been shown that contrastive learning generally improves representation quality across domains." | mơ hồ, dùng "generally", "domains" không rõ là gì | ambiguous scope — agent cần nhận ra claim mơ hồ dù snippet tồn tại |
| C10 | "The abstract reports a performance gain, but the authors caution that the experimental setup may not reflect real-world deployment." | rõ, có hedge từ chính tác giả | abstract-only + contrasting intent — Priority Review vì intent, không phải vì source tệ |
| C10 | "Although the study claims robustness across noise levels, this finding has not been independently verified." | rõ, "although...has not been verified" | intent override tier — claim rõ nhưng có caveat ngữ nghĩa |
| C11 | "Building on both Zhang et al. (2021) and Kim et al. (2022), our approach combines attention pruning with quantization for real-time inference." | rõ về câu chữ nhưng ghép 2 nguồn | multi-reference ambiguous — claim hợp nhất ý từ 2 paper khác nhau |
| C11 | "As established in prior work, transformers benefit from both relative positional encoding and sparse attention for long sequences." | rõ nhưng "prior work" không chỉ rõ nguồn nào | multi-claim ghép — agent cần xử lý từng phần claim riêng so với từng source |
| C12 | "The model fails to improve performance on the held-out test set." | ngắn, nghe negative | full text thực ra có qualifier (vd: chỉ fail khi thiếu warm-up scheduling) — surface contradiction nhưng là nuance |
| C12 | "Performance does not increase with additional training data." | rõ, tuyên bố tuyệt đối nhưng thiếu ngữ cảnh | fulltext có thể là "...beyond 50K examples, performance plateaus" — agent dễ auto-delete oan nếu không đọc kỹ context |

### 5.3 Danh sách inputs đã lọc (≥20)

Tổng 24 inputs từ AI output — tất cả passed filter sau human review.

**Tiêu chí filter đã dùng:**
- Giữ: claim đủ cụ thể để grader xác định expected behavior
- Giữ: style trong cùng combination phải khác nhau (1 ngắn / 1 dài, hoặc explicit / hedging)
- Loại: claim quá generic không gắn được với combination cụ thể
- Không có row nào bị loại — 24/24 inputs chuyển vào Dataset v0

→ Xem Dataset v0 bên dưới (A01–A24).

---

## Bài 6 — Scenario Dataset v0 (Individual)

> Schema: scenario_id / owner / use_case / quality_question / combination_id / dimension_values / user_input / style / expected_behavior / why_included / set_type

| scenario_id | combination_id | dimension_values | user_input | style | expected_behavior | why_included | set_type |
|---|---|---|---|---|---|---|---|
| A01 | C01 | abstract-only + exact match + neutral | "Smith et al. (2021) report that transformer-based encoders outperform LSTM baselines on long-document summarization." | rõ, ngắn | Case C → Ask, KHÔNG Supported | Failure mode nguy hiểm nhất — agent có thể báo Supported dù chỉ có abstract | high-risk |
| A02 | C01 | abstract-only + exact match + neutral | "Recent findings indicate that low-rank adaptation reduces GPU memory usage by over 60% during fine-tuning (Chen et al., 2022)." | rõ, có số liệu cụ thể | Case C → Ask, KHÔNG Supported | Trap case — số liệu cụ thể dễ tạo cảm giác chắc chắn nhưng source vẫn chỉ là abstract | high-risk |
| A03 | C02 | snippet + exact match + neutral | "According to the dataset documentation, the corpus contains 1.2 million annotated sentence pairs across eight languages." | rõ, factual | Act auto-keep (Supported) | Happy path — snippet khớp hoàn toàn, không mơ hồ, không làm phiền user | representative |
| A04 | C02 | snippet + exact match + neutral | "The original paper states that the proposed algorithm runs in O(n log n) time in the worst case." | rõ, ngắn, kỹ thuật | Act auto-keep (Supported) | Happy path biến thể — claim đơn giản, snippet exact match | representative |
| A05 | C03 | arxiv fulltext + overgeneralized + neutral | "Our results, consistent with Lee et al. (2023), show that diffusion models are superior to GANs for all image generation tasks." | rõ nhưng overclaim với "all" | Partially Supported → Ask | Scope mismatch — fulltext chỉ test 1 domain nhưng claim generalize "all tasks" | representative |
| A06 | C03 | arxiv fulltext + overgeneralized + neutral | "Reinforcement learning from human feedback universally improves alignment across all language model architectures (Ouyang et al., 2022)." | rõ, overgeneralize bằng "universally" + "all" | Partially Supported → Ask | "universally/all architectures" là signal scope mismatch rõ — agent phải nhận ra không nên auto-keep | representative |
| A07 | C04 | arxiv fulltext + contradicted + neutral | "As demonstrated by Park et al. (2020), increasing model size always leads to better generalization performance." | rõ, dùng "always" | Act auto-delete + log + undo | "always" mâu thuẫn trực tiếp với fulltext có counterexample — test auto-delete + log đúng không | high-risk |
| A08 | C04 | arxiv fulltext + contradicted + neutral | "The study confirms that batch normalization eliminates the need for careful weight initialization in deep networks." | rõ, tuyên bố tuyệt đối | Act auto-delete + log + undo | Claim trái với fulltext gốc — test agent có log đúng và cho phép undo không | high-risk |
| A09 | C05 | no source found + neutral | "It has been established that quantum annealing provides an exponential speedup for combinatorial optimization problems." | rõ, nghe authoritative nhưng không có citation | Ask / escalate review | Claim nghe chắc chắn nhưng pipeline không retrieve được source — agent không được tự quyết | representative |
| A10 | C05 | no source found + neutral | "Earlier research has shown that synthetic data alone can fully replace real-world data in NLP training pipelines." | rõ, "has shown" mơ hồ về nguồn | Ask / escalate review | No source fallback — agent phải Ask dù claim dùng confident language | representative |
| A11 | C06 | snippet + exact match + contrasting intent | "Although the benchmark report confirms a 15% accuracy improvement, this result does not hold under domain shift conditions." | rõ, có "although" + limitation | Priority Review | Source tốt (snippet exact match) nhưng contrasting intent ghi đè — test agent có detect limitations signal không | high-risk |
| A12 | C06 | snippet + exact match + contrasting intent | "The documentation states the system processes 10,000 requests/second; however, this figure excludes network latency overhead." | rõ, hai mệnh đề đối lập với "however" explicit | Priority Review | "however" explicit — test agent detect contrasting intent kể cả khi source quality cao | high-risk |
| A13 | C07 | abstract-only + contradicted + contrasting intent | "While the abstract claims state-of-the-art results, several follow-up studies suggest the reported gains may not generalize." | hedging, dài | Case C + Priority Review | Double ambiguity — abstract-only + contradicted + contrasting intent cùng lúc | challenge |
| A14 | C07 | abstract-only + contradicted + contrasting intent | "It is often assumed that larger context windows improve reasoning, though this might not actually be the case based on the abstract." | hedging mạnh, "might not actually be" | Case C + Priority Review | Claim thiếu context rõ, vừa contradicted vừa contrasting intent — agent phải handle 2 signals cùng lúc | challenge |
| A15 | C08 | arxiv fulltext + multi-source conflict + neutral | "Across the literature, there is consensus that pretraining on multilingual corpora improves zero-shot cross-lingual transfer." | rõ, dùng "consensus" | Ask — không tự quyết | "consensus" che giấu multi-source conflict — agent không được chọn source thuận tiện để auto-keep | high-risk |
| A16 | C08 | arxiv fulltext + multi-source conflict + neutral | "Multiple studies agree that knowledge distillation preserves over 95% of teacher model performance in all settings." | rõ, số liệu cụ thể, "all settings" | Ask — không tự quyết | "all settings" + multi-source conflict — test agent có average hoặc chọn nguồn tiện nhất không | high-risk |
| A17 | C09 | snippet + overgeneralized + ambiguous scope | "These methods tend to work well in practice." | mơ hồ, rất ngắn | Partially Supported → Ask | Claim thiếu scope hoàn toàn (methods nào? practice ở đâu?) — agent phải nhận ra ambiguity dù snippet tồn tại | challenge |
| A18 | C09 | snippet + overgeneralized + ambiguous scope | "It has been shown that contrastive learning generally improves representation quality across domains." | mơ hồ, "generally" + "domains" không rõ | Partially Supported → Ask | Ambiguous scope dù snippet overgeneralized tồn tại — agent không được auto-keep | challenge |
| A19 | C10 | abstract-only + contrasting intent + claim rõ | "The abstract reports a performance gain, but the authors caution that the experimental setup may not reflect real-world deployment." | rõ, hedge từ chính tác giả trong abstract | Priority Review | Intent override tier — source là abstract-only nhưng contrasting intent từ chính tác giả phải trigger Priority Review | challenge |
| A20 | C10 | abstract-only + contrasting intent + claim rõ | "Although the study claims robustness across noise levels, this finding has not been independently verified." | rõ, "although...has not been verified" | Priority Review | Caveat ngữ nghĩa ghi đè tier — agent không được bỏ qua "has not been verified" | challenge |
| A21 | C11 | snippet + exact match + ambiguous multi-reference | "Building on both Zhang et al. (2021) and Kim et al. (2022), our approach combines attention pruning with quantization for real-time inference." | rõ câu chữ nhưng ghép 2 nguồn | Partially Supported hoặc Ask | Multi-reference — claim hợp nhất ý từ 2 paper khác nhau, agent cần xử lý từng phần | challenge |
| A22 | C11 | snippet + exact match + ambiguous multi-reference | "As established in prior work, transformers benefit from both relative positional encoding and sparse attention for long sequences." | "prior work" không chỉ rõ nguồn nào | Partially Supported hoặc Ask | Multi-claim ghép mơ hồ — "prior work" không rõ nguồn nào, agent cần Ask thay vì tự match | challenge |
| A23 | C12 | arxiv fulltext + nuance (looks contradicted) + neutral | "The model fails to improve performance on the held-out test set." | ngắn, nghe negative | KHÔNG auto-delete → Ask | Surface contradiction nhưng là nuance — fulltext có qualifier (chỉ fail khi thiếu warm-up scheduling) | high-risk |
| A24 | C12 | arxiv fulltext + nuance (looks contradicted) + neutral | "Performance does not increase with additional training data." | rõ, tuyên bố tuyệt đối thiếu ngữ cảnh | KHÔNG auto-delete → Ask | False positive auto-delete risk — fulltext thực ra nói "plateaus after 50K examples", không phải never | high-risk |

**owner:** Trần Nguyễn Anh Thư

**use_case:** AI Academic Research Assistant — Claim Verifier

**quality_question:** Agent có phân loại đúng verification status và route đến đúng action, để không vi phạm tính academic và không overload user không?

### 6.1 Coverage Note cá nhân

*(Viết 5–7 dòng)*

- **Cover tốt:** Cover tốt 3 failure modes nguy hiểm nhất: abstract-only báo Supported (A01–A02), contrasting intent bị bỏ qua (A11–A12), và auto-delete oan claim hợp lệ (A23–A24). Happy path cũng có đủ 2 rows để baseline.
- **Chưa cover:** Source là preprint chưa peer-review, arxiv hiện tại được xem như fulltext, nhưng preprint chưa verified là grey area chưa được model. Claim rất dài (nhiều mệnh đề) vì tất cả inputs hiện tại đều 1–2 câu, chưa test paragraph-length claim
- **Combination cố tình chưa chọn:** C01 + mâu thuẫn (abstract-only + contradicted + neutral): Không chọn vì theo thiết kế 3-tier, abstract-only luôn → Case C → Ask bất kể alignment. Contradicted signal trên abstract không thay đổi routing decision, thêm combination này sẽ tạo row trùng expected behavior với A01–A02 mà không test thêm failure mode mới. Tương tự A03 + contrasting intent: abstract-only sẽ luôn → Case C → Ask bất kể contradicting intent, không thêm giá trị mới.
- **Input high-risk nhất:** A01 (abstract-only báo Supported) và A11 vì agent cần đọc semantic của claim chứ không chỉ check source quality.
- **Boundary case khó nhất:** C12 — claim nghe có vẻ contradicted nhưng thực ra là nuance hợp lệ

---

# PHẦN NHÓM (Solo — tự review)


---



---

## Checklist trước khi nộp

### Cá nhân
- [ ] Có use case từ Day 18/19
- [ ] Có Unit of AI Work
- [ ] Có một quality question rõ
- [ ] Có ít nhất 3 dimensions và values
- [ ] Có tối thiểu 10 scenarios/combinations (đang có: 12)
- [ ] Có prompt đã dùng để generate inputs
- [ ] Có tối thiểu 20 user inputs sau khi lọc
- [ ] Có Scenario Dataset v0 cá nhân
- [ ] Có coverage note cá nhân

### Nhóm (Solo)
- [ ] Có bảng chuẩn hóa dimensions
- [ ] Có coverage matrix
- [ ] Có danh sách merge/dedup decisions
- [ ] Có Scenario Dataset v1 gồm ít nhất 30 rows
- [ ] Có known gaps
- [ ] Có handoff note cho bước chạy agent và đọc trace sau này
