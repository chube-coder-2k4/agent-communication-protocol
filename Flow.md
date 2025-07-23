# ACP Insurance-Hospital Flow Example

## Scenario: Xử lý yêu cầu bảo hiểm y tế

**Tình huống**: Một bệnh nhân vừa xuất viện và cần xử lý thanh toán bảo hiểm cho chi phí điều trị.

## Kiến trúc hệ thống

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│                 │    │                  │    │                 │
│ Insurance Server│    │ ACP CALLING      │    │ Hospital Server │
│ (Agent)         │◄──►│ AGENT            │◄──►│ (Agent)         │
│                 │    │ (Orchestrator)   │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

## Chi tiết các Agent

### 1. Hospital Server Agent
**Chức năng**: Quản lý thông tin bệnh nhân và chi phí điều trị
- Truy xuất hồ sơ bệnh án
- Tính toán chi phí điều trị
- Cung cấp báo cáo y tế
- Xác thực thông tin bệnh nhân

### 2. Insurance Server Agent  
**Chức năng**: Xử lý các yêu cầu bảo hiểm
- Kiểm tra tư cách bảo hiểm
- Tính toán số tiền được bồi thường
- Phê duyệt/từ chối yêu cầu
- Xử lý thanh toán

### 3. ACP Calling Agent
**Chức năng**: Điều phối và orchestrate toàn bộ quy trình
- Nhận yêu cầu từ user/system
- Gọi các agents theo thứ tự logic
- Xử lý lỗi và retry
- Trả kết quả cuối cùng

## Flow Chi Tiết

### Phase 1: Khởi tạo yêu cầu
```
User Request → ACP Calling Agent
"Xử lý bảo hiểm cho bệnh nhân ID: P12345, Hospital Bill: HB67890"
```

### Phase 2: Thu thập thông tin bệnh viện
```
ACP Calling Agent → Hospital Server Agent
Message: {
  "action": "get_patient_info",
  "patient_id": "P12345",
  "bill_id": "HB67890"
}

Hospital Server Agent → ACP Calling Agent
Response: {
  "patient_info": {
    "id": "P12345",
    "name": "Nguyen Van A",
    "insurance_number": "INS987654321"
  },
  "treatment_details": {
    "diagnosis": "Phẫu thuật ruột thừa",
    "treatment_period": "2025-07-20 to 2025-07-23",
    "total_cost": 15000000,
    "itemized_costs": [
      {"item": "Phẫu thuật", "cost": 8000000},
      {"item": "Thuốc", "cost": 3000000},
      {"item": "Phòng", "cost": 4000000}
    ]
  },
  "medical_documents": ["diagnosis_report.pdf", "surgery_report.pdf"]
}
```

### Phase 3: Kiểm tra bảo hiểm
```
ACP Calling Agent → Insurance Server Agent
Message: {
  "action": "validate_coverage",
  "insurance_number": "INS987654321",
  "treatment_info": {
    "diagnosis": "Phẫu thuật ruột thừa",
    "total_cost": 15000000,
    "treatment_date": "2025-07-20"
  }
}

Insurance Server Agent → ACP Calling Agent
Response: {
  "coverage_status": "active",
  "policy_details": {
    "coverage_percentage": 80,
    "max_coverage": 50000000,
    "deductible": 500000
  },
  "pre_approval_required": false
}
```

### Phase 4: Tính toán bồi thường
```
ACP Calling Agent → Insurance Server Agent
Message: {
  "action": "calculate_reimbursement",
  "total_cost": 15000000,
  "itemized_costs": [...],
  "policy_id": "INS987654321"
}

Insurance Server Agent → ACP Calling Agent
Response: {
  "calculation": {
    "total_eligible_cost": 14000000, // Trừ các chi phí không được bảo hiểm
    "deductible": 500000,
    "covered_amount": 10800000, // (14000000 - 500000) * 80%
    "patient_responsibility": 4200000
  },
  "status": "approved",
  "approval_code": "APP2025072301"
}
```

### Phase 5: Cập nhật thanh toán bệnh viện
```
ACP Calling Agent → Hospital Server Agent
Message: {
  "action": "update_payment_status",
  "bill_id": "HB67890",
  "insurance_payment": 10800000,
  "patient_payment": 4200000,
  "approval_code": "APP2025072301"
}

Hospital Server Agent → ACP Calling Agent
Response: {
  "payment_updated": true,
  "remaining_balance": 4200000,
  "payment_due_date": "2025-08-23",
  "receipt_number": "REC2025072301"
}
```

