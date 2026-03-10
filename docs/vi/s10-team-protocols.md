# s10: Giao thức Nhóm

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > [ s10 ] s11 > s12`

> *"Đồng đội cần có quy tắc giao tiếp chung"* -- một mẫu yêu cầu-phản hồi điều khiển mọi cuộc đàm phán.

## Vấn đề

Trong s09, các đồng đội làm việc và giao tiếp nhưng thiếu sự phối hợp có cấu trúc:

**Tắt máy**: Hủy một luồng sẽ để lại tệp viết dở và config.json lỗi thời. Bạn cần một quy trình bắt tay: lead yêu cầu, đồng đội chấp thuận (hoàn thành và thoát) hoặc từ chối (tiếp tục làm việc).

**Phê duyệt kế hoạch**: Khi lead nói "tái cấu trúc module auth," đồng đội bắt đầu ngay lập tức. Với những thay đổi rủi ro cao, lead nên xem xét kế hoạch trước.

Cả hai đều có cùng cấu trúc: một bên gửi yêu cầu với ID duy nhất, bên kia phản hồi tham chiếu đến ID đó.

## Giải pháp

```
Giao thức Tắt máy          Giao thức Phê duyệt Kế hoạch
==================           ======================

Lead             Đồng đội   Đồng đội           Lead
  |                 |           |                 |
  |--shutdown_req-->|           |--plan_req------>|
  | {req_id:"abc"}  |           | {req_id:"xyz"}  |
  |                 |           |                 |
  |<--shutdown_resp-|           |<--plan_resp-----|
  | {req_id:"abc",  |           | {req_id:"xyz",  |
  |  approve:true}  |           |  approve:true}  |

FSM chung:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]

Bộ theo dõi:
  shutdown_requests = {req_id: {target, status}}
  plan_requests     = {req_id: {from, plan, status}}
```

## Cách hoạt động

1. Lead khởi tạo việc tắt máy bằng cách tạo một request_id và gửi qua hộp thư.

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

2. Đồng đội nhận yêu cầu và phản hồi bằng chấp thuận/từ chối.

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

3. Phê duyệt kế hoạch tuân theo mẫu giống hệt. Đồng đội gửi kế hoạch (tạo một request_id), lead xem xét (tham chiếu cùng request_id đó).

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

Một FSM, hai ứng dụng. Cùng một máy trạng thái `pending -> approved | rejected` xử lý mọi giao thức yêu cầu-phản hồi.

## Thay đổi so với s09

| Thành phần     | Trước (s09)      | Sau (s10)                        |
|----------------|------------------|----------------------------------|
| Công cụ        | 9                | 12 (+shutdown_req/resp +plan)    |
| Tắt máy        | Chỉ thoát tự nhiên | Bắt tay yêu cầu-phản hồi      |
| Kiểm soát kế hoạch | Không có    | Gửi/xem xét với phê duyệt       |
| Tương quan     | Không có         | request_id cho mỗi yêu cầu      |
| FSM            | Không có         | pending -> approved/rejected     |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. Gõ `/team` để theo dõi trạng thái
