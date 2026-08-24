# Bài 4: Giám Sát Chi Phí & Phân Tích Latency

## 1. Cơ chế Token & Cost Tracking của Langfuse

Langfuse theo dõi chi phí chủ yếu trên các observation kiểu **Generation** và **Embedding**. Với mỗi lần gọi LLM, hệ thống có thể lưu:

* `input`: số token đầu vào.
* `output`: số token đầu ra.
* `total`: tổng số token.
* Các loại token khác như cached tokens hoặc reasoning tokens nếu provider cung cấp.

Chi phí có thể đến từ hai nguồn:

### Cách 1: Tự động thu thập

Khi integration/SDK nhận được token usage từ provider, Langfuse sẽ tự ghi nhận usage.

Ví dụ:

```text
Input tokens: 1,000
Output tokens: 500
Total tokens: 1,500
```

Nếu model đã có bảng giá phù hợp, Langfuse tiếp tục tính chi phí:

```text
Cost =
(input_tokens × input_price)
+
(output_tokens × output_price)
```

Langfuse lưu `usage_details` và `cost_details` theo từng Generation. Nếu vừa có dữ liệu được gửi trực tiếp từ provider vừa có dữ liệu Langfuse tự suy luận, dữ liệu **ingested** được ưu tiên.

> Lưu ý: các usage bucket cần không chồng chéo nhau. Nếu cùng một token bị tính vào nhiều bucket, chi phí có thể bị tính trùng.

---

## 2. Custom Model Prices

Trong Langfuse Dashboard, vào:

**Project Settings → Models**

Tại đây có thể:

1. Thêm Custom Model.
2. Khai báo `match_pattern` để Langfuse nhận diện tên model.
3. Khai báo tokenizer nếu model được hỗ trợ.
4. Thiết lập giá cho từng loại token:

    * Input price.
    * Output price.
    * Cached input price nếu có.
5. Lưu Model Definition.

Langfuse ánh xạ model được gửi từ ứng dụng với Model Definition thông qua `model` và `match_pattern` dạng regex. Custom Model do người dùng tạo được ưu tiên hơn model definition có sẵn của Langfuse.

Ví dụ có thể tạo hai Model Definition:

### Gemini-2.5-Flash

```text
Model:
gemini-2.5-flash

Match Pattern:
(?i)^gemini-2\.5-flash$

Prices:
input  = giá input/token theo bảng giá hiện hành
output = giá output/token theo bảng giá hiện hành
```

### DeepSeek-V3

```text
Model:
deepseek-v3

Match Pattern:
(?i)^deepseek-v3$

Prices:
input  = giá input/token theo bảng giá hiện hành
output = giá output/token theo bảng giá hiện hành
```

Sau đó Langfuse có thể thống kê:

```text
Cost theo Model

Gemini-2.5-Flash  → Tổng chi phí / số request
DeepSeek-V3       → Tổng chi phí / số request
```

Công thức chi phí trung bình mỗi lượt gọi:

```text
Average Cost per AI Call
= Total AI Cost / Total Number of Generations
```

Do đó ban giám đốc có thể so sánh trực tiếp chi phí trung bình giữa hai model.

Langfuse cũng hỗ trợ **Pricing Tiers**, tức một model có thể có nhiều mức giá dựa trên điều kiện như số lượng input token hoặc model parameters.

---

## 3. Cách phân tích Latency của RAG

Một request RAG nên được chia thành các bước riêng biệt trong một Trace:

```text
RAG Trace
│
├── Retrieval
│   ├── Embedding Query
│   └── Vector DB Search
│
└── Generation
    └── LLM Call
```

Langfuse lưu thời gian bắt đầu và kết thúc của Trace/Observation để tính Duration. Vì vậy cần instrument riêng từng bước thay vì chỉ theo dõi tổng thời gian của cả request. Langfuse hỗ trợ theo dõi latency trên trace và observation, đồng thời dashboard có thể phân tích latency theo model, trace, feature hoặc các dimension khác.

### Trường hợp 1: Retrieval là bottleneck

Ví dụ:

```text
Vector DB Retrieval: 2,800 ms
LLM Generation:       900 ms
Total:              3,700 ms
```

Kết luận:

> Bottleneck nằm ở Retrieval/Vector DB.

Cần kiểm tra:

* HNSW hoặc index của pgvector.
* Kích thước dữ liệu.
* Embedding query.
* Similarity search.
* Network latency đến database.
* `topK` quá lớn.

### Trường hợp 2: Generation là bottleneck

Ví dụ:

```text
Vector DB Retrieval:   200 ms
LLM Generation:      4,500 ms
Total:               4,700 ms
```

Kết luận:

> Bottleneck nằm ở LLM Generation.

Cần kiểm tra:

* Model đang sử dụng.
* Số lượng input token.
* Context RAG quá dài.
* `maxTokens` quá cao.
* Network latency tới LLM provider.

### Cách đọc Dashboard

Vào:

```text
Dashboards → Latency Dashboard
```

Có thể theo dõi:

* Average Latency.
* P50.
* P95.
* P99.
* Latency theo model.
* Latency theo trace.
* Latency theo từng observation.

Ví dụ nên tạo các observation riêng:

