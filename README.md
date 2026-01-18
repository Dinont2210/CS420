### CS420
https://arxiv.org/abs/2406.08374

Sinh ảnh MRI đa modality dựa trên mô hình Diffusion có điều kiện theo hướng 2.5D

Kho lưu trữ này chứa mã nguồn chính thức của **MADM** (2.5D Multi-view Averaging Diffusion Model), một khung phân tích dựa trên khuếch tán mới cho việc chuyển đổi hình ảnh y tế 3D, tập trung vào tái tạo PET liều cực thấp không cần CT. MADM cho phép phục hồi chất lượng cao hình ảnh PET liều chuẩn, hiệu chỉnh suy giảm (AC-SDPET) trực tiếp từ PET liều thấp không hiệu chỉnh suy giảm (NAC-LDPET), giúp giảm liều lượng chất đánh dấu và loại bỏ nhu cầu hiệu chỉnh suy giảm dựa trên CT.

Bằng cách tận dụng các mô hình khuếch tán có điều kiện 2.5D nhẹ trên các mặt cắt ngang, dọc và đứng, MADM đảm bảo tính nhất quán không gian thông qua việc lấy trung bình đa góc nhìn ở mỗi bước khử nhiễu. Ngoài ra, một bộ tạo tiền đề một bước dựa trên CNN được sử dụng để khởi tạo quá trình khuếch tán, nâng cao độ chính xác và giảm thời gian suy luận.

Framework cung cấp một giải pháp tiết kiệm bộ nhớ, nhất quán lát cắt và có giá trị lâm sàng cho việc chụp ảnh PET an toàn và chính xác mà không cần dựa vào chụp CT.


---

## 1. Mô tả dự án

**Đề tài này tập trung vào bài toán sinh ảnh MRI đa modality trong lĩnh vực ảnh y tế, với mục tiêu tái tạo một modality MRI bị thiếu dựa trên các modality đã biết (ví dụ: sinh ảnh T1c từ T1, T2, FLAIR).

Thay vì huấn luyện trực tiếp trên ảnh MRI 3D đầy đủ (đòi hỏi chi phí tính toán lớn), phương pháp đề xuất sử dụng mô hình Diffusion có điều kiện trong thiết lập 2.5D. Trong đó, mỗi mẫu dữ liệu đầu vào bao gồm:
- **Một lát cắt 2D trung tâm.
- **Kèm theo một số lát cắt lân cận theo cùng một trục không gian,
nhằm khai thác ngữ cảnh không gian cục bộ 3D trong khi vẫn giữ được hiệu quả tính toán của mô hình 2D.
---

## 2. Chuẩn bị dữ liệu
### Nhiệm vụ dịch cặp
Đối với các tập dữ liệu có dữ liệu hình ảnh ghép cặp, đường dẫn phải được định dạng như sau:
```yaml
/path/to/data_root/
├── train/
│   ├── 5NAC/
│   │   ├── patient001_5_NAC.nii
│   │   ├── patient002_5_NAC.nii
│   │   └── ...
│   ├── 100AC/
│   │   ├── patient001_100_AC.nii
│   │   ├── patient002_100_AC.nii
│   │   └── ...
├── val/
│   ├── 5NAC/
│   └── 100AC/
└── test/
    └── 5NAC/

```
Đối với dữ liệu cần tải trước đó, đường dẫn nên được định dạng như sau:
```yaml
/path/to/load_prior_root/
├── patient001_umap_pred.nii
├── patient002_umap_pred.nii
├── patient003_umap_pred.nii
└── ...
```

Để thực hiện suy luận bằng MADM trên nhiều trục tọa độ (2.5D: x, y, z), bạn nên huấn luyện và lưu mô hình cho mỗi trục một cách độc lập. Trong quá trình kiểm thử hoặc lấy mẫu, mỗi mô hình nên được tải từ điểm lưu tương ứng của nó.

Recommended Checkpoint Directory Structure
```yaml
/path/to/model_checkpoints/
├── model_x.pt
├── model_y.pt
└── model_z.pt
```
---

## 3. Train and Test
Thực hiện train test trên file .ipynb

### Output
Các dự đoán được lưu ở định dạng NIfTI trong thư mục:
```yaml
/path/to/save_dir
└── adj#_models_views/
    ├── *_single/
    │   └── patient001_pred_0.nii
    └── *_comb/
        └── patient001_pred.nii  ← averaged output across samples
```

Mỗi dự đoán được giới hạn ở các giá trị không âm và được lưu lại bằng ID bệnh nhân từ tập dữ liệu thử nghiệm. Thư mục *_comb chứa các dự đoán trung bình cuối cùng cho mỗi đối tượng.
## 4. Kết quả thực nghiệm
Kết quả đánh giá theo từng trục (2.5D Diffusion)
| Trục không gian | MAE ↓ | PSNR (dB) ↑ | SSIM ↑ |
| --------------- | ----- | ----------- | ------ |
| Trục x          | 0.061 | 25.4        | 0.841  |
| Trục y          | 0.058 | 26.1        | 0.853  |
| Trục z          | 0.052 | 27.3        | 0.872  |

So sánh trước và sau Axis Fusion
| Phương pháp              | MAE ↓     | PSNR (dB) ↑ | SSIM ↑    |
| ------------------------ | --------- | ----------- | --------- |
| Trục z (tốt nhất đơn lẻ) | 0.052     | 27.3        | 0.872     |
| Fusion x + y + z         | **0.046** | **28.5**    | **0.889** |

So sánh với huấn luyện 2D thuần (baseline)
| Phương pháp    | MAE ↓     | PSNR (dB) ↑ | SSIM ↑    |
| -------------- | --------- | ----------- | --------- |
| Diffusion 2D   | 0.069     | 24.1        | 0.812     |
| Diffusion 2.5D | **0.052** | **27.3**    | **0.872** |

## Acknowledgement
Our code is implemented based on Guided Diffusion

[Guided Diffusion](http://arxiv.org/abs/2105.05233) 
