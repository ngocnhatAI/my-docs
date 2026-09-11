

### 1 - Prob density function

**Mật độ xác suất (Probability Density - PDF)** cho biết mức độ tập trung tương đối của một biến ngẫu nhiên liên tục tại từng vùng giá trị khác nhau.

- Khả năng xuất hiện tương đối (Relative Likelihood): Giá trị $f(x)$ tại điểm $x$ càng cao thì biến ngẫu nhiên càng có nhiều khả năng rơi vào vùng lân cận điểm $x$ đó.

- Xác suất trong một khoảng (Interval Probability): Diện tích dưới đường cong $f(x)$ trong khoảng $[a, b]$ chính là xác suất biến ngẫu nhiên nhận giá trị trong khoảng đó:  $P(a \le X \le b) = \int_{a}^{b} f(x)\,dx$

- Hình dạng và đặc trưng phân phối: Đường cong PDF cho biết vị trí tập trung nhiều nhất (Mode - đỉnh đồ thị), độ đối xứng/lệch (Skewness), và độ phân tán của dữ liệu.

- Quy chuẩn tổng thể: Tổng diện tích dưới toàn bộ đường cong mật độ luôn bằng 1 ($\int_{-\infty}^{\infty} f(x)\,dx = 1$).

**Lưu ý quan trọng:**

- $f(x)$ không phải là xác suất tại điểm $x$ 

- Giá trị $f(x)$ có thể lớn hơn 1, miễn là tổng tích phân trên toàn miền bằng 1.

<img title="" src="https://scontent.fhan3-2.fna.fbcdn.net/v/t1.6435-9/132398988_4409805345703219_8664583865233989877_n.png?stp=dst-jpg_tt6&cstp=mx1920x1080&ctp=s1920x1080&_nc_cat=107&ccb=1-7&_nc_sid=9eae26&_nc_eui2=AeHZXambmnyTXF1RMyyJmOPWAGM6sfJxTacAYzqx8nFNp24Pj-4qRp9s7MFHYw966UqADqWEXxom3Ae-xERGQqX3&_nc_ohc=t4XRq1q97q4Q7kNvwG90aPa&_nc_oc=Adr0t-i5cVKxVOo3sSYJKVN7kT8a7gJDj2Y7hoh071yOVwLlVtJDY7B6ilJ4cQwDO5znFcGNpyGbEl6XNQ__eUQy&_nc_zt=23&_nc_ht=scontent.fhan3-2.fna&_nc_gid=tBGZqJsVbpGnWwD9J0eMIw&_nc_ss=782a8&oh=00_AQJ4ulU3T5wtiB0w48Ot8zJCMUi5X6KEt8YdAYZ5LRPyAg&oe=6AC98F92" alt="Không có mô tả ảnh." width="533" data-align="center">

### 2 - Ma trận hiệp phương sai

Trong không gian $d$ chiều ($d$-D), phân phối Gaussian mở rộng từ biến đơn sang vectơ ngẫu nhiên $\mathbf{x} \in \mathbb{R}^d$.

* Trung bình ($\mu$): Trong $d$-D, mỗi chiều đúng là có một $\mu_i$ riêng. Tập hợp lại thành vectơ trung bình:  $\boldsymbol{\mu} = \begin{bmatrix} \mu_1 \\ \mu_2 \\ \vdots \\ \mu_d \end{bmatrix} \in \mathbb{R}^d$

* Độ phân tán ($\sigma$ vs. $\mathbf{\Sigma}$): Trong 1-D chỉ có một $\sigma^2$ (phương sai). Trong $d$-D, không chỉ từng chiều biến thiên mà các chiều còn có thể tương quan với nhau. Do đó, $\sigma^2$ được thay bằng Ma trận hiệp phương sai (Covariance Matrix) $\mathbf{\Sigma} \in \mathbb{R}^{d \times d}$.

            $\mathbf{\Sigma} = \begin{bmatrix} \operatorname{Var}(X_1) & \operatorname{Cov}(X_1, X_2) & \dots & \operatorname{Cov}(X_1, X_d) \\ \operatorname{Cov}(X_2, X_1) & \operatorname{Var}(X_2) & \dots & \operatorname{Cov}(X_2, X_d) \\ \vdots & \vdots & \ddots & \vdots \\ \operatorname{Cov}(X_d, X_1) & \operatorname{Cov}(X_d, X_2) & \dots & \operatorname{Var}(X_d) \end{bmatrix}$

* Trong đó:
  * Đường chéo chính: Là phương sai riêng của từng chiều ($\sigma_1^2, \sigma_2^2, \dots, \sigma_d^2$).
  * Các phần tử ngoài đường chéo: Là hiệp phương sai giữa 2 chiều bất kỳ ($\operatorname{Cov}(X_i, X_j)$), phản ánh mức độ biến thiên tuyến tính giữa các chiều
  * Hàm mật độ tổng hợp (Multivariate Normal PDF):

                $f(\mathbf{x}) = \frac{1}{\sqrt{(2\pi)^d \vert{}\mathbf{\Sigma}\vert{}}} \exp\left( -\frac{1}{2} (\mathbf{x} - \boldsymbol{\mu})^T \mathbf{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu}) \right)$

*(Trong đó $\vert{}\mathbf{\Sigma}\vert{}$ là định thức, thay thế cho $\sigma$; số hạng bậc 2 chuyển thành dạng toàn phương $(\mathbf{x}-\boldsymbol{\mu})^T \mathbf{\Sigma}^{-1} (\mathbf{x}-\boldsymbol{\mu})$).*

- $\mathbf{\Sigma} = \sigma^2 \mathbf{I}$ (Isotropic Gaussian) khi thỏa mãn 2 điều kiện
  
  - Độc lập thống kê: Tất cả các chiều hoàn toàn không tương quan với nhau $\implies \operatorname{Cov}(X_i, X_j) = 0$ với mọi $i \neq j$ (ma trận đường chéo).
  
  - Đẳng hướng (Isotropic): Phương sai trên tất cả các chiều bằng nhau: $\sigma_1^2 = \sigma_2^2 = \dots = \sigma_d^2 = \sigma^2$.
