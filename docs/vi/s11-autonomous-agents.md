# s11: Agent Tự chủ

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > [ s11 ] s12`

> *"Đồng đội tự quét bảng công việc và nhận nhiệm vụ"* -- không cần lead phân công từng việc.

## Vấn đề

Trong s09-s10, đồng đội chỉ làm việc khi được yêu cầu rõ ràng. Lead phải khởi tạo từng người với một prompt cụ thể. 10 nhiệm vụ chưa ai nhận trên bảng? Lead phải phân công từng cái một. Không thể mở rộng.

Tự chủ thực sự: đồng đội tự quét bảng nhiệm vụ, nhận các nhiệm vụ chưa ai nhận, thực hiện chúng, rồi tìm thêm việc mới.

Một điểm tinh tế: sau khi nén ngữ cảnh (s06), agent có thể quên mình là ai. Việc tái chèn danh tính sẽ khắc phục điều này.

## Giải pháp

```
Vòng đời đồng đội với chu kỳ nghỉ:

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (hoặc gọi công cụ idle)
    v
+--------+
|  IDLE  |  kiểm tra mỗi 5 giây trong tối đa 60 giây
+---+----+
    |
    +---> kiểm tra hộp thư --> có tin nhắn? -------> WORK
    |
    +---> quét .tasks/ --> có việc chưa nhận? -----> nhận -> WORK
    |
    +---> hết 60 giây -----------------------------> SHUTDOWN

Tái chèn danh tính sau khi nén:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

## Cách hoạt động

1. Vòng lặp đồng đội có hai giai đoạn: WORK và IDLE. Khi LLM ngừng gọi công cụ (hoặc gọi `idle`), đồng đội chuyển sang IDLE.

```python
def _loop(self, name, role, prompt):
    while True:
        # -- GIAI ĐOẠN LÀM VIỆC --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # thực thi công cụ...
            if idle_requested:
                break

        # -- GIAI ĐOẠN NGHỈ --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

2. Giai đoạn nghỉ kiểm tra hộp thư và bảng nhiệm vụ trong một vòng lặp.

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12
        time.sleep(POLL_INTERVAL)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True
    return False  # hết thời gian -> tắt máy
```

3. Quét bảng nhiệm vụ: tìm các nhiệm vụ đang chờ, chưa có chủ, không bị chặn.

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

4. Tái chèn danh tính: khi ngữ cảnh quá ngắn (đã bị nén), chèn một khối danh tính.

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

## Thay đổi so với s10

| Thành phần     | Trước (s10)      | Sau (s11)                        |
|----------------|------------------|----------------------------------|
| Công cụ        | 12               | 14 (+idle, +claim_task)          |
| Tự chủ         | Do lead điều khiển | Tự tổ chức                     |
| Giai đoạn nghỉ | Không có         | Kiểm tra hộp thư + bảng nhiệm vụ |
| Nhận nhiệm vụ  | Chỉ thủ công     | Tự nhận nhiệm vụ chưa ai nhận   |
| Danh tính      | System prompt    | + tái chèn sau khi nén           |
| Hết giờ        | Không có         | 60 giây nghỉ -> tự động tắt     |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
2. `Spawn a coder teammate and let it find work from the task board itself`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. Gõ `/tasks` để xem bảng nhiệm vụ với chủ sở hữu
5. Gõ `/team` để theo dõi ai đang làm việc và ai đang nghỉ
