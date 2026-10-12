# Phân tích năng suất cây trồng tại Ấn Độ

Dự án báo cáo giữa kỳ gồm bốn lớp phân tích: mô tả, chẩn đoán, dự đoán và chỉ định. Mục tiêu là khảo sát lịch sử sản xuất, đánh giá khả năng dự đoán năng suất và đề xuất các tổ hợp cây–mùa có bằng chứng hỗ trợ tại từng địa bàn.

Phạm vi: 12 cây do nhóm lựa chọn và tất cả bang/vùng lãnh thổ có trong dữ liệu bàn giao. Số địa bàn được xác nhận từ dữ liệu thực tế; không ép đủ 33 bằng cách tạo dữ liệu. Một bang có trong dữ liệu chung có thể không có quan sát trong validation hoặc test.

Nguồn dữ liệu: [Crop Production in India — Kaggle](https://www.kaggle.com/datasets/abhinand05/crop-production-in-india). Đặt CSV nguồn tại `data/crop_production.csv`.

## 1. Cấu trúc thư mục

```text
crop_project/
├── requirements.txt
├── README.md
├── data/
│   └── crop_production.csv
├── notebooks/
│   ├── 01_cleaning.ipynb
│   ├── 02_descriptive_diagnostic.ipynb
│   ├── 03_model_recommendation.ipynb
│   └── outputs/
│       ├── config.json
│       ├── model_data_top12.csv
│       ├── train.csv
│       ├── validation.csv
│       ├── test.csv
│       ├── output_01/
│       ├── output_02/
│       ├── output_03/
│       └── figures/
├── dashboard/
│   └── app.py
├── report/
│   └── bao_cao.docx
└── slides/
    └── bao_cao.pptx
```

Các tên notebook là quy ước. Nếu notebook hiện tại mang tên khác, chạy theo vai trò 01 → 02 → 03 và sửa tên tương ứng trong hướng dẫn.

Ba tập chia chỉ lưu trực tiếp trong `notebooks/outputs/`, tránh giữ bản sao cũ trong `output_02`.

## 2. Môi trường chạy

Khuyến nghị Python 3.12, chạy trên máy cá nhân bằng JupyterLab hoặc VS Code có hỗ trợ notebook. `requirements.txt` gồm các thư viện phân tích và thư viện thường dùng cho dashboard Streamlit/Plotly; không yêu cầu Power BI hoặc Tableau.

Từ thư mục gốc `crop_project`, mở terminal và chọn một trong hai cách sau.

### Cách A: Miniconda / Anaconda

Chạy trong Anaconda Prompt:

```bash
conda create -n crop_project python=3.12 -y
conda activate crop_project
python -m pip install -r requirements.txt
python -m ipykernel install --user --name crop_project --display-name "Python (crop_project)"
```

Nếu đã có môi trường Python 3.12 của nhóm, kích hoạt môi trường đó rồi cài requirements và đăng ký kernel; không cần tạo thêm môi trường.

### Cách B: Python trên Windows

```powershell
py -3.12 -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe -m ipykernel install --user --name crop_project --display-name "Python (crop_project)"
```

Lệnh `py -3.12` cần Python 3.12 và Python Launcher được cài trên máy. Môi trường `.venv` không cần đưa vào bài nộp.

Các khoảng phiên bản trong requirements là môi trường đề xuất, chưa phải bản khóa phiên bản đã kiểm chứng với toàn bộ dự án. Sau khi cả notebook và dashboard chạy thành công, nhóm có thể ghi lại phiên bản thực tế bằng `python -m pip freeze > requirements-tested.txt` trong môi trường riêng.

## 3. Mở notebook và kiểm tra đường dẫn

Các notebook hiện dùng `ROOT = Path.cwd()`. Vì vậy, thư mục làm việc phải là `crop_project/notebooks`, không phải thư mục gốc dự án.

Với Conda, từ thư mục gốc dự án:

```bash
conda activate crop_project
cd notebooks
python -m jupyterlab
```

Với môi trường venv Windows, từ thư mục gốc:

```powershell
cd notebooks
..\.venv\Scripts\python.exe -m jupyterlab
```

Chọn kernel `Python (crop_project)`. Nếu dùng VS Code, mở notebook và chọn cùng kernel; kiểm tra thư mục làm việc thay vì giả định vị trí notebook là thư mục đang chạy:

```python
from pathlib import Path
print(Path.cwd())
```

Kết quả cần trỏ đến thư mục `notebooks`. Nếu khác, chỉnh thư mục làm việc hoặc cấu hình đường dẫn trước khi chạy.

Đường dẫn thống nhất trong notebook:

```python
ROOT = Path.cwd()
DATA_PATH = ROOT.parent / "data" / "crop_production.csv"
OUT = ROOT / "outputs"
OUT1 = OUT / "output_01"
OUT2 = OUT / "output_02"
OUT3 = OUT / "output_03"
FIG = OUT / "figures"

for folder in [OUT1, OUT2, OUT3, FIG]:
    folder.mkdir(parents=True, exist_ok=True)
```

## 4. Chạy theo thứ tự

### Bước 1 — Thành viên 1: làm sạch

Chạy toàn bộ `01_cleaning.ipynb` từ trên xuống:

- Đọc dữ liệu gốc; chuẩn hóa kiểu số và tên nhóm.
- Xử lý thiếu dữ liệu, bản ghi trùng và giá trị không hợp lệ.
- Giữ `Area > 0`, `Production >= 0`; không tự xóa sản lượng bằng 0.
- Tạo `District_ID = State_Name + " / " + District_Name`.
- Tính `Yield = Production / Area` và gắn cờ ngoại lệ theo phương pháp nhóm đã thống nhất; không tự động xóa ngoại lệ.
- Chọn 12 cây, giữ mọi bang trong phạm vi; ghi danh sách cây vào cấu hình.
- Bàn giao `outputs/config.json` và `outputs/model_data_top12.csv`.

Theo quy trình kiểm tra độ phủ đã chốt, file bàn giao ở bước này còn năm 2015 nếu nguồn có dữ liệu năm đó. Không lọc mất 2015 trước khi thành viên 2 kiểm tra coverage. Các nhật ký/bảng làm sạch lưu trong `output_01`.

### Bước 2 — Thành viên 1 và 2: mô tả, chẩn đoán và chia tập

Chạy `02_descriptive_diagnostic.ipynb`:

- Đọc `outputs/config.json` và `outputs/model_data_top12.csv`.
- Tạo coverage theo năm và bang–năm trước khi lọc. Đối chiếu số bản ghi, địa bàn và cây giữa các năm để giải thích quyết định về 2015.
- Sau khi quyết định loại 2015, giữ dữ liệu đến hết 2014, bắt đầu từ năm đầu tiên thực có trong nguồn (hiện là 1997). Không ghi đè file đầu vào còn 2015 nếu cần giữ bằng chứng trước–sau. Một số biểu đồ riêng vẫn khảo sát 1998–2014 theo nhãn của hình.
- Tạo thống kê, availability và biểu đồ mô tả từ dữ liệu cuối cùng.
- Thực hiện chẩn đoán phục vụ minh họa khuyến nghị trên dữ liệu đến hết 2012. Nếu kết quả dùng để thiết kế/chọn biến mô hình, chỉ dùng train.
- Tạo `Year_Index = Crop_Year - 1998` trước khi chia tập, nếu chưa có.
- Lưu ba tập đầy đủ cột trực tiếp vào `outputs/`.

| Tập | Năm | Mục đích |
|---|---|---|
| `train.csv` | 1997–2010 với nguồn hiện tại | Học mô hình |
| `validation.csv` | 2011–2012 | Chọn thuật toán và tham số |
| `test.csv` | 2013–2014 | Đánh giá cuối cùng |

Chia theo thời gian để mô phỏng học từ quá khứ và dự đoán các năm tiếp theo. Chia ngẫu nhiên có thể đưa quan sát tương lai vào tập huấn luyện, làm đánh giá khả năng dự báo quá lạc quan. Chỉ chọn thuật toán/tham số bằng validation; sau khi chốt cấu hình mới fit lại trên train + validation và đánh giá test. Bảng `output_02/split_summary.csv` ghi năm và số dòng thực tế của từng tập.

Phần mô tả giữ tất cả bang có dữ liệu. Mô hình chỉ xét 8 bang do nhóm chọn cho phạm vi minh họa: Uttar Pradesh, Madhya Pradesh, Maharashtra, Rajasthan, Karnataka, Bihar, West Bengal và Tamil Nadu. Danh sách được đặt trước khi so sánh mô hình; đây không phải kết quả chọn các bang có điểm test cao và không đại diện mọi địa bàn. `output_02/state_split_coverage.csv` ghi số dòng train/validation/test của từng bang. Bang không có test được ghi là chưa được kiểm chứng riêng, không bổ sung dữ liệu giả để đủ địa bàn.

Các tập có cùng cấu trúc cột. Đầu vào mô hình là `Year_Index`, `State_Name`, `District_ID`, `Crop`, `Season`; biến mục tiêu là `Yield`. Giữ thêm `District_Name`, `Crop_Year`, `Area`, `Production`, `Outlier_Flag` để đối chiếu nếu các cột này có trong dữ liệu bàn giao. Không đưa `Yield`, `Production`, `Outlier_Flag` vào FEATURES.

Bảng kết quả mô tả/chẩn đoán và `figure_notes_analysis.csv` lưu trong `output_02`. Biểu đồ lưu trong `figures`; mỗi hình có nhận xét 1–2 câu được tính từ dữ liệu và in ngay bên dưới. Bảng ghi chú có `File`, `Chart`, `Comment`, dùng tên tệp hình thực tế; gồm cả các hình bổ sung mang tiền tố `extra_` để thành viên 3 gộp.

### Bước 3 — Thành viên 3: dự đoán và chỉ định

Chạy `03_model_recommendation.ipynb`:

- Đọc ba tập trực tiếp từ `outputs/`; không chia lại và không dùng test để huấn luyện.
- Tạo bổ sung `Year_Index`, `District_ID` nếu file bàn giao thiếu.
- So sánh baseline theo năm, Ridge và Random Forest trên validation.
- Chọn cấu hình theo tiêu chí định trước, hiện là Macro_NMAE thấp nhất.
- Fit lại các cấu hình đã chốt trên train + validation, đánh giá test 2013–2014.
- Xuất MAE, RMSE, R², Macro_NMAE; xem riêng theo cây, bang và các vụ năng suất thấp.
- Diễn giải mô hình bằng permutation importance.
- Tạo khuyến nghị từ lịch sử đến hết 2012 và dự đoán hồi cứu; nhóm ít dữ liệu cần ghi chưa đủ bằng chứng.
- Lưu kết quả vào `output_03`; hình 7, 8 vào `figures`.

Các file chính gồm `validation_metrics.csv`, `test_metrics.csv`, `test_predictions.csv`, `metrics_by_crop.csv`, `metrics_by_state.csv`, `low_yield_metrics.csv`, `importance.csv`, `recommendations.csv`, `season_candidates.csv`, `model_info.json`.

Các ngưỡng phân loại đề xuất được giữ trong `THRESHOLDS`, xuất ra `recommendation_thresholds.csv` (giá trị, lý do và nguồn quy ước) và `recommendation_thresholds.json`. Đây là quy tắc minh họa của nhóm, chưa được xác nhận là ngưỡng tối ưu. Cấu hình gốc yêu cầu tối thiểu 8 năm lịch sử, 3 năm gần đây, dữ liệu đến 2012 và 10 quan sát validation cấp bang–cây–mùa. Mức ưu tiên yêu cầu ít nhất 10 năm, CV ≤ 25%, NMAE ≤ 0,3, xu hướng không âm và dự đoán ≥ trung bình lịch sử; cảnh báo khi CV > 40%, NMAE > 0,5 hoặc slope/mean < −2%/năm.

Kiểm tra độ nhạy trong `recommendation_sensitivity.csv` so sánh cấu hình gốc với cấu hình siết/nới 20% các ngưỡng CV, NMAE và mức giảm, giữ nguyên yêu cầu độ phủ và mô hình. `recommendation_sensitivity_detail.csv` ghi mức đề xuất từng nhóm trong cả 3 cấu hình. `recommendations.csv` bổ sung `Rule_Explanation` để giải thích phân loại và `Threshold_Stable` cho biết phân loại có giữ nguyên trong cả 3 cấu hình hay không. Phép kiểm tra chỉ dùng lịch sử phát triển và sai số validation, không dùng test để điều chỉnh ngưỡng; độ ổn định theo ngưỡng chưa chứng minh hiệu quả canh tác.

Đọc ghi chú phân tích từ `OUT2 / "figure_notes_analysis.csv"`, không phải OUT3. Nếu chưa có file đó, lưu riêng `figure_notes_model.csv`; bổ sung bảng nhận xét tổng hợp sau khi thành viên 2 bàn giao.

Notebook 03 hỗ trợ `config.json` cũ chưa có `split_strategy`, `split_years`, `model_scope_note`, `states_without_test`: khoảng năm và độ phủ được suy ra từ các CSV train/validation/test đã đọc; chiến lược chỉ ghi `time_based` khi các khoảng năm nối tiếp không chồng lấn. Notebook in thông báo khi phải bổ sung metadata. Các CSV đầu vào vẫn phải tồn tại; chạy 01 → 02 → 03 để tạo lại toàn bộ kết quả nhất quán. Bảng nhận xét cũ dạng `Main_Finding`/`Limitation` cũng được chuyển đổi sang `Comment` và tên hình hiện tại.

Phiên bản này đã được kiểm tra từ thư mục chạy mới, không dùng lại outputs/config cũ: notebook 01 chạy 13/13 ô mã, notebook 02 chạy 28/28 và notebook 03 chạy 8/8, không có output lỗi. Hai notebook 02 và 03 trong repository đã lưu `execution_count` liên tục cùng output của lần chạy này. Kiểm tra kết quả xác nhận 13 hình có nhận xét, 3 mô hình có metric hữu hạn, 4.969 nhóm đề xuất, bảng độ phủ giữ tên bang và năm 1997. Đoạn xuất metadata được kiểm tra với 10 trường hợp config mới/cũ, thiếu hoặc null từng khóa mới; đoạn gộp nhận xét được kiểm tra với cả định dạng cũ và mới.

Các thống kê toàn giai đoạn chứa năm test chỉ phục vụ mô tả; không dùng để chỉnh mô hình hoặc thiết kế quy tắc rồi coi test là độc lập. Nếu thay dữ liệu, đặc trưng hoặc cấu hình, chạy lại các bước liên quan theo thứ tự; không dùng mô hình hoặc kết quả cũ cho phạm vi mới.

## 5. Dashboard

Sau khi các file kết quả đã được tạo, chạy từ thư mục gốc dự án trong môi trường đã cài requirements:

```bash
python -m streamlit run dashboard/app.py
```

Với venv Windows, có thể dùng:

```powershell
.venv\Scripts\python.exe -m streamlit run dashboard\app.py
```

Dashboard cần được nhóm hoàn thiện và đối chiếu đúng các file đầu ra. Trong `dashboard/app.py`, nên dùng đường dẫn theo vị trí script:

```python
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parent.parent
OUT = PROJECT_ROOT / "notebooks" / "outputs"
OUT2 = OUT / "output_02"
OUT3 = OUT / "output_03"
```

Cho phép lọc địa bàn/cây/mùa và xem lịch sử, dự đoán, sai số, mức khuyến nghị. Nếu app.py hiện nằm ở vị trí khác hoặc dùng tên file khác, điều chỉnh đường dẫn tương ứng.

## 6. Diễn giải kết quả và giới hạn

- Đây là bài hồi quy: R² = 0,8 không có nghĩa chính xác 80%; Macro_NMAE cũng không phải accuracy.
- Đánh giá cuối trên dữ liệu có quan sát thực tế. Địa bàn không có test chưa được kiểm chứng riêng.
- Số bản ghi thấp là dấu hiệu độ phủ cần kiểm tra, chưa tự chứng minh nguyên nhân thiếu dữ liệu. Ô không có bản ghi không đồng nghĩa năng suất bằng 0.
- Xác minh đơn vị Production/Area của từng cây trước khi ghi tấn/ha hoặc so sánh. Không xếp hạng lợi nhuận các cây bằng năng suất khối lượng.
- Dữ liệu không có biến khí hậu trong các FEATURES đang dùng; tương quan năm–năng suất không chứng minh tác động nhân quả của biến đổi khí hậu.
- Dự đoán 2014 từ lịch sử đến 2012 là minh họa hồi cứu, không phải dự báo đáng tin cho năm 2026.
- Whole Year không phải thời vụ gieo cụ thể. Các ngưỡng khuyến nghị trong code là quy tắc minh họa, không phải chuẩn nông học. Mỗi đề xuất cần dẫn số liệu từ mô tả, chẩn đoán và dự đoán.

## 7. Lỗi thường gặp

| Lỗi | Cách xử lý |
|---|---|
| FileNotFoundError | Kiểm tra Path.cwd(), tên file và thư mục đọc |
| Cannot save file into a non-existent directory | Chạy mkdir cho đúng OUT1/OUT2/OUT3 trước khi lưu |
| Year_Index hoặc District_ID không có trong bảng | Tạo cột và bảo đảm FEATURES giữ cột đó; chạy lại cell chuẩn bị dữ liệu |
| NameError: LogNorm | Thêm `from matplotlib.colors import LogNorm` |
| PermissionError khi ghi CSV | Đóng file đang mở trong Excel; kiểm tra quyền ghi |
| Sửa code nhưng kết quả chưa đổi | Restart kernel và Run All theo thứ tự |

Lưu hình bằng `plt.savefig(FIG / "ten_hinh.png", ...)` trước `plt.show()`. Không viết `"OUT/ten_file.csv"` khi muốn sử dụng biến OUT.

## 8. Sản phẩm nộp

- Báo cáo Word đủ 9 mục: giới thiệu, dữ liệu, mô tả, chẩn đoán, dự đoán, chỉ định, kết luận, phân công, tài liệu tham khảo.
- Slide trình bày.
- Ba notebook chạy được, có comment; CSV nguồn, requirements và README.
- Tối thiểu một dashboard tương tác.
- Ít nhất 6 loại biểu đồ khác nhau; nhóm dự kiến 8 biểu đồ chính. Cột dọc và thanh ngang cùng thuộc loại bar chart. Mỗi hình có tiêu đề, nhãn trục/đơn vị phù hợp, chú thích khi cần và 1–2 câu nhận xét.

Trước khi nộp, chạy lại dự án từ đầu trên môi trường của nhóm, đối chiếu số liệu trong Word/slide với CSV mới nhất. Cấu trúc trong README là hướng dẫn bàn giao, không xác nhận rằng tất cả notebook và dashboard đã được kiểm thử.
