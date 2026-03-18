# Hướng dẫn chi tiết: Tính năng Chat Real-time với SignalR

Tài liệu này giải thích chi tiết cách tính năng Chat hoạt động trong dự án, từ luồng dữ liệu (data flow) đến code implementation.

## 1. Tổng quan Kiến trúc

Tính năng Chat được xây dựng dựa trên mô hình **Client-Server-Client**:

1.  **Client A** gửi tin nhắn lên Server.
2.  **Server** lưu tin nhắn vào bộ nhớ tạm (In-memory Store).
3.  **Server** forward (chuyển tiếp) tin nhắn đó xuống **TẤT CẢ** các Client đang kết nối (bao gồm cả Client A).
4.  Khi một **Client mới** tham gia, nó sẽ tải lại lịch sử tin nhắn cũ từ Server.

---

## 2. Phân tích chi tiết Server-side (Backend)

File: `Hubs/ChatHub.cs`

### A. Data Model (`ChatMessage`)
Định nghĩa cấu trúc của một tin nhắn.

```csharp
public class ChatMessage
{
    public string UserName { get; set; } = string.Empty; // Tên người chat
    public string Message { get; set; } = string.Empty;  // Nội dung chat
    public DateTime Timestamp { get; set; }              // Thời gian gửi
}
```

### B. Lưu trữ dữ liệu (`ChatMessageStore`)
Vì demo đơn giản nên ta dùng một `static List` để lưu tin nhắn trong RAM. `lock` được sử dụng để đảm bảo an toàn khi nhiều người chat cùng lúc (Thread-safety).

```csharp
public static class ChatMessageStore
{
    private static List<ChatMessage> _messages = new(); // Danh sách tin nhắn
    private static object _lock = new object();         // Khóa đồng bộ

    // Lấy toàn bộ lịch sử tin nhắn
    public static List<ChatMessage> GetMessages()
    {
        lock (_lock) // Chặn các luồng khác khi đang đọc
        {
            return _messages.OrderBy(x => x.Timestamp).ToList();
        }
    }

    // Thêm tin nhắn mới
    public static void AddMessage(ChatMessage message)
    {
        lock (_lock) // Chặn các luồng khác khi đang ghi
        {
            _messages.Add(message);
            // Giới hạn chỉ lưu 500 tin nhắn gần nhất để tránh tràn RAM
            if (_messages.Count > 500)
                _messages.RemoveAt(0);
        }
    }
}
```

### C. Hub xử lý tín hiệu (`ChatHub`)
Đây là trung tâm điều phối của SignalR.

```csharp
public class ChatHub : Hub
{
    // Method này được Client gọi trực tiếp thông qua JS
    public async Task SendMessage(string userName, string message)
    {
        // 1. Tạo object tin nhắn
        var chatMessage = new ChatMessage
        {
            UserName = userName,
            Message = message,
            Timestamp = DateTime.Now
        };

        // 2. Lưu vào bộ nhớ
        ChatMessageStore.AddMessage(chatMessage);

        // 3. Gửi tin nhắn đến TẤT CẢ clients đang kết nối
        // "ReceiveMessage" là tên sự kiện mà Client JS sẽ lắng nghe
        await Clients.All.SendAsync("ReceiveMessage", chatMessage);
    }
}
```

---

## 3. Phân tích chi tiết Client-side (Frontend)

File: `Pages/Chat/Index.cshtml`

### A. Kết nối đến Hub

```javascript
// Tạo kết nối đến đường dẫn "/chatHub" (đã config trong Program.cs)
var connection = new signalR.HubConnectionBuilder()
    .withUrl("/chatHub")
    .build();
```

### B. Gửi tin nhắn (Client -> Server)

Khi người dùng bấm nút "Send":

