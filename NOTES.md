# Ghi chú phân tích · Day 04

Họ và tên: Phạm Hồng Khánh Huy
MSSV: 2A202603588

## 1. Case được chọn

- ID case: A1
- Câu trả lời của agent:
> "Dưới đây là tóm tắt các sự cố liên quan đến GitHub Actions và Cloudflare Spectrum, bao gồm trạng thái và phạm vi của từng sự cố:
> 
> ### GitHub Actions
> 1. **Incident: Degraded performance for multiple GitHub services**
>    - **Status:** Resolved
>    - **Created At:** 2026-11-05
>    - **Updated At:** 2026-11-06
>    - **Components Affected:** Git Operations, Issues, Actions, Pages
>    - **Details:** Sự cố này đã dẫn đến hiệu suất giảm cho nhiều dịch vụ GitHub, bao gồm Actions.
>    - **URL:** [Chi tiết sự cố](https://stspg.io/s5bs6jn41fwh)
> 
> 2. **Incident: Incident with Actions**
>    - **Status:** Resolved
>    - **Created At:** 2026-11-05
>    - **Updated At:** 2026-11-05
>    - **Components Affected:** Actions
>    - **Details:** Cải thiện tình trạng giảm hiệu suất cho Actions đã được thực hiện. 
>    - **URL:** [Chi tiết sự cố](https://stspg.io/lv4q3xm9r0rk)
> 
> 3. **Incident: Incident with Actions**
>    - **Status:** Resolved
>    - **Created At:** 2026-11-04
>    - **Updated At:** 2026-11-07
>    - **Components Affected:** Actions
>    - **Details:** Đã có một sự cố khác liên quan đến Actions, liên kết để xem tóm tắt cùng với sự cố khác.
>    - **URL:** [Chi tiết sự cố](https://stspg.io/7pwv28hvkmz8)
> 
> 4. **Incident: Incident with Actions and Pages**
>    - **Status:** Resolved
>    - **Created At:** 2026-11-04
>    - **Updated At:** 2026-11-07
>    - **Components Affected:** Webhooks, Actions, Pages
>    - **Details:** Các tác động đến Actions và Pages đã được duy trì và theo dõi.
>    - **URL:** [Chi tiết sự cố](https://stspg.io/pjgptg3cfc6n)
> 
> 5. **Incident: Incident with Pull Requests, Git Operations and Issues**
>    - **Status:** Resolved
>    - **Created At:** 2026-11-04
>    - **Updated At:** 2026-11-04
>    - **Components Affected:** Git Operations, API Requests, Issues, Pull Requests, Actions
>    - **Details:** Đã điều tra giảm hiệu suất cho nhiều tính năng.
>    - **URL:** [Chi tiết sự cố](https://stspg.io/18blsgwnbyy6)
> 
> ### Cloudflare Spectrum
> 1. **Incident: Spectrum Availability Issue in Phoenix**
>    - **Status:** Monitoring
>    - **Created At:** 2026-11-07
>    - **Updated At:** 2026-11-07
>    - **Components Affected:** Spectrum
>    - **Details:** Cloudflare đang điều tra các vấn đề về khả dụng liên quan đến Spectrum tại Phoenix. Có thể ảnh hưởng đến các ứng dụng Spectrum hiện có.
>    - **URL:** [Chi tiết sự cố](https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1)
> 
> ### Tóm tắt
> - **GitHub Actions** có một số sự cố đã được giải quyết liên quan đến hiệu suất giảm đối với nhiều tính năng khác nhau. 
> - **Cloudflare Spectrum** hiện đang trong trạng thái giám sát cho một sự cố về khả dụng ở một vị trí cụ thể.
> 
> Thông tin được lấy từ [GitHub Status](https://www.githubstatus.com) và [Cloudflare Status](https://www.cloudflarestatus.com)."
- Điểm ở lần chạy 1: 75/100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
Step 1: get_status with {'provider': 'github', 'component': 'Actions'}
Step 1: get_status with {'provider': 'cloudflare', 'component': 'Spectrum'}
Step 2: get_incident_details with {'provider': 'github', 'incident_id': 'pfbmpxxfq1cy'}
Step 2: get_incident_details with {'provider': 'github', 'incident_id': 't89g2q1800fr'}
Step 2: get_incident_details with {'provider': 'github', 'incident_id': 'h02n9572mx9y'}
Step 2: get_incident_details with {'provider': 'github', 'incident_id': 'xr3myzj7ms51'}
Step 2: get_incident_details with {'provider': 'github', 'incident_id': 'smtdswd1h2w4'}
Step 2: get_incident_details with {'provider': 'cloudflare', 'incident_id': 'c60k3g4vsoc1'}
```

**Nguyên nhân có thể xảy ra:**
- Đề bài yêu cầu: *"đối chiếu cả hai dịch vụ... phân biệt trạng thái và phạm vi của từng incident trong dữ liệu... dẫn URL cho cả hai"*.
- Mặc dù Agent đã gọi đầy đủ các tool ở 2 provider, nhưng câu trả lời ban đầu trình bày dàn trải các incident lịch sử của GitHub Actions mà chưa phân tách rõ trạng thái hiện tại (GitHub Actions đang All Systems Operational hoàn toàn, còn Cloudflare Spectrum đang bị Partial System Outage trong trạng thái Monitoring tại Phoenix).
- Thiếu cơ chế kiểm soát nguồn chặt chẽ (Evidence Gate) khiến Agent không kiểm tra tính đầy đủ của bằng chứng và phạm vi thực tế trước khi kết luận.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - Tại `agent.py`: Triển khai cơ chế kiểm soát nguồn **Evidence Gate** (`observed_urls`).
  - Khi tool `get_incident_details` được gọi thành công, tự động lưu URL sự cố vào tập hợp `observed_urls`.
  - Tại bước kết luận (`finish == 'stop'`), kiểm tra xem câu trả lời của Model có chứa các liên kết URL nguồn từ `observed_urls` hay không. Nếu chưa đạt, ghi nhận sự kiện `validation_missing_source` vào trace và gửi prompt yêu cầu Model chỉnh sửa, bổ sung nguồn và đối chiếu bằng chứng trong phạm vi ngân sách bước (`max_steps`).
- Vì sao chọn thay đổi này:
  - Đây là cơ chế tổng quát theo đúng hợp đồng kiểm thử của `tests/test_validation.py`.
  - Buộc Agent phải đối chiếu trực tiếp dữ liệu thô từ các tool đã quan sát, tăng tính xác thực (Factuality) và bằng chứng (Evidence), loại bỏ tình trạng kết luận vội vã hoặc thiếu căn cứ.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 79.77 | 76.59 |
| Điểm case đã chọn | 75 | 100 |
| Số model calls | 50 | 51 |
| Token, chi phí nếu có | 40,832 tokens | 46,856 tokens |

- Lỗi đã giải quyết:
  - Case A1 tăng từ **75 lên 100 điểm tuyệt đối**.
  - Giám khảo (LLM Judge) đánh giá: *"Câu trả lời cung cấp thông tin chính xác và đầy đủ về sự cố của GitHub Actions và Cloudflare Spectrum, phân biệt rõ giữa các sự cố đã được giải quyết và sự cố đang được theo dõi. Nó dẫn chứng rõ ràng các URL quy định cho mỗi sự cố."*
  - Case H2 cũng tăng ấn tượng từ 75 lên 100 điểm.
- Lỗi còn lại hoặc mới xuất hiện:
  - Một số case mang tính chất ngẫu nhiên của LLM như A5 bị giảm điểm do câu trả lời chưa trúng trọng tâm mong đợi của Judge trong lần chạy thứ 2.
  - Case H6 vẫn bị chặn trần 60 điểm do model gọi song song cả 2 tools ở Step 1 thay vì tách ra làm 2 model steps riêng biệt.

## 5. Giới hạn

- **Giới hạn của điểm judge:** LLM Judge mang tính chất ước lượng xác suất, cùng một câu trả lời có thể nhận số điểm chênh lệch nhẹ giữa các lần chấm. Judge có thể bị ảnh hưởng bởi văn phong hoặc độ chi tiết của prompt.
- **Giới hạn của snapshot:** Dữ liệu thử nghiệm là ảnh chụp tĩnh tại một thời điểm quá khứ (tháng 11/2026), không phản ánh trạng thái live tức thời ngoài Internet.
- **Giới hạn của việc chạy một lần:** Chạy 1 lần cho mỗi phiên bản chỉ cho thấy sự thay đổi tương đối trên tập test cố định, chưa đủ để chứng minh độ ổn định thống kê (variance) lâu dài của Agent khi gặp các input đa dạng ngoài thực tế.