### Phase 6: Xác nhận với bảo hiểm
```
ACP Calling Agent → Insurance Server Agent
Message: {
  "action": "confirm_payment",
  "approval_code": "APP2025072301",
  "hospital_confirmation": "REC2025072301"
}

Insurance Server Agent → ACP Calling Agent
Response: {
  "payment_processed": true,
  "transaction_id": "TXN20250723001",
  "payment_date": "2025-07-23",
  "claim_number": "CLM2025072301"
}
```

## Code Implementation

### ACP Calling Agent (Orchestrator)
```python
# acp_calling_agent.py
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
    """Orchestrates insurance claim processing"""
    
    # Parse input
    request_data = json.loads(input[0].parts[0].content)
    patient_id = request_data["patient_id"]
    bill_id = request_data["bill_id"]
    
    yield {"thought": f"Starting insurance processing for patient {patient_id}"}
    
    try:
        # Step 1: Get hospital information
        yield {"status": "Fetching patient information from hospital..."}
        
        async with Client(base_url="http://hospital-server:8001") as hospital_client:
            hospital_response = await hospital_client.run_sync(
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
        
        patient_data = json.loads(hospital_response.output[0].parts[0].content)
        yield {"hospital_data_received": True, "patient_name": patient_data["patient_info"]["name"]}
        
        # Step 2: Validate insurance coverage
        yield {"status": "Validating insurance coverage..."}
        
        async with Client(base_url="http://insurance-server:8002") as insurance_client:
            coverage_response = await insurance_client.run_sync(
                agent="coverage_validator",
                input=[Message(parts=[MessagePart(
                    content=json.dumps({
                        "action": "validate_coverage",
                        "insurance_number": patient_data["patient_info"]["insurance_number"],
                        "treatment_info": patient_data["treatment_details"]
                    }),
                    content_type="application/json"
                )])]
            )
        
        coverage_data = json.loads(coverage_response.output[0].parts[0].content)
        
        if coverage_data["coverage_status"] != "active":
            yield {"error": "Insurance coverage is not active", "status": "failed"}
            return
            
        yield {"coverage_validated": True, "coverage_percentage": coverage_data["policy_details"]["coverage_percentage"]}
        
        # Step 3: Calculate reimbursement
        yield {"status": "Calculating reimbursement amount..."}
        
        reimbursement_response = await insurance_client.run_sync(
            agent="reimbursement_calculator",
            input=[Message(parts=[MessagePart(
                content=json.dumps({
                    "action": "calculate_reimbursement",
                    "total_cost": patient_data["treatment_details"]["total_cost"],
                    "itemized_costs": patient_data["treatment_details"]["itemized_costs"],
                    "policy_id": patient_data["patient_info"]["insurance_number"]
                }),
                content_type="application/json"
            )])]
        )
        
        reimbursement_data = json.loads(reimbursement_response.output[0].parts[0].content)
        
        if reimbursement_data["status"] != "approved":
            yield {"error": "Reimbursement not approved", "status": "failed"}
            return
            
        yield {"reimbursement_calculated": True, "covered_amount": reimbursement_data["calculation"]["covered_amount"]}
        
        # Step 4: Update hospital payment status
        yield {"status": "Updating hospital payment records..."}
        
        async with Client(base_url="http://hospital-server:8001") as hospital_client:
            payment_response = await hospital_client.run_sync(
                agent="payment_updater",
                input=[Message(parts=[MessagePart(
                    content=json.dumps({
                        "action": "update_payment_status",
                        "bill_id": bill_id,
                        "insurance_payment": reimbursement_data["calculation"]["covered_amount"],
                        "patient_payment": reimbursement_data["calculation"]["patient_responsibility"],
                        "approval_code": reimbursement_data["approval_code"]
                    }),
                    content_type="application/json"
                )])]
            )
        
        payment_data = json.loads(payment_response.output[0].parts[0].content)
        yield {"hospital_updated": True, "receipt_number": payment_data["receipt_number"]}
        
        # Step 5: Confirm payment with insurance
        yield {"status": "Confirming payment with insurance provider..."}
        
        async with Client(base_url="http://insurance-server:8002") as insurance_client:
            confirmation_response = await insurance_client.run_sync(
                agent="payment_processor",
                input=[Message(parts=[MessagePart(
                    content=json.dumps({
                        "action": "confirm_payment",
                        "approval_code": reimbursement_data["approval_code"],
                        "hospital_confirmation": payment_data["receipt_number"]
                    }),
                    content_type="application/json"
                )])]
            )
        
        confirmation_data = json.loads(confirmation_response.output[0].parts[0].content)
        
        # Final result
        yield {
            "status": "completed",
            "summary": {
                "patient_id": patient_id,
                "patient_name": patient_data["patient_info"]["name"],
                "total_cost": patient_data["treatment_details"]["total_cost"],
                "insurance_payment": reimbursement_data["calculation"]["covered_amount"],
                "patient_responsibility": reimbursement_data["calculation"]["patient_responsibility"],
                "claim_number": confirmation_data["claim_number"],
                "transaction_id": confirmation_data["transaction_id"]
            }
        }
        
    except Exception as e:
        yield {"error": str(e), "status": "failed"}

if __name__ == "__main__":
    server.run(host="0.0.0.0", port=8000)
```

