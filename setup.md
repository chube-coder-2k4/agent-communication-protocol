# ACP - Setup hoàn toàn miễn phí

## ✅ ACP không cần API key hay chi phí

### 🆓 Hoàn toàn miễn phí
- **ACP Protocol**: Open source, không có license fee
- **ACP SDK**: Miễn phí sử dụng
- **Infrastructure**: Chạy trên máy local của bạn
- **No API calls**: Không gọi external services tính phí

### 🏠 Chạy hoàn toàn local
```bash
# Tất cả chạy trên máy bạn, không cần internet
localhost:8000  # ACP Calling Agent
localhost:8001  # Hospital Server  
localhost:8002  # Insurance Server
```

## 💻 Setup chi tiết không tốn tiền

### Bước 1: Cài đặt Python (miễn phí)
```bash
# Nếu chưa có Python
# Download từ python.org (miễn phí)
python --version  # Check version >= 3.11
```

### Bước 2: Cài đặt uv (miễn phí)
```bash
# uv là package manager miễn phí
curl -LsSf https://astral.sh/uv/install.sh | sh
# Hoặc trên Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### Bước 3: Tạo project (0 đồng)
```bash
mkdir acp-insurance-demo
cd acp-insurance-demo

# Tạo 3 folders cho 3 agents
mkdir acp-calling-agent hospital-server insurance-server
```

### Bước 4: Setup từng agent (miễn phí)

#### ACP Calling Agent
```bash
cd acp-calling-agent
uv init --python '>=3.11' .
uv add acp-sdk  # Miễn phí từ PyPI
```

#### Hospital Server
```bash
cd ../hospital-server
uv init --python '>=3.11' .
uv add acp-sdk  # Miễn phí
```

#### Insurance Server
```bash
cd ../insurance-server
uv init --python '>=3.11' .
uv add acp-sdk  # Miễn phí
```

## 🚀 Chạy hệ thống (0 chi phí vận hành)

### Terminal 1: Hospital Server
```bash
cd hospital-server
# Copy code từ ví dụ trước vào hospital_server.py
uv run hospital_server.py
# ✅ Server running at http://localhost:8001
```

### Terminal 2: Insurance Server  
```bash
cd insurance-server
# Copy code từ ví dụ trước vào insurance_server.py
uv run insurance_server.py
# ✅ Server running at http://localhost:8002
```

### Terminal 3: ACP Calling Agent
```bash
cd acp-calling-agent
# Copy code từ ví dụ trước vào acp_calling_agent.py
uv run acp_calling_agent.py
# ✅ Server running at http://localhost:8000
```

### Terminal 4: Test client
```bash
# Copy code client từ ví dụ trước vào client.py
uv run client.py
# ✅ Xem kết quả xử lý bảo hiểm
```

## 🎯 Demo code hoàn chỉnh miễn phí

### hospital_server.py
```python
import asyncio
from collections.abc import AsyncGenerator
from acp_sdk.models import Message, MessagePart
from acp_sdk.server import Context, RunYield, RunYieldResume, Server
import json

server = Server()

# Mock database - không cần real database
PATIENTS = {
    "P12345": {
        "name": "Nguyen Van A",
        "insurance_number": "INS987654321"
    }
}

BILLS = {
    "HB67890": {
        "patient_id": "P12345", 
        "diagnosis": "Phẫu thuật ruột thừa",
        "total_cost": 15000000,
        "itemized_costs": [
            {"item": "Phẫu thuật", "cost": 8000000},
            {"item": "Thuốc", "cost": 3000000}, 
            {"item": "Phòng", "cost": 4000000}
        ]
    }
}