```text
rag-retrieval
rag-generation
```

Sau đó tạo biểu đồ:

```text
Metric: Latency
Dimension: Observation Name
```

Nếu:

```text
P95(rag-retrieval) > P95(rag-generation)
```

thì Retrieval là bottleneck.

Ngược lại:

```text
P95(rag-generation) > P95(rag-retrieval)
```

thì LLM Generation là bottleneck.

Langfuse có sẵn các curated dashboard cho Latency và Cost, đồng thời cho phép tạo dashboard tùy chỉnh để theo dõi P95/P99, model latency và tool/retrieval latency.

---

# 4. Java gửi Token Usage thủ công sang Langfuse

Với custom model hoặc self-hosted model không hỗ trợ tự động đếm token, backend có thể lấy token usage từ provider hoặc tự tính rồi gửi thủ công sang Langfuse.

Ví dụ dưới đây minh họa ý tưởng: tạo một Generation, sau khi gọi model thì cập nhật `input`, `output` và `total`.

```java
package org.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
@RequiredArgsConstructor
public class AiMonitoringService {

    private final ChatClient chatClient;
    private final LangfuseService langfuseService;
    private final TokenCounter tokenCounter;

    public String callCustomModel(String userInput) {

        // 1. Tạo Generation trên Langfuse
        String generationId = langfuseService.startGeneration(
                "custom-llm-generation",
                "deepseek-v3",
                userInput
        );

        long startTime = System.currentTimeMillis();

        try {
            // 2. Gọi LLM
            String response = chatClient.prompt()
                    .user(userInput)
                    .call()
                    .content();

            // 3. Đếm token thủ công
            int inputTokens = tokenCounter.count(userInput);
            int outputTokens = tokenCounter.count(response);

            int totalTokens = inputTokens + outputTokens;

            // 4. Gửi usage sang Langfuse
            langfuseService.updateGeneration(
                    generationId,
                    response,
                    Map.of(
                            "input", inputTokens,
                            "output", outputTokens,
                            "total", totalTokens
                    )
            );

            return response;

        } finally {

            // 5. Kết thúc Generation
            long durationMs = System.currentTimeMillis() - startTime;

            langfuseService.endGeneration(
                    generationId,
                    durationMs
            );
        }
    }
}
```

Ví dụ service gửi dữ liệu sang Langfuse:

```java
package org.example.service;

import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class LangfuseService {

    public String startGeneration(
            String name,
            String model,
            String input
    ) {

        /*
         * Tạo Generation bằng Langfuse SDK.
         *
         * generation:
         * - name = custom-llm-generation
         * - model = deepseek-v3
         * - input = userInput
         *
         * return generationId;
         */

        return "generation-id";
    }

    public void updateGeneration(
            String generationId,
            String output,
            Map<String, Integer> usageDetails
    ) {

        /*
         * Cập nhật Generation:
         *
         * output = response
         *
         * usage_details = {
         *   "input": 100,
         *   "output": 50,
         *   "total": 150
         * }
         *
         * Langfuse sẽ sử dụng model = deepseek-v3
         * để match với Custom Model Price.
         *
         * Sau đó:
         * cost = input * input_price
         *      + output * output_price
         */
    }

    public void endGeneration(
            String generationId,
            long durationMs
    ) {

        /*
         * Kết thúc observation.
         *
         * Duration sẽ được Langfuse dùng
         * để phân tích latency.
         */
    }
}
```

## 5. Ví dụ đầy đủ luồng RAG nên giám sát

```text
User Request
     │
     ▼
Langfuse Trace: rag-query
     │
     ├── Observation: embedding-query
     │      └── Duration: 120 ms
     │
     ├── Observation: vector-retrieval
     │      └── Duration: 850 ms
     │
     └── Generation: llm-generation
            ├── Model: Gemini-2.5-Flash
            ├── Input Tokens: 1,200
            ├── Output Tokens: 300
            ├── Cost: tự động tính
            └── Duration: 1,400 ms
```

Từ dữ liệu này:

```text
Retrieval Latency = 850 ms
Generation Latency = 1,400 ms
```

Có thể kết luận:

> Trong trường hợp này LLM Generation là bước có latency cao nhất, nhưng nếu muốn đánh giá ổn định trong production cần ưu tiên xem P95/P99 thay vì chỉ nhìn một request đơn lẻ.

---

## 6. Kết luận

Langfuse hỗ trợ quản lý chi phí theo hai cách:

1. **Tự động:** lấy usage từ integration/provider và suy luận chi phí từ Model Price List.
2. **Thủ công:** ứng dụng gửi `usage_details` hoặc trực tiếp gửi `cost_details`.

Để RikkeiPay so sánh Gemini-2.5-Flash và DeepSeek-V3, cần đảm bảo:

* Tên model trong Generation được chuẩn hóa.
* Model được match đúng với Model Definition.
* Có token usage chính xác.
* Thiết lập đúng Custom Model Prices.
* So sánh Average Cost và P95/P99 Latency.

Với RAG, nên tách riêng **Retrieval** và **Generation** thành các observation để Langfuse xác định chính xác bottleneck, thay vì chỉ nhìn tổng thời gian của cả request.
