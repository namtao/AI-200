# Module Assessment — Build Serverless AI Backends with Azure Functions

> Khoá: AI-200 · Azure Functions — Build serverless AI backends with Azure Functions

---

## Câu 1

**Your AI inference function experiences cold start latency of several seconds because it loads large ML libraries and initializes connections to Azure AI services. You need to reduce latency for the first request while keeping costs low during idle periods. Which approach should you use?**

- Use the Flex Consumption plan with always-ready instances configured for the function ✅
- Use the Consumption plan with increased function timeout settings
- Use the Premium plan with pre-warmed workers disabled

**Đáp án: Flex Consumption với always-ready instances**

Flex Consumption cho phép cấu hình **always-ready instances cho specific functions**. Khi function có always-ready = 1, một instance luôn warm → không cold start. Các functions khác trong app vẫn scale to zero → tiết kiệm cost.

Cân bằng: trả cho always-ready baseline + không trả khi idle functions are at zero.

Tại sao các đáp án còn lại sai:
- **Consumption plan:** Đây là legacy Linux plan sẽ **retire Sept 2028**. Hơn nữa, Consumption không có always-ready feature — không giải quyết cold start. Timeout settings không liên quan đến cold start.
- **Premium với pre-warmed disabled:** Premium mặc định có pre-warmed workers. Nếu disable → mất toàn bộ premium plan benefit. Premium với pre-warmed workers thì works nhưng không phải "keeps costs low during idle" — Premium luôn run ít nhất 1 instance → higher baseline cost hơn Flex Consumption.

---

## Câu 2

**You're setting up a local development environment for Azure Functions. Your function app uses a Service Bus queue trigger, but you notice the trigger doesn't fire when you run the function locally. What is the most likely cause?**

- The AzureWebJobsStorage setting in local.settings.json isn't configured with a valid storage connection ✅
- The function app needs to be deployed to Azure before Service Bus triggers can be tested
- Visual Studio Code doesn't support debugging Service Bus-triggered functions

**Đáp án: AzureWebJobsStorage không được configure**

Azure Functions runtime yêu cầu storage connection (`AzureWebJobsStorage`) để coordinate trigger processing, track state, và manage locks. **Non-HTTP triggers như Service Bus queue trigger không hoạt động** nếu thiếu valid storage connection.

```json
{
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",  // Azurite
        "ServiceBusConnection": "Endpoint=sb://...",
        "FUNCTIONS_WORKER_RUNTIME": "python"
    }
}
```

Tại sao các đáp án còn lại sai:
- **Cần deploy lên Azure:** Sai hoàn toàn. Local development là mục đích của Core Tools — triggers bao gồm Service Bus có thể test locally với connection string thực.
- **VS Code không support:** VS Code với Azure Functions extension hỗ trợ đầy đủ debugging cho tất cả trigger types. Đây không phải limitation.

---

## Câu 3

**Your function processes documents asynchronously. An HTTP endpoint accepts requests, writes messages to a Service Bus queue, and a separate queue-triggered function performs the processing. You want to configure the queue trigger to process one message at a time per instance to maximize CPU and memory available for each document. Which configuration should you use?**

- Set `maxConcurrentCalls` to 1 in the `serviceBus` section of `host.json` ✅
- Set `batchSize` to 1 in the `serviceBus` section of `host.json`
- Set `maxDequeueCount` to 1 on the Service Bus queue resource

**Đáp án: `maxConcurrentCalls: 1` trong host.json**

```json
{
    "version": "2.0",
    "extensions": {
        "serviceBus": {
            "maxConcurrentCalls": 1
        }
    }
}
```

`maxConcurrentCalls` = số messages mỗi **instance** xử lý đồng thời. Setting = 1 → mỗi instance chỉ process 1 message tại một thời điểm → full instance resources (CPU + memory) dành cho từng document.

