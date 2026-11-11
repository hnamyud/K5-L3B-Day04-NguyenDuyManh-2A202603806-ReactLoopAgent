# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Duy Mạnh
MSSV: 2A202603806

## 1. Case được chọn

- ID case: H2
- Câu trả lời của agent: "Không thể xác nhận rằng GitHub Actions **đang** outage. Theo snapshot lúc **2026-11-09 03:50:47 UTC**, incident `t89g2q1800fr` — “Incident with Actions” — có trạng thái **resolved**. GitHub ghi nhận sự cố ảnh hưởng đến Actions ngày 5/11, sau đó được giảm thiểu và chuyển sang theo dõi trước khi đánh dấu đã giải quyết lúc **17:42:35 UTC** ngày 5/11. 【Nguồn: [chi tiết incident trên GitHub Status](https://stspg.io/lv4q3xm9r0rk), snapshot do công cụ truy vấn】\n\nThông tin này xác nhận từng có incident liên quan đến component **Actions**, không xác nhận sự cố đang tiếp diễn hoặc ảnh hưởng cụ thể đến tài khoản, workflow hay khu vực của bạn."
- Điểm ở lần chạy 1: 50/100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[
  {
    "event": "model",
    "step": 1,
    "finish": "tool_calls"
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "t89g2q1800fr"
    },
    "result": {
      "ok": true,
      "name": "Incident with Actions",
      "status": "resolved",
      "url": "https://stspg.io/lv4q3xm9r0rk"
    }
  },
  {
    "event": "model",
    "step": 2,
    "finish": "stop"
  }
]
```

Nguyên nhân:
```text
Agent chỉ gọi get_incident_details để kiểm tra incident t89g2q1800fr. Tool trả về incident đã resolved, sau đó model kết thúc ở bước 2 mà không gọi get_status để kiểm tra trạng thái GitHub Actions tại thời điểm snapshot. Vì vậy, câu trả lời có căn cứ về incident cũ nhưng chưa kiểm tra đầy đủ trạng thái hiện tại của component. Nguyên nhân có thể là prompt chưa hướng dẫn rõ việc cần gọi get_status khi người dùng hỏi về trạng thái hiện tại.
```

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong `agent.py`, bổ sung hướng dẫn gọi `get_status` khi kiểm tra trạng thái hiện tại và thêm bước kiểm tra URL nguồn trong câu trả lời.
- Vì sao chọn: Case H2 chưa kiểm tra trạng thái component; bước kiểm tra URL giúp agent nhắc model bổ sung nguồn khi thiếu.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92,05/100 | 95,45/100 |
| Điểm case đã chọn | 50/100 | 100/100 |
| Số model calls | 58 | 56 |
| Token của agent | 53.828 | 57.183 |
| Token của judge | 39.912 | 43.205 |

- Lỗi đã giải quyết: H2 đã kiểm tra trạng thái GitHub Actions và chi tiết
  incident, đồng thời dẫn nguồn cho câu trả lời.
- Lỗi còn lại hoặc mới xuất hiện: H3 vẫn đạt 50/100; agent chưa gọi
  `get_incident_details` để kiểm tra ID incident và có nhận định vượt quá
  bằng chứng được trả về.

## 5. Giới hạn

- Điểm LLM judge chỉ mang tính tham khảo và có thể đánh giá chưa chính xác.
- Snapshot là dữ liệu tại thời điểm được lưu, không phản ánh trạng thái hiện tại.
- Mỗi phiên bản chỉ được chạy một lần trên toàn bộ bộ eval nên chưa thể kết luận
  kết quả cải thiện ổn định.
- Kiểm tra URL chỉ xác nhận câu trả lời có chứa nguồn đã quan sát, không chứng
  minh mọi thông tin trong câu trả lời đều đúng hoặc được nguồn hỗ trợ.