### Hospital Server Agent
```python
# hospital_server.py
import asyncio
from collections.abc import AsyncGenerator
from acp_sdk.models import Message, MessagePart
from acp_sdk.server import Context, RunYield, RunYieldResume, Server
import json

server = Server()

# Mock database
PATIENTS_DB = {
    "P12345": {
        "name": "Nguyen Van A",
        "insurance_number": "INS987654321",
        "phone": "0901234567"
    }
}

BILLS_DB = {
    "HB67890": {
        "patient_id": "P12345",
        "diagnosis": "Phẫu thuật ruột thừa",
        "treatment_period": "2025-07-20 to 2025-07-23",
        "total_cost": 15000000,
        "itemized_costs": [
            {"item": "Phẫu thuật", "cost": 8000000},
            {"item": "Thuốc", "cost": 3000000},
            {"item": "Phòng", "cost": 4000000}
        ],
        "status": "pending_payment"
    }
}

@server.agent()
async def hospital_info(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    """Provides patient and billing information"""
    
    request = json.loads(input[0].parts[0].content)
    action = request["action"]
    
    if action == "get_patient_info":
        patient_id = request["patient_id"]
        bill_id = request["bill_id"]
        
        yield {"thought": f"Looking up patient {patient_id} and bill {bill_id}"}
        
        if patient_id not in PATIENTS_DB or bill_id not in BILLS_DB:
            yield {"error": "Patient or bill not found"}
            return
            
        response = {
            "patient_info": PATIENTS_DB[patient_id],
            "treatment_details": BILLS_DB[bill_id],
            "medical_documents": ["diagnosis_report.pdf", "surgery_report.pdf"]
        }
        
        yield Message(parts=[MessagePart(
            content=json.dumps(response),
            content_type="application/json"
        )])

@server.agent()
async def payment_updater(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    """Updates payment status for bills"""
    
    request = json.loads(input[0].parts[0].content)
    bill_id = request["bill_id"]
    
    yield {"thought": f"Updating payment for bill {bill_id}"}
    
    # Update database (mock)
    BILLS_DB[bill_id]["status"] = "insurance_processed"
    BILLS_DB[bill_id]["insurance_payment"] = request["insurance_payment"]
    BILLS_DB[bill_id]["patient_payment"] = request["patient_payment"]
    
    response = {
        "payment_updated": True,
        "remaining_balance": request["patient_payment"],
        "payment_due_date": "2025-08-23",
        "receipt_number": f"REC{bill_id}001"
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

if __name__ == "__main__":
    server.run(host="0.0.0.0", port=8001)
```

