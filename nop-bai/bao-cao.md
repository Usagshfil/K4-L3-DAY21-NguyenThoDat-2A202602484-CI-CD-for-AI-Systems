# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Nguyễn Thọ Đạt |
| MSSV | 2A202602484 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/Usagshfil/K4-L3-DAY21-NguyenThoDat-2A202602484-CI-CD-for-AI-Systems |
| Ngày nộp | 08/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ siêu tham số ở lần chạy 3 đạt chỉ số `f1_score` cao nhất (0.7149), vượt qua ngưỡng quy định 0.65 của bài toán. Mặc dù lần chạy 1 đạt accuracy cao hơn một chút (0.8780 so với 0.8740), nhưng lần chạy 3 lại tối ưu khả năng phát hiện lớp thiểu số (thu nhập cao) tốt hơn, thể hiện qua điểm F1 vượt trội. Trong mô hình Gradient Boosting, ta quan sát thấy sự đánh đổi rõ rệt: việc tăng số lượng cây (`n_estimators` từ 50 lên 200) kết hợp tăng độ sâu tối đa (`max_depth` từ 2 lên 5) giúp mô hình học sâu hơn các tương tác phi tuyến tính giữa các đặc trưng, khắc phục tình trạng underfitting của lần chạy 2 (F1 chỉ đạt 0.6051 và sẽ bị Quality Gate chặn).

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult Census Income có sự mất cân bằng lớp nghiêm trọng khi chỉ có 24.8% số lượng mẫu thuộc lớp thu nhập cao (target = 1) và có tới 75.2% thuộc lớp thu nhập thấp (target = 0). Do sự chênh lệch này, một mô hình phân loại vô dụng chỉ đơn thuần dự đoán nhãn "thu nhập thấp" cho tất cả mọi trường hợp vẫn có thể đạt mức accuracy cao giả tạo là 75.2% (0.752), trong khi điểm F1 thực tế của lớp dương bằng 0 vì hoàn toàn không nhận diện được bất kỳ người có thu nhập cao nào.

Do đó, chỉ số F1-score của lớp dương là thước đo then chốt vì nó cân bằng điều hòa giữa Precision (độ chính xác khi dự đoán thu nhập cao) và Recall (độ bao phủ các trường hợp thu nhập cao thực tế). Chúng ta tuyệt đối không sử dụng tham số `average="weighted"` hay `average="macro"` khi tính toán `f1_score`, bởi vì trọng số của lớp đa số (75.2%) sẽ kéo mức điểm tổng thể lên cao, làm lu mờ hiệu quả dự đoán trên lớp thiểu số và vô hiệu hóa ý nghĩa kiểm soát của cổng chất lượng (Quality Gate).

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Lỗi thiếu `pkg_resources` và xung đột `SQLAlchemy` khi chạy MLflow cục bộ. | Python 3.12 mặc định bỏ setuptools và SQLAlchemy bản mới làm hỏng các class nội bộ của MLflow 2.13.0. | Cài đặt `setuptools<70` và ghim phiên bản `sqlalchemy<2.0.36` trong requirements. |
| Pipeline GitHub Actions bị lỗi JSON parse khi đọc secret `STORAGE_CREDENTIALS`. | Trích xuất chuỗi JSON trực tiếp trong script bash bị xung đột ký tự nháy đơn/kép. | Truyền secret qua biến môi trường của GitHub Actions runner và dùng Python `ast.literal_eval` để parse an toàn. |
| Service `income-api` trên AWS EC2 báo lỗi unpickle loss function khi khởi động. | Lệch phiên bản `scikit-learn` giữa môi trường GitHub Actions runner (1.4.2) và EC2. | Đồng bộ và cài đặt chính xác phiên bản `scikit-learn==1.4.2` trên máy ảo EC2. |

---

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.8820 |

**Nhận xét:** Khi bổ sung thêm 22.361 mẫu dữ liệu mới ở Bước 3, chỉ số F1 tăng từ 0.7149 lên 0.7354 (+0.0205) và accuracy tăng nhẹ từ 0.8740 lên 0.8820. Mặc dù dữ liệu bổ sung có cùng phân phối xác suất với tập dữ liệu ban đầu, việc tăng gấp đôi quy mô mẫu huấn luyện vẫn giúp Gradient Boosting tinh chỉnh các ranh giới quyết định chính xác hơn trên tập holdout. Điều quan trọng nhất được minh chứng là toàn bộ quy trình CI/CD hoạt động hoàn toàn tự động: chỉ cần cập nhật phiên bản dữ liệu qua DVC và đẩy commit git, hệ thống đã tự động tái huấn luyện, kiểm thử và triển khai phiên bản mô hình mới lên môi trường production mà không cần bất kỳ sự can thiệp thủ công nào.
