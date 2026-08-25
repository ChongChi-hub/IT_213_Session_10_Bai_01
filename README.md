# Bài 1: Phân Tích & Lựa Chọn Giải Pháp Triển Khai Langfuse

## 1. Bảng so sánh 3 phương án triển khai

| Tiêu chí | Phương án A (PostgreSQL Only) | Phương án B (PostgreSQL + ClickHouse) | Phương án C (External PostgreSQL) |
|---|---|---|---|
| **Bảo mật dữ liệu** | Trung bình — Dữ liệu nằm trong Docker container, dễ mất nếu container bị xóa | Trung bình — Tương tự PA, hai database đều nằm trong Docker network | **Tốt nhất** — Dữ liệu nằm trên RDS/server vật lý bên ngoài, tách biệt hoàn toàn khỏi vòng đời container |
| **Chi phí & Tài nguyên** | **Thấp nhất** — Chỉ chạy 1 container PostgreSQL, phù hợp local dev | **Cao nhất** — Chạy thêm ClickHouse (tốn thêm 2-4 GB RAM), phù hợp production lớn | Trung bình — Không tốn thêm RAM cho DB, nhưng cần trả phí RDS hoặc duy trì server |
| **Độ phức tạp triển khai** | **Đơn giản nhất** — Ít cấu hình Docker Compose | **Phức tạp nhất** — Phải cấu hình cả PostgreSQL và ClickHouse, dễ xảy ra lỗi kết nối giữa 2 DB | Trung bình — Cần cấu hình biến môi trường kết nối External DB và đảm bảo network rules đúng |
| **Backup & Recovery** | **Kém nhất** — Không có tầng Backup riêng biệt, chỉ có thể dùng `pg_dump` thủ công | Trung bình — Cần backup riêng cho cả 2 engine DB | **Tốt nhất** — Tận dụng cơ chế Automated Backup, Point-in-Time Recovery, Multi-AZ của AWS RDS |
| **Phù hợp môi trường** | Local / Development | Production lớn (scale-out analytics) | **Production chuẩn doanh nghiệp** |

## 2. Lựa chọn tối ưu: Phương án C

**Phương án C (Langfuse Self-Host kết nối External PostgreSQL)** là lựa chọn tối ưu nhất cho hệ thống RikkeiPay vì những lý do sau:

### Lý do chọn Phương án C

1. **Bảo mật dữ liệu ngân hàng là ưu tiên số 1:** Dữ liệu trace của RikkeiPay chứa thông tin giao dịch tài chính nhạy cảm. Việc lưu trữ trên AWS RDS (với mã hóa at-rest/in-transit, VPC private subnet, IAM authentication) đáp ứng tiêu chuẩn bảo mật ngân hàng nghiêm ngặt hơn bất kỳ phương án nào. Nếu container Langfuse bị xóa hay crash, dữ liệu vẫn an toàn trên RDS.

2. **Backup & Disaster Recovery tự động:** AWS RDS cung cấp automated backup hàng ngày và Point-in-Time Recovery (PITR) lên đến 35 ngày. Điều này cực kỳ quan trọng khi cần audit log giao dịch theo yêu cầu pháp lý của ngân hàng.

3. **Linh hoạt mở rộng:** Khi lưu lượng tăng, chỉ cần scale RDS instance lên tier cao hơn mà không ảnh hưởng đến cấu hình Langfuse containers.

## 3. Nhược điểm của các phương án bị loại bỏ

### Phương án A bị loại vì:
- **Rủi ro mất dữ liệu nghiêm trọng:** Nếu Docker volume bị xóa (thao tác `docker compose down -v`), toàn bộ trace lịch sử mất trắng, không thể audit giao dịch, vi phạm quy định lưu trữ log tối thiểu 5 năm của Ngân hàng Nhà nước.
- Không có tầng bảo mật riêng biệt cho database trong môi trường production.

### Phương án B bị loại vì:
- **Tốn tài nguyên quá lớn không cần thiết:** ClickHouse yêu cầu thêm 2-4GB RAM. Đối với quy mô vừa của RikkeiPay, PostgreSQL đơn thuần đã đủ xử lý analytics traces mà không cần đến ClickHouse.
- Tăng độ phức tạp vận hành: phải maintain 2 database engine khác nhau, tăng nguy cơ lỗi và chi phí DevOps.