### Insurance Server Agent
```python
# insurance_server.py
import asyncio
from collections.abc import AsyncGenerator
from acp_sdk.models import Message, MessagePart
from acp_sdk.server import Context, RunYield, RunYieldResume, Server
import json
from datetime import datetime

server = Server()

# Mock insurance database
POLICIES_DB = {
    "INS987654321": {
        "status": "active",
        "coverage_percentage": 80,
        "max_coverage": 50000000,
        "deductible": 500000,
        "covered_treatments": ["surgery", "medication", "hospitalization"]
    }
}

@server.agent()
async def coverage_validator(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    """Validates insurance coverage"""
    
    request = json.loads(input[0].parts[0].content)
    insurance_number = request["insurance_number"]
    
    yield {"thought": f"Validating coverage for {insurance_number}"}
    
    if insurance_number not in POLICIES_DB:
        yield {"error": "Insurance policy not found"}
        return
        
    policy = POLICIES_DB[insurance_number]
    
    response = {
        "coverage_status": policy["status"],
        "policy_details": {
            "coverage_percentage": policy["coverage_percentage"],
            "max_coverage": policy["max_coverage"],
            "deductible": policy["deductible"]
        },
        "pre_approval_required": False
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

@server.agent()
async def reimbursement_calculator(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    """Calculates reimbursement amount"""
    
    request = json.loads(input[0].parts[0].content)
    total_cost = request["total_cost"]
    policy_id = request["policy_id"]
    
    yield {"thought": "Calculating reimbursement based on policy terms"}
    
    policy = POLICIES_DB[policy_id]
    
    # Assume some costs are not covered (e.g., luxury room upgrade)
    eligible_cost = total_cost * 0.93  # 93% eligible
    
    # Apply deductible
    after_deductible = max(0, eligible_cost - policy["deductible"])
    
    # Apply coverage percentage
    covered_amount = int(after_deductible * (policy["coverage_percentage"] / 100))
    patient_responsibility = total_cost - covered_amount
    
    response = {
        "calculation": {
            "total_eligible_cost": int(eligible_cost),
            "deductible": policy["deductible"],
            "covered_amount": covered_amount,
            "patient_responsibility": patient_responsibility
        },
        "status": "approved",
        "approval_code": f"APP{datetime.now().strftime('%Y%m%d%H%M')}"
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

@server.agent()
async def payment_processor(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    """Processes final payment confirmation"""
    
    request = json.loads(input[0].parts[0].content)
    approval_code = request["approval_code"]
    
    yield {"thought": f"Processing payment for approval {approval_code}"}
    
    response = {
        "payment_processed": True,
        "transaction_id": f"TXN{datetime.now().strftime('%Y%m%d%H%M%S')}",
        "payment_date": datetime.now().strftime('%Y-%m-%d'),
        "claim_number": f"CLM{datetime.now().strftime('%Y%m%d%H%M')}"
    }
    
    yield Message(parts=[MessagePart(
        content=json.dumps(response),
        content_type="application/json"
    )])

if __name__ == "__main__":
    server.run(host="0.0.0.0", port=8002)
```

## Usage Example

### Client Request
```python
# client.py
import asyncio
from acp_sdk.client import Client
from acp_sdk.models import Message, MessagePart
import json

async def process_insurance_claim():
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
        
        print("Insurance processing result:")
        for output in run.output:
            result = json.loads(output.parts[0].content)
            print(json.dumps(result, indent=2, ensure_ascii=False))

if __name__ == "__main__":
    asyncio.run(process_insurance_claim())
```

## Deployment với Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  acp-calling-agent:
    build: ./acp-calling-agent
    ports:
      - "8000:8000"
    depends_on:
      - hospital-server
      - insurance-server
    environment:
      - HOSPITAL_URL=http://hospital-server:8001
      - INSURANCE_URL=http://insurance-server:8002

  hospital-server:
    build: ./hospital-server
    ports:
      - "8001:8001"

  insurance-server:
    build: ./insurance-server
    ports:
      - "8002:8002"
```

## Expected Output Flow

```
1. ✅ Starting insurance processing for patient P12345
2. ✅ Fetching patient information from hospital...
3. ✅ Hospital data received: Nguyen Van A
4. ✅ Validating insurance coverage...
5. ✅ Coverage validated: 80% coverage
6. ✅ Calculating reimbursement amount... 
7. ✅ Reimbursement calculated: 10,800,000 VND covered
8. ✅ Updating hospital payment records...
9. ✅ Hospital updated: Receipt REC2025072301
10. ✅ Confirming payment with insurance provider...
11. ✅ COMPLETED: Claim CLM2025072301 processed successfully

Final Summary:
- Total Cost: 15,000,000 VND
- Insurance Payment: 10,800,000 VND  
- Patient Responsibility: 4,200,000 VND
- Transaction ID: TXN20250723145030
```

## Benefits của ACP trong scenario này

1. **Loose Coupling**: Mỗi system có thể được phát triển và maintain độc lập
2. **Error Handling**: Centralized error handling trong orchestrator
3. **Transparency**: Clear visibility vào từng bước của process
4. **Scalability**: Có thể thêm các agents khác (Pharmacy, Lab, etc.)
5. **Retry Logic**: Built-in retry mechanisms cho failed requests
6. **Audit Trail**: Complete log của toàn bộ transaction flow
