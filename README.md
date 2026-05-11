# Bai_Tap_3
Họ tên: Bùi Xuân Dũng

MSSV: K235480106009

Lớp: K59KMT.K01

Đề tài: THIẾT KẾ VÀ CÀI ĐẶT CSDL QUẢN LÝ CẦM ĐỒ 

- Mô tả nghiệp vụ tổng quát

Hệ thống quản lý các thực thể chính: Khách hàng, Hợp đồng, Tài sản và Lịch sử giao dịch.

Lãi suất được tính theo cơ chế: Lãi đơn trong thời hạn (Deadline 1), sau đó chuyển sang lãi kép (lãi chồng lãi) để thúc đẩy khách hàng trả nợ sớm.

Tài sản được quản lý chặt chẽ theo trạng thái từ lúc cầm cố đến khi thanh lý.
# Nhiệm vụ 1: Thiết kế CSDL 
```
CREATE DATABASE QuanLyCamDo;
GO
USE QuanLyCamDo;
GO

-- 1. Bảng Khách hàng
CREATE TABLE KhachHang (
    MaKH INT PRIMARY KEY IDENTITY(1,1),
    HoTen NVARCHAR(100) NOT NULL,
    SoDienThoai VARCHAR(15),
    DiaChi NVARCHAR(255)
);

-- 2. Bảng Hợp đồng
CREATE TABLE HopDong (
    MaHD INT PRIMARY KEY IDENTITY(1,1),
    MaKH INT FOREIGN KEY REFERENCES KhachHang(MaKH),
    SoTienVayGoc DECIMAL(18, 2) NOT NULL,
    NgayLap DATETIME DEFAULT GETDATE(),
    Deadline1 DATETIME NOT NULL, -- Mốc tính lãi đơn -> lãi kép
    Deadline2 DATETIME NOT NULL, -- Mốc bắt đầu thanh lý
    TrangThai NVARCHAR(50) DEFAULT N'Dang vay', -- 'Dang vay', 'Qua han', 'Da thanh toan', 'Da thanh ly'
    DuNoGocHienTai DECIMAL(18, 2) -- Dùng để tracking số tiền còn nợ sau khi trả một phần
);

-- 3. Bảng Tài sản
CREATE TABLE TaiSan (
    MaTS INT PRIMARY KEY IDENTITY(1,1),
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    TenTaiSan NVARCHAR(255),
    GiaTriDinhGia DECIMAL(18, 2),
    TrangThai NVARCHAR(50) DEFAULT N'Dang cam co', -- 'Dang cam co', 'Da tra khach', 'Da ban thanh ly'
    IsSold BIT DEFAULT 0
);

-- 4. Bảng Log giao dịch (Audit Log)
CREATE TABLE TransactionLog (
    MaLog INT PRIMARY KEY IDENTITY(1,1),
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    NgayGiaoDich DATETIME DEFAULT GETDATE(),
    SoTienTra DECIMAL(18, 2),
    NoiDung NVARCHAR(255)
);
```
<img width="1920" height="1020" alt="SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - Microsoft SQL Server Management Studio 5_11_2026 2_10_38 AM" src="https://github.com/user-attachments/assets/79ab9468-a49f-47c7-8dff-acfd4fd7627d" />
Tạo database và 4 bảng dữ liệu chuẩn hoá 3NF
<img width="1920" height="1020" alt="SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - Microsoft SQL Server Management Studio 5_11_2026 2_13_55 AM" src="https://github.com/user-attachments/assets/f20a0abe-cf66-4249-8a35-83abf8cdd56a" />
Sơ đồ thực thể liên kết ERD

# Nhiệm vụ 2: Cài đặt SQL
### Event 1: Đăng ký hợp đồng mới

- **Mục tiêu**: Tiếp nhận khách hàng và thiết lập khoản vay ban đầu

- **Nghiệp vụ**: Lưu thông tin định danh khách hàng (nếu khách mới) hoặc liên kết mã khách hàng cũ

    Ghi nhận số tiền gốc vay, danh sách tài sản kèm giá trị định giá của từng món

     Thiết lập 2 mốc quan trọng: **Deadline 1** (Hết hạn lãi đơn) và **Deadline 2** (Bắt đầu được phép thanh lý tài sản).

- **Logic dữ liệu**: Khởi tạo trạng thái hợp đồng là `Dang vay`
```
CREATE PROCEDURE sp_RegisterContract
    @MaKH INT,
    @SoTienVay DECIMAL(18,2),
    @Deadline1 DATETIME,
    @Deadline2 DATETIME,
    @DanhSachTaiSan NVARCHAR(MAX)
AS
BEGIN
    INSERT INTO HopDong (MaKH, SoTienVayGoc, Deadline1, Deadline2, DuNoGocHienTai)
    VALUES (@MaKH, @SoTienVay, @Deadline1, @Deadline2, @SoTienVay);
    
END;
```
<img width="1920" height="1020" alt="SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - Microsoft SQL Server Management Studio 5_11_2026 2_21_10 AM" src="https://github.com/user-attachments/assets/d9e40dab-0377-4734-b31e-4f17b231997a" />
### Event 2: Tính toán công nợ thời gian thực 