@server.agent()
async def hospital_info(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    
    request = json.loads(input[0].parts[0].content)
    
    if request["action"] == "get_patient_info":
        patient_id = request["patient_id"]
        bill_id = request["bill_id"]
        
        yield {"status": f"Looking up patient {patient_id}"}
        
        response = {
            "patient_info": PATIENTS[patient_id],
            "treatment_details": BILLS[bill_id]
        }
        
        yield Message(parts=[MessagePart(
            content=json.dumps(response),
            content_type="application/json"
        )])

@server.agent() 
async def payment_updater(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    
    request = json.loads(input[0].parts[0].content)
    
    yield {"status": "Updating payment status"}
    
    response = {
        "payment_updated": True,
        "receipt_number": "REC2025001"
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

if __name__ == "__main__":
    server.run(host="0.0.0.0", port=8001)
```

### insurance_server.py
```python
import asyncio
from collections.abc import AsyncGenerator
from acp_sdk.models import Message, MessagePart
from acp_sdk.server import Context, RunYield, RunYieldResume, Server
import json

server = Server()

# Mock insurance database
POLICIES = {
    "INS987654321": {
        "status": "active",
        "coverage_percentage": 80,
        "deductible": 500000
    }
}

@server.agent()
async def coverage_validator(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    
    request = json.loads(input[0].parts[0].content)
    insurance_number = request["insurance_number"]
    
    yield {"status": f"Validating {insurance_number}"}
    
    policy = POLICIES[insurance_number]
    response = {
        "coverage_status": policy["status"],
        "policy_details": policy
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

@server.agent()
async def reimbursement_calculator(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    
    request = json.loads(input[0].parts[0].content)
    total_cost = request["total_cost"]
    
    yield {"status": "Calculating reimbursement"}
    
    # Simple calculation
    deductible = 500000
    after_deductible = max(0, total_cost - deductible)
    covered_amount = int(after_deductible * 0.8)  # 80%
    
    response = {
        "calculation": {
            "covered_amount": covered_amount,
            "patient_responsibility": total_cost - covered_amount
        },
        "status": "approved",
        "approval_code": "APP2025001"
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

@server.agent()
async def payment_processor(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    
    yield {"status": "Processing payment"}
    
    response = {
        "payment_processed": True,
        "transaction_id": "TXN2025001",
        "claim_number": "CLM2025001"
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

if __name__ == "__main__":
    server.run(host="0.0.0.0", port=8002)
```

### acp_calling_agent.py
```python
import asyncio
from collections.abc import AsyncGenerator
from acp_sdk.models import Message, MessagePart
from acp_sdk.server import Context, RunYield, RunYieldResume, Server
from acp_sdk.client import Client
import json

server = Server()

@server.agent()
async def insurance_orchestrator(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    
    request_data = json.loads(input[0].parts[0].content)
    patient_id = request_data["patient_id"] 
    bill_id = request_data["bill_id"]
    
    yield {"status": f"Processing insurance for patient {patient_id}"}
    
    try:
        # Step 1: Get hospital info
        yield {"step": "1/4 - Getting hospital information"}
        
        async with Client(base_url="http://localhost:8001") as client:
            hospital_response = await client.run_sync(
                agent="hospital_info",
                input=[Message(parts=[MessagePart(
                    content=json.dumps({
                        "action": "get_patient_info",
                        "patient_id": patient_id,
                        "bill_id": bill_id
                    }),
                    content_type="application/json"
                )])]
            )
        
        hospital_data = json.loads(hospital_response.output[0].parts[0].content)
        yield {"hospital_data": "✅ Received"}
        
        # Step 2: Validate insurance
        yield {"step": "2/4 - Validating insurance"}
        
        async with Client(base_url="http://localhost:8002") as client:
            coverage_response = await client.run_sync(
                agent="coverage_validator", 
                input=[Message(parts=[MessagePart(
                    content=json.dumps({
                        "action": "validate_coverage",
                        "insurance_number": hospital_data["patient_info"]["insurance_number"],
                        "treatment_info": hospital_data["treatment_details"]
                    }),
                    content_type="application/json"
                )])]
            )
        
        coverage_data = json.loads(coverage_response.output[0].parts[0].content)
        yield {"insurance_validation": "✅ Active coverage"}
        
        # Step 3: Calculate reimbursement
        yield {"step": "3/4 - Calculating reimbursement"}
        
        async with Client(base_url="http://localhost:8002") as client:
            calc_response = await client.run_sync(
                agent="reimbursement_calculator",
                input=[Message(parts=[MessagePart(
                    content=json.dumps({
                        "action": "calculate_reimbursement",
                        "total_cost": hospital_data["treatment_details"]["total_cost"],
                        "policy_id": hospital_data["patient_info"]["insurance_number"]
                    }),
                    content_type="application/json"
                )])]
            )
        
        calc_data = json.loads(calc_response.output[0].parts[0].content)
        yield {"reimbursement": f"✅ {calc_data['calculation']['covered_amount']:,} VND"}
        
        # Step 4: Update hospital
        yield {"step": "4/4 - Updating hospital records"}
        
        async with Client(base_url="http://localhost:8001") as client:
            update_response = await client.run_sync(
                agent="payment_updater",
                input=[Message(parts=[MessagePart(
                    content=json.dumps({
                        "action": "update_payment_status",
                        "bill_id": bill_id,
                        "insurance_payment": calc_data["calculation"]["covered_amount"]
                    }),
                    content_type="application/json"
                )])]
            )
        
        yield {"hospital_update": "✅ Records updated"}
        
        # Final summary
        yield {
            "status": "✅ COMPLETED",
            "summary": {
                "patient": hospital_data["patient_info"]["name"],
                "total_cost": f"{hospital_data['treatment_details']['total_cost']:,} VND",
                "insurance_pays": f"{calc_data['calculation']['covered_amount']:,} VND", 
                "patient_pays": f"{calc_data['calculation']['patient_responsibility']:,} VND"
            }
        }
        
    except Exception as e:
        yield {"error": f"❌ {str(e)}"}

if __name__ == "__main__":
    server.run(host="0.0.0.0", port=8000)
```

### client.py (Test script)
```python
import asyncio
from acp_sdk.client import Client
from acp_sdk.models import Message, MessagePart
import json

async def test_insurance_flow():
    print("🏥 Testing Insurance Processing Flow...")
    
    async with Client(base_url="http://localhost:8000") as client:
        run = await client.run_sync(
            agent="insurance_orchestrator",
            input=[Message(parts=[MessagePart(
                content=json.dumps({
                    "patient_id": "P12345",
                    "bill_id": "HB67890" 
                }),
                content_type="application/json"
            )])],
        )
        
        for output in run.output:
            if output.parts[0].content_type == "application/json":
                result = json.loads(output.parts[0].content)
                print(json.dumps(result, indent=2, ensure_ascii=False))
            else:
                print(output.parts[0].content)

if __name__ == "__main__":
    asyncio.run(test_insurance_flow())
```

## 📋 Checklist chạy demo

### ✅ Không cần:
- [ ] API keys
- [ ] Credit cards  
- [ ] Subscriptions
- [ ] Cloud services
- [ ] External databases
- [ ] Docker (optional)

### ✅ Chỉ cần:
- [x] Python 3.11+
- [x] uv (miễn phí)
- [x] acp-sdk (miễn phí)
- [x] 4 terminal windows
- [x] Copy/paste code

## 🎯 Expected Output (miễn phí)

```json
{
  "status": "Processing insurance for patient P12345"
}
{
  "step": "1/4 - Getting hospital information"
}
{
  "hospital_data": "✅ Received"
}
{
  "step": "2/4 - Validating insurance" 
}
{
  "insurance_validation": "✅ Active coverage"
}
{
  "step": "3/4 - Calculating reimbursement"
}
{
  "reimbursement": "✅ 11,600,000 VND"
}
{
  "step": "4/4 - Updating hospital records"
}
{
  "hospital_update": "✅ Records updated"
}
{
  "status": "✅ COMPLETED",
  "summary": {
    "patient": "Nguyen Van A",
    "total_cost": "15,000,000 VND",
    "insurance_pays": "11,600,000 VND",
    "patient_pays": "3,400,000 VND"
  }
}
```

## 💡 Tại sao ACP miễn phí?

1. **Open Source**: Mã nguồn mở, community-driven
2. **Local Infrastructure**: Không dùng cloud services
3. **No External APIs**: Không gọi services tính phí
4. **Self-contained**: Tất cả chạy trên máy bạn
5. **Mock Data**: Dùng fake data, không cần real databases

## 🚀 Mở rộng (vẫn miễn phí)

- **Thêm agents**: Pharmacy, Lab tests, etc.
- **Real databases**: SQLite (miễn phí)
- **Web UI**: Flask/FastAPI (miễn phí)  
- **Monitoring**: Logging libraries (miễn phí)
- **Docker**: Containerization (optional, miễn phí)

Bạn có thể chạy toàn bộ hệ thống này mà không tốn một xu nào! 🎉
