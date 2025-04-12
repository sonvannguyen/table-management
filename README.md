1. Khởi Tạo Dữ Liệu
    Hàm generateInitialData() tạo bảng cells kích thước 10 hàng x 8 cột.

    Mỗi ô có id và value mặc định.

2. Quản Lý Sự Kiện Chọn Ô
    Click ô: gọi selectCell(id), set selectedCell và focus input.

    Kéo chọn nhiều ô:

        handleMouseDown(row, col): bắt đầu kéo (isDragging = true, lưu dragStart).

        handleMouseMove(row, col): cập nhật selectedRange.

        handleMouseUp(): kết thúc kéo (isDragging = false).

3. Cập Nhật Giá Trị Ô
    Hàm updateCell(id, value):

    Cập nhật nội dung ô.

    Ghi lại trạng thái trước thay đổi vào undoStack để hỗ trợ undo.

4. Copy - Paste
    Copy:

    Ctrl+C → copySelectedRange() → Ghi giá trị các ô đã chọn vào clipboard.

    Paste:

    Ctrl+V → navigator.clipboard.readText() → pasteContent(content) → Ghi đè dữ liệu và lưu trạng thái.

5. Undo - Redo
    Undo:

    Ctrl+Z → Phục hồi từ undoStack.

    Redo:

    Ctrl+Y → Phục hồi từ redoStack.

6. Điều Hướng Bằng Phím
    Sử dụng các phím Arrow keys, Tab, Enter để di chuyển selectedCell.

7. Lắng Nghe Sự Kiện Toàn Cục
    Dùng useEventListener để lắng nghe:

    keydown (quản lý phím tắt)

    mouseup (kết thúc kéo chọn)