- **Mục tiêu**: Xác định chính xác số tiền khách đang nợ tại bất kỳ thời điểm nào

- **Cơ chế tính lãi**:

  Giai đoạn 1 (Trước Deadline 1): Lãi đơn tính theo công thức: Số tiền gốc x Lãi suất x Số ngày

  Giai đoạn 2 (Sau Deadline 1): Lãi kép tính trên tổng (Gốc + Lãi đơn đã tích lũy). Sử dụng hàm lũy thừa $A = P(1 + r)^n$.

- **Logic dữ liệu**: Số tiền phải trả cuối cùng = `Tổng gốc và lãi` - `Tổng số tiền khách đã trả`
```
-- Function tính tổng tiền phải trả (Gốc + Lãi đơn + Lãi kép)
CREATE FUNCTION fn_CalcMoneyContract (@MaHD INT, @TargetDate DATETIME)
RETURNS DECIMAL(18, 2)
AS
BEGIN
    DECLARE @Goc DECIMAL(18, 2), @NgayVay DATETIME, @DL1 DATETIME;
    DECLARE @TongTien DECIMAL(18, 2);
    DECLARE @DaysDon INT, @DaysKep INT;
    DECLARE @LaiSuatNgay FLOAT = 0.005; -- 5000 / 1.000.000

    SELECT @Goc = SoTienVayGoc, @NgayVay = NgayLap, @DL1 = Deadline1 
    FROM HopDong WHERE MaHD = @MaHD;

    -- Nếu ngày tính toán nhỏ hơn ngày vay
    IF (@TargetDate <= @NgayVay) RETURN @Goc;

    -- Tính số ngày lãi đơn (từ ngày vay đến DL1 hoặc TargetDate)
    IF (@TargetDate <= @DL1)
    BEGIN
        SET @DaysDon = DATEDIFF(DAY, @NgayVay, @TargetDate);
        SET @TongTien = @Goc * (1 + @DaysDon * @LaiSuatNgay);
    END
    ELSE
    BEGIN
        -- Giai đoạn lãi đơn (đến DL1)
        SET @DaysDon = DATEDIFF(DAY, @NgayVay, @DL1);
        DECLARE @VaySauDL1 DECIMAL(18,2) = @Goc * (1 + @DaysDon * @LaiSuatNgay);
        
        -- Giai đoạn lãi kép (từ DL1 đến TargetDate)
        SET @DaysKep = DATEDIFF(DAY, @DL1, @TargetDate);
        -- Công thức lãi kép: A = P * (1 + r)^n
        SET @TongTien = @VaySauDL1 * POWER((1 + @LaiSuatNgay), @DaysKep);
    END

    -- Trừ đi số tiền khách đã trả trong quá khứ
    DECLARE @DaTra DECIMAL(18,2) = (SELECT ISNULL(SUM(SoTienTra), 0) FROM TransactionLog WHERE MaHD = @MaHD);
    
    RETURN @TongTien - @DaTra;
END;
```
<img width="1920" height="1020" alt="SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - Microsoft SQL Server Management Studio 5_11_2026 2_36_41 AM" src="https://github.com/user-attachments/assets/21cfbaf9-cb2c-4507-a1d3-428c9c11fe82" />

### Event 3: Xử lý trả nợ và hoàn trả tài sản 
- **Mục tiêu**: Ghi nhận tiền khách trả và quyết định việc trả lại tài sản thế chấp

- **Quy tắc**:

  Chặn thanh toán: Nếu tài sản đã bị thanh lý, hệ thống từ chối thu tiền

  Trả nợ từng phần: Cho phép trả một phần nợ, hệ thống cập nhật vào bảng Log và đổi trạng thái hợp đồng thành `Dang tra gop`

