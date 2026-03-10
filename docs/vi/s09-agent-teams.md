# s09: Nhóm Agent

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > [ s09 ] s10 > s11 > s12`

> *"Khi nhiệm vụ quá lớn cho một mình, hãy ủy thác cho đồng đội"* -- đồng đội bền vững + hộp thư bất đồng bộ.

## Vấn đề

Các subagent (s04) là dùng một lần: khởi tạo, làm việc, trả về tóm tắt, kết thúc. Không có danh tính, không có bộ nhớ giữa các lần gọi. Tác vụ nền (s08) chạy lệnh shell nhưng không thể đưa ra quyết định dựa trên LLM.

Làm việc nhóm thực sự cần: (1) các agent bền vững tồn tại lâu hơn một prompt đơn lẻ, (2) quản lý danh tính và vòng đời, (3) một kênh giao tiếp giữa các agent.

## Giải pháp

```
Vòng đời đồng đội:
  spawn -> WORKING -> IDLE -> WORKING -> ... -> SHUTDOWN

Giao tiếp:
  .team/
    config.json           <- danh sách đội + trạng thái
    inbox/
      alice.jsonl         <- chỉ ghi thêm, đọc xong xóa
      bob.jsonl
      lead.jsonl

              +--------+    send("alice","bob","...")    +--------+
              | alice  | -----------------------------> |  bob   |
              | loop   |    bob.jsonl << {json_line}    |  loop  |
              +--------+                                +--------+
                   ^                                         |
                   |        BUS.read_inbox("alice")          |
                   +---- alice.jsonl -> read + drain ---------+
```

## Cách hoạt động

1. TeammateManager duy trì config.json với danh sách thành viên đội.

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads = {}
```

2. `spawn()` tạo một đồng đội và khởi chạy vòng lặp agent của nó trong một luồng.

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}' (role: {role})"
```

3. MessageBus: hộp thư JSONL chỉ ghi thêm. `send()` ghi thêm một dòng JSON; `read_inbox()` đọc tất cả rồi xóa sạch.

```python
class MessageBus:
    def send(self, sender, to, content, msg_type="message", extra=None):
        msg = {"type": msg_type, "from": sender,
               "content": content, "timestamp": time.time()}
        if extra:
            msg.update(extra)
        with open(self.dir / f"{to}.jsonl", "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, name):
        path = self.dir / f"{name}.jsonl"
        if not path.exists(): return "[]"
        msgs = [json.loads(l) for l in path.read_text().strip().splitlines() if l]
        path.write_text("")  # xóa sạch
        return json.dumps(msgs, indent=2)
```

4. Mỗi đồng đội kiểm tra hộp thư trước mỗi lần gọi LLM, đưa các tin nhắn nhận được vào ngữ cảnh.

```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            messages.append({"role": "assistant",
                "content": "Noted inbox messages."})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # thực thi công cụ, thêm kết quả...
    self._find_member(name)["status"] = "idle"
```

## Thay đổi so với s08

| Thành phần     | Trước (s08)      | Sau (s09)                    |
|----------------|------------------|------------------------------|
| Công cụ        | 6                | 9 (+spawn/send/read_inbox)   |
| Agent          | Đơn lẻ           | Lead + N đồng đội            |
| Bền vững       | Không có         | config.json + hộp thư JSONL  |
| Luồng          | Lệnh nền         | Vòng lặp agent đầy đủ mỗi luồng |
| Vòng đời       | Bắn-và-quên      | idle -> working -> idle      |
| Giao tiếp      | Không có         | message + broadcast          |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s09_agent_teams.py
```

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. Gõ `/team` để xem danh sách đội với trạng thái
5. Gõ `/inbox` để kiểm tra thủ công hộp thư của lead
