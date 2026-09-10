### 1 - Prob density function

**Mật độ xác suất (Probability Density - PDF)** cho biết mức độ tập trung tương đối của một biến ngẫu nhiên liên tục tại từng vùng giá trị khác nhau.

- **Khả năng xuất hiện tương đối (Relative Likelihood):** Giá trị $f(x)$ tại điểm $x$ càng cao thì biến ngẫu nhiên càng có nhiều khả năng rơi vào vùng lân cận điểm $x$ đó.

- **Xác suất trong một khoảng (Interval Probability):** Diện tích dưới đường cong $f(x)$ trong khoảng $[a, b]$ chính là xác suất biến ngẫu nhiên nhận giá trị trong khoảng đó:  $P(a \le X \le b) = \int_{a}^{b} f(x)\,dx$

- **Hình dạng và đặc trưng phân phối:** Đường cong PDF cho biết vị trí tập trung nhiều nhất (Mode - đỉnh đồ thị), độ đối xứng/lệch (Skewness), và độ phân tán của dữ liệu.

- **Quy chuẩn tổng thể:** Tổng diện tích dưới toàn bộ đường cong mật độ luôn bằng 1 ($\int_{-\infty}^{\infty} f(x)\,dx = 1$).

**Lưu ý quan trọng:**

- $f(x)$ **không phải** là xác suất tại điểm $x$ 

- Giá trị $f(x)$ có thể lớn hơn 1, miễn là tổng tích phân trên toàn miền bằng 1.

![Không có mô tả ảnh.](https://scontent.fhan3-2.fna.fbcdn.net/v/t1.6435-9/132398988_4409805345703219_8664583865233989877_n.png?stp=dst-jpg_tt6&cstp=mx1920x1080&ctp=s1920x1080&_nc_cat=107&ccb=1-7&_nc_sid=9eae26&_nc_eui2=AeHZXambmnyTXF1RMyyJmOPWAGM6sfJxTacAYzqx8nFNp24Pj-4qRp9s7MFHYw966UqADqWEXxom3Ae-xERGQqX3&_nc_ohc=t4XRq1q97q4Q7kNvwG90aPa&_nc_oc=Adr0t-i5cVKxVOo3sSYJKVN7kT8a7gJDj2Y7hoh071yOVwLlVtJDY7B6ilJ4cQwDO5znFcGNpyGbEl6XNQ__eUQy&_nc_zt=23&_nc_ht=scontent.fhan3-2.fna&_nc_gid=tBGZqJsVbpGnWwD9J0eMIw&_nc_ss=782a8&oh=00_AQJ4ulU3T5wtiB0w48Ot8zJCMUi5X6KEt8YdAYZ5LRPyAg&oe=6AC98F92)