Tại sao các đáp án còn lại sai:
- **`batchSize` to 1:** `batchSize` là property của **Storage Queue trigger** (Azure Storage Queues), không phải Service Bus trigger. Service Bus trigger dùng `maxConcurrentCalls`. Wrong property for wrong service.
- **`maxDequeueCount` to 1 on queue resource:** `maxDequeueCount` là queue-level setting kiểm soát bao nhiêu lần message được delivery attempt trước khi dead-lettered. Setting = 1 → message chỉ được try 1 lần → ngay khi fail = dead-letter → mất toàn bộ retry capability. Không liên quan đến concurrency.

---

## Câu 4

**You need to store an API key for an Azure AI service in your function app's configuration. The security team requires that the key be stored in Azure Key Vault and rotated regularly without requiring application redeployment. Which approach meets these requirements?**

- Create a Key Vault reference in the application setting using a versionless secret URI ✅
- Create a Key Vault reference in the application setting using a versioned secret URI
- Store the API key directly in the application setting with encryption enabled

**Đáp án: Versionless secret URI**

```
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/AiServiceKey/)
```

Versionless URI (không có version GUID ở cuối) → runtime **tự động detect new secret versions** và switch sang latest version trong **24 giờ**. Config change cũng trigger immediate refetch. Không cần redeploy.

Tại sao các đáp án còn lại sai:
- **Versioned secret URI:** Versioned URI ghim vào một version cụ thể:
  `@Microsoft.KeyVault(SecretUri=https://vault.azure.net/secrets/AiKey/abc123def456)`
  Khi rotate key = tạo version mới → vẫn cần update application setting để trỏ version mới → cần redeploy. Không đáp ứng requirement "không cần redeploy."
- **Store directly with encryption:** Application settings được encrypt at rest bởi Azure, nhưng đây không phải Key Vault. Security team explicitly yêu cầu key phải trong Key Vault (auditing, centralized management, rotation policies). Direct storage không đáp ứng security requirement.

---

## Câu 5

**You're configuring identity-based connections for a Service Bus trigger in your function app. You've enabled a system-assigned managed identity and set `ServiceBusConnection__fullyQualifiedNamespace` to your namespace. The function fails to receive messages. What role assignment is missing?**

- Azure Service Bus Data Receiver on the Service Bus namespace ✅
- Azure Service Bus Data Owner on the Service Bus namespace
- Key Vault Secrets User on the Service Bus namespace

**Đáp án: Azure Service Bus Data Receiver**

Identity-based connection cần managed identity authenticate với Service Bus. Để **receive messages** (trigger), phải có **`Azure Service Bus Data Receiver`** role trên namespace.

```
ServiceBusConnection__fullyQualifiedNamespace = mynamespace.servicebus.windows.net
```
+ Managed identity với `Azure Service Bus Data Receiver` role → trigger hoạt động.

Tại sao các đáp án còn lại sai:
- **Azure Service Bus Data Owner:** Quá broad — Owner bao gồm read, write, manage. Least privilege principle → chỉ cần Data Receiver cho trigger. Mặc dù về mặt kỹ thuật sẽ work, đây không phải answer đúng về security best practice.
- **Key Vault Secrets User on Service Bus namespace:** Key Vault Secrets User là role cho **Azure Key Vault** để đọc secrets. Service Bus namespace không phải Key Vault — role này không apply và không giải quyết Service Bus authentication.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Reduce cold start, keep costs low | Flex Consumption + always-ready instances |
| 2 | Service Bus trigger không fire locally | AzureWebJobsStorage không configure |
| 3 | One message per instance | `maxConcurrentCalls: 1` trong host.json |
| 4 | Key Vault, rotate without redeploy | Versionless secret URI |
| 5 | Identity-based Service Bus trigger role | Azure Service Bus Data Receiver |

---

*Module hoàn thành. Learning Path kết thúc.*

---

[← Bài 5](./backend-m3-bai5-identity-access.md) · [🏠 Mục lục](../README.md) · [Tổng hợp →](./backend-summary.md)