```javascript
document.getElementById("sendButton").addEventListener("click", function (event) {
    var user = document.getElementById("userInput").value;
    var message = document.getElementById("messageInput").value;
    
    // Gọi method "SendMessage" trên Server (trong ChatHub class)
    if(user && message) {
        connection.invoke("SendMessage", user, message)
            .catch(function (err) {
                return console.error(err.toString());
            });
            
        // Xóa ô nhập liệu sau khi gửi
        document.getElementById("messageInput").value = '';
    }
    event.preventDefault();
});
```

**Lưu ý:** Client **KHÔNG** tự hiển thị tin nhắn ngay khi bấm Send. Nó chờ Server gửi lại sự kiện `ReceiveMessage` để đảm bảo đồng bộ với mọi người.

### C. Nhận tin nhắn (Server -> Client)

Client lắng nghe sự kiện `ReceiveMessage` từ Server:

```javascript
connection.on("ReceiveMessage", function (msg) {
    // msg là object ChatMessage do server trả về
    
    // Tạo thẻ div mới
    var div = document.createElement("div");
    
    // Format nội dung: "Tên: Tin nhắn"
    div.textContent = msg.userName + ": " + msg.message;
    
    // Thêm vào danh sách tin nhắn
    document.getElementById("messagesList").appendChild(div);
});
```

### D. Tải lịch sử chat (Khi mới vào)

Khi vừa kết nối thành công (`connection.start()`), Client gọi API để lấy tin nhắn cũ:

```javascript
connection.start().then(function () {
     console.log("SignalR Connected.");
     
     // Gọi Handler GetMessages trong file Chat/Index.cshtml.cs
     fetch('/Chat?handler=GetMessages')
        .then(response => response.json())
        .then(data => {
            // Duyệt qua từng tin nhắn cũ và hiển thị lên
            data.forEach(msg => {
                // ... code hiển thị tin nhắn (giống phần ReceiveMessage)
            });
        });
});
```

API Backup (`Pages/Chat/Index.cshtml.cs`):
```csharp
public IActionResult OnGetGetMessages()
{
    // Lấy dữ liệu từ Store trả về JSON cho client
    var messages = ChatMessageStore.GetMessages();
    return new JsonResult(messages);
}
```

---

## 4. Luồng hoạt động đầy đủ (Sequence Flow)

Giả sử **User A** (Tên: "Alice") gửi "Hello":

1.  **Browser Alice**:
    *   User nhập "Alice", "Hello" -> Bấm Send.
    *   JS gọi `connection.invoke("SendMessage", "Alice", "Hello")`.
2.  **Server (`ChatHub`)**:
    *   Nhận yêu cầu `SendMessage`.
    *   Tạo object `ChatMessage { User="Alice", Message="Hello", ... }`.
    *   Lưu vào `ChatMessageStore` (List trong RAM).
    *   Gọi `Clients.All.SendAsync("ReceiveMessage", chatMessage)`.
3.  **Browser Alice** (và **Browser Bob**, **Browser Charlie**...):
    *   Nhận sự kiện `ReceiveMessage`.
    *   Kích hoạt callback function trong `.on("ReceiveMessage", ...)`.
    *   JS tạo `<div>Alice: Hello</div>` và chèn vào khung chat.

## 5. Tại sao cần `ChatMessageStore`?

Nếu không có `ChatMessageStore`:
*   Alice chat "Hello".
*   Bob nhận được "Hello".
*   **Charlie** vào phòng chat muộn 5 phút sau.
*   Charile sẽ **không thấy** tin "Hello" của Alice vì nó đã được gửi xong rồi.

=> `ChatMessageStore` giúp người vào sau vẫn đọc được nội dung cũ (Persistence).

## 6. Config trong `Program.cs`

Để SignalR hoạt động, cần 2 dòng lệnh quan trọng trong `Program.cs`:

```csharp
// 1. Đăng ký dịch vụ SignalR vào Container
builder.Services.AddSignalR(); 

// ...

// 2. Định tuyến (Ranking) URL đến Hub
app.MapHub<ChatHub>("/chatHub");
```