- **Điều kiện lấy tài sản**: Chỉ cho phép khách lấy lại tài sản khi tổng giá trị định giá của các tài sản còn lại vẫn lớn hơn hoặc bằng dư nợ hiện tại. Nếu trả hết tiền, chuyển sang `Da thanh toan du`
```
CREATE PROCEDURE sp_XuLyNo
    @MaHD INT,
    @SoTienNop DECIMAL(18,2)
AS
BEGIN
    -- 1. Kiểm tra tài sản đã bị thanh lý chưa
    IF EXISTS (SELECT 1 FROM TaiSan WHERE MaHD = @MaHD AND IsSold = 1)
    BEGIN
        PRINT N'Tài sản đã bị thanh lý, không thu tiền.';
        RETURN;
    END

    -- 2. Cập nhật Log
    INSERT INTO TransactionLog (MaHD, SoTienTra, NoiDung) VALUES (@MaHD, @SoTienNop, N'Khách trả tiền');

    -- 3. Tính toán tổng nợ hiện tại
    DECLARE @TongNo DECIMAL(18,2) = dbo.fn_CalcMoneyContract(@MaHD, GETDATE());

    IF @TongNo <= 0
    BEGIN
        UPDATE HopDong SET TrangThai = N'Da thanh toan du' WHERE MaHD = @MaHD;
        PRINT N'Đã thanh toán đủ. Khách có thể nhận lại toàn bộ tài sản.';
    END
    ELSE
    BEGIN
        UPDATE HopDong SET TrangThai = N'Dang tra gop' WHERE MaHD = @MaHD;
        
        SELECT MaTS, TenTaiSan FROM TaiSan 
        WHERE MaHD = @MaHD AND GiaTriDinhGia >= @TongNo;
    END
END;
```
<img width="1920" height="1020" alt="SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - Microsoft SQL Server Management Studio 5_11_2026 2_47_30 AM" src="https://github.com/user-attachments/assets/c6730df1-1868-456b-9812-eb9eaffa4a13" />

### Event 4: Truy vấn danh sách nợ xấu (Nợ khó đòi) 
- **Mục tiêu**: Lọc ra các khách hàng có rủi ro cao để xử lý thu hồi nợ

- **Điều kiện lọc**: Các hợp đồng có ngày hiện tại đã vượt quá `Deadline 1` mà vẫn ở trạng thái chưa hoàn tất thanh toán

- **Thông tin hiển thị**: Bao gồm thông tin liên lạc, số tiền gốc, số ngày đã quá hạn và dự báo số nợ nếu khách để thêm 1 tháng nữa (giúp nhân viên tư vấn gây áp lực trả nợ)
```
SELECT 
    kh.HoTen, 
    kh.SoDienThoai, 
    hd.SoTienVayGoc,
    DATEDIFF(DAY, hd.Deadline1, GETDATE()) AS SoNgayQuaHan,
    dbo.fn_CalcMoneyContract(hd.MaHD, GETDATE()) AS TongNoHienTai,
    dbo.fn_CalcMoneyContract(hd.MaHD, DATEADD(MONTH, 1, GETDATE())) AS DuKienNoSau1Thang
FROM HopDong hd
JOIN KhachHang kh ON hd.MaKH = kh.MaKH
WHERE hd.TrangThai = N'Qua han (no xau)';
```
<img width="1920" height="1020" alt="SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - Microsoft SQL Server Management Studio 5_11_2026 2_51_24 AM" src="https://github.com/user-attachments/assets/cf5cec13-9af6-4a52-8eef-2e04a06d8ced" />

### Event 5: Quản lý thanh lý tài sản
- **Mục tiêu**: Tự động hóa việc chuyển đổi trạng thái mà không cần thao tác thủ công

- **Các kịch bản tự động**:

  **Quá hạn 1**: Tự động chuyển Hợp đồng sang `No xau` khi vượt `Deadline 1`

  **Quá hạn 2**: Tự động chuyển Tài sản sang `San sang thanh ly` khi vượt `Deadline 2`

  **Kết thúc**: Khi nhân viên bấm thanh lý hợp đồng, tất cả tài sản liên quan tự động chuyển thành `Da ban thanh ly`
  ```
  CREATE TRIGGER trg_CapNhatNoXau
ON HopDong
AFTER UPDATE, INSERT
AS
BEGIN
    UPDATE HopDong
    SET TrangThai = N'Quá hạn (nợ xấu)'
    FROM HopDong
    INNER JOIN inserted i ON HopDong.MaHD = i.MaHD
    WHERE HopDong.TrangThai = N'Đang vay' 
      AND GETDATE() > HopDong.Deadline1; 
END;
```


```
CREATE TRIGGER trg_SanSangThanhLy
ON HopDong
AFTER UPDATE
AS
BEGIN
    IF EXISTS (SELECT 1 FROM inserted WHERE TrangThai = N'Quá hạn (nợ xấu)' AND GETDATE() > Deadline2) 
    BEGIN
        UPDATE TaiSan
        SET TrangThai = N'Sẵn sàng thanh lý'
        FROM TaiSan
        INNER JOIN inserted i ON TaiSan.MaHD = i.MaHD
        WHERE i.TrangThai = N'Quá hạn (nợ xấu)' 
          AND GETDATE() > i.Deadline2;
    END
END;
```
<img width="1920" height="1020" alt="SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - SQLQuery2 sql - ADMIN-PC QLCD (ADMIN-PC_Admin (86))_ - Microsoft SQL Server Management Studio 5_11_2026 3_02_06 AM" src="https://github.com/user-attachments/assets/9be5171c-c357-4a41-a1fc-b1997d980e6f" />














