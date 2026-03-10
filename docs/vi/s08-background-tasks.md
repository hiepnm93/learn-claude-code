# s08: Tác vụ nền (Background Tasks)

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > [ s08 ] s09 > s10 > s11 > s12`

> *"Chạy các thao tác chậm ở nền; agent tiếp tục suy nghĩ"* -- các luồng daemon chạy lệnh, tiêm thông báo khi hoàn thành.

## Vấn đề

Một số lệnh mất hàng phút: `npm install`, `pytest`, `docker build`. Với vòng lặp chặn, model ngồi chờ không làm gì. Nếu người dùng yêu cầu "cài đặt dependencies và trong khi chờ, tạo file cấu hình," agent sẽ thực hiện tuần tự, không song song.

## Giải pháp

```
Luồng chính                  Luồng nền
+-----------------+        +-----------------+
| agent loop      |        | subprocess chạy |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^rút hàng đợi |        +-----------------+
+-----------------+

Dòng thời gian:
Agent --[khởi A]--[khởi B]--[công việc khác]----
             |          |
             v          v
          [A chạy]   [B chạy]      (song song)
             |          |
             +-- kết quả được tiêm trước lần gọi LLM tiếp theo --+
```

## Cách hoạt động

1. BackgroundManager theo dõi các tác vụ với hàng đợi thông báo an toàn luồng.

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

2. `run()` khởi chạy một luồng daemon và trả về ngay lập tức.

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

3. Khi subprocess hoàn thành, kết quả của nó được đưa vào hàng đợi thông báo.

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

4. Vòng lặp agent rút các thông báo trước mỗi lần gọi LLM.

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
            messages.append({"role": "assistant",
                "content": "Noted background results."})
        response = client.messages.create(...)
```

Vòng lặp vẫn chạy đơn luồng. Chỉ I/O của subprocess được song song hóa.

## Thay đổi so với s07

| Thành phần       | Trước (s07)      | Sau (s08)                          |
|------------------|------------------|------------------------------------|
| Công cụ          | 8                | 6 (cơ bản + background_run + check)|
| Thực thi         | Chỉ chặn         | Chặn + luồng nền                  |
| Thông báo        | Không có         | Hàng đợi rút mỗi vòng lặp        |
| Đồng thời        | Không có         | Luồng daemon                       |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

1. `Run "sleep 5 && echo done" in the background, then create a file while it runs`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
3. `Run pytest in the background and keep working on other things`
