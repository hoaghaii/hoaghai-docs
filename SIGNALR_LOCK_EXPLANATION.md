# Giải thích chi tiết về cơ chế Lock trong SignalR

Trong các ứng dụng web đa luồng (multi-threaded) như SignalR, nhiều người dùng có thể gửi yêu cầu đến server cùng một lúc. Điều này đặt ra vấn đề về an toàn dữ liệu khi nhiều luồng cùng truy cập vào một vùng nhớ chung (Shared Resource).

Tài liệu này sẽ giải thích tại sao chúng ta cần sử dụng `lock` trong class `ChatMessageStore`.

## 1. Vấn đề: Truy cập đồng thời (Concurrency Issue)

Trong dự án này, chúng ta sử dụng một danh sách tĩnh (`static List<ChatMessage>`) để lưu trữ tất cả tin nhắn chat:

```csharp
private static List<ChatMessage> _messages = new();
```

Biến `static` có nghĩa là **chỉ có một bản sao duy nhất** của danh sách này tồn tại trong bộ nhớ server và được chia sẻ cho **tất cả người dùng**.

### Kịch bản lỗi (Race Condition):

Giả sử có 2 người dùng (User A và User B) gửi tin nhắn **cùng một lúc**:
1. Server tạo ra **Thread 1** để xử lý tin nhắn của User A.
2. Server tạo ra **Thread 2** để xử lý tin nhắn của User B.

Cả hai Thread này cùng chạy vào hàm `AddMessage` gần như đồng thời:

```csharp
public static void AddMessage(ChatMessage message)
{
    _messages.Add(message); // Nguy hiểm!
}
```

Vấn đề xảy ra với `List<T>` trong C# (vì nó **không thread-safe**):
- **Mất dữ liệu:** Cả hai thread cùng đọc vị trí cuối cùng của mảng bên trong List, cùng ghi đè lên vị trí đó. Kết quả là chỉ có 1 tin nhắn được lưu, tin nhắn kia bị mất.
- **Lỗi IndexOutOfRangeException:** Nếu mảng nội bộ của List cần mở rộng (resize), hai thread cùng thực hiện resize có thể làm hỏng cấu trúc dữ liệu.
- **Lỗi khi đang đọc (Reader-Writer Problem):**
    - Thread 3 đang đọc danh sách để lấy lịch sử chat (`GetMessages`).
    - Thread 1 đang thêm mới (`AddMessage`).
    - => Thread 3 sẽ bị crash ngay lập tức với lỗi: `InvalidOperationException: Collection was modified; enumeration operation may not execute.` (Không được sửa đổi danh sách khi đang duyệt qua nó).

## 2. Giải pháp: Cơ chế Lock (Khóa)

Để giải quyết vấn đề trên, chúng ta sử dụng từ khóa `lock` trong C#.

```csharp
private static object _lock = new object(); // Chiếc chìa khóa

public static void AddMessage(ChatMessage message)
{
    lock (_lock) // Yêu cầu lấy chìa khóa
    {
        // Vùng an toàn (Critical Section)
        // Chỉ có 1 Thread được vào đây tại một thời điểm
        _messages.Add(message);
    } // Trả chìa khóa, Thread khác mới được vào
}
```

### Cơ chế hoạt động:
Hãy tưởng tượng đoạn code bên trong `lock` là một **phòng thay đồ** chỉ chứa được 1 người, và `_lock` là cái **chìa khóa** duy nhất.

1. **Thread A** đến `lock (_lock)`. Nó thấy chìa khóa đang treo ở đó -> Nó cầm lấy chìa khóa và đi vào phòng.
2. **Thread B** đến `lock (_lock)` ngay sau đó. Nó thấy chìa khóa đã mất -> Nó phải **đứng chờ (block)** ở ngoài cửa.
3. **Thread A** thực hiện xong việc thêm tin nhắn -> Đi ra khỏi phòng và trả lại chìa khóa (`Monitor.Exit`).
4. **Thread B** thấy chìa khóa -> Cầm lấy và đi vào thực hiện công việc của mình.

Điều này đảm bảo rằng **không bao giờ** có 2 luồng cùng chỉnh sửa danh sách `_messages` cùng một lúc.

## 3. Tại sao cần Lock cả khi Đọc (`GetMessages`)?

```csharp
public static List<ChatMessage> GetMessages()
{
    lock (_lock) // Cũng cần khóa khi đọc!
    {
        return _messages.OrderBy(x => x.Timestamp).ToList();
    }
}
```

Bạn có thể thắc mắc: "Đọc thôi mà, có sửa gì đâu mà cần lock?".

Lý do là để tránh lỗi **"Đang đọc thì bị sửa"** đã nhắc ở trên.
- Nếu `GetMessages` không có lock: Nó đang chạy vòng lặp để copy dữ liệu ra (`ToList()`).
- Trong lúc đó, `AddMessage` chạy ở thread khác và thay đổi cấu trúc của `_messages`.
- Kết quả: `GetMessages` sẽ crash hoặc copy ra dữ liệu rác.

Bằng cách dùng chung một object `_lock` cho cả hàm Đọc và hàm Ghi, chúng ta đảm bảo rằng **nếu có ai đang ghi, thì không ai được đọc** (và ngược lại).

## 4. Tóm lại

| Thành phần | Vai trò |
| :--- | :--- |
| `static List<T>` | Tài nguyên dùng chung (Shared Resource), không an toàn đa luồng. |
| `static object _lock` | Token đồng bộ hóa (như chiếc chìa khóa duy nhất). |
| `lock (_lock) { ... }` | Đảm bảo đoạn code bên trong chỉ chạy tuần tự (Serial execution), từng người một. |

Đây là kỹ thuật cơ bản nhất để xử lý đa luồng (Concurrency Control) trong C#. Trong các hệ thống lớn hơn, người ta có thể dùng Database hoặc Redis để thay thế, vì `lock` chỉ có tác dụng trên 1 server vật lý (nếu deploy nhiều server thì cách này không dùng được).

## 5. Ví dụ thực tế: Chuyện gì xảy ra khi 2 người cùng chat?

Giả sử **User A (Alice)** và **User B (Bob)** nhấn nút gửi tin nhắn **chính xác cùng một thời điểm** (cùng mili-giây).

### Diễn biến tại Server:

1.  **Tiếp nhận:** Server nhận được 2 request gần như đồng thời. Nó giao cho **Thread A** xử lý tin của Alice và **Thread B** xử lý tin của Bob.
2.  **Đụng độ tại `lock`:**
    *   Cả hai Thread cùng chạy đến dòng code `lock (_lock)`.
    *   Theo nguyên tắc "Ai đến trước phục vụ trước", giả sử **Thread A** nhanh hơn 1 nano-giây -> Thread A **chiếm được khóa** và đi vào trong.
    *   **Thread B** đến chậm hơn -> Thấy khóa đã bị lấy -> Nó bị hệ điều hành **tạm dừng (Block)** và phải đứng chờ ngay tại dòng đó.
3.  **Xử lý tuần tự:**
    *   **Thread A:** Thêm tin nhắn của Alice vào danh sách `_messages`. Sau đó thoát ra khỏi block `lock`.
    *   **Thread B:** Ngay khi A thoát ra, Thread B được "đánh thức", nó lấy khóa, đi vào, và thêm tin nhắn của Bob vào sau tin nhắn của Alice.
4.  **Phản hồi:**
    *   Sau khi thêm vào danh sách xong, cả hai Thread tiếp tục chạy xuống dòng `Clients.All.SendAsync`.
    *   SignalR gửi cả 2 tin nhắn xuống cho tất cả mọi người.

### Kết quả người dùng thấy:

*   **Không có lỗi:** Ứng dụng không bị crash.
*   **Không mất tin:** Cả tin của Alice và Bob đều được lưu.
*   **Độ trễ bằng 0:** Quá trình "chờ khóa" của Thread B diễn ra cực nhanh (tính bằng micro-giây), nên người dùng cảm thấy như 2 tin nhắn xuất hiện tức thời.
*   **Thứ tự:** Nếu Thread A chiếm được khóa trước, tin của Alice nằm trước Bob trong danh sách.
