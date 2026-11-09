# Scan_Excompress
对扫描的pdf进行最小化黑白二值压缩

！！！该方法只适合黑白扫描位图且边界清晰的文档！！！

## 一、将pdf导出为图片
### 1.下载poppler工具并解压缩
下载地址：https://poppler.freedesktop.org/

以我的目录为例：

`D:\Tools\poppler-25.12.0\Library\bin\`

将上述目录（以实际目录为例）配置到系统环境变量

### 2.WIN+R cmd调出控制台

poppler无损导出pdf的所有页面为tiff图片的指令是：

`pdfimages -tiff "X:\sample_file.pdf" "X:\sample_folder"`

其中 `X:\sample_file.pdf`填写要操作的pdf位置，

`"X:\sample_folder"`填写导出图片的保存位置。


## 二、压缩图片为黑白二值tiff

```
import os
import re
import numpy as np
from PIL import Image, ImageOps

input_folder  = r'.\sample_folder'
output_folder = r'.\sample_folder_compressed'
os.makedirs(output_folder, exist_ok=True)

exts = ('.tif', '.tiff', '.png', '.jpg', '.jp2', '.j2k', '.jpf', '.pgm', '.ppm', '.pbm')

def otsu(gray_np):
    hist = np.bincount(gray_np.ravel(), minlength=256).astype(np.float64)
    total = gray_np.size
    sum_total = np.dot(np.arange(256), hist)
    sum_b, w_b, best_t, best_var = 0.0, 0, 128, -1.0
    for t in range(256):
        w_b += hist[t]
        if w_b == 0: continue
        w_f = total - w_b
        if w_f == 0: break
        sum_b += t * hist[t]
        m_b, m_f = sum_b / w_b, (sum_total - sum_b) / w_f
        var = w_b * w_f * (m_b - m_f) ** 2
        if var > best_var:
            best_var, best_t = var, t
    return best_t

files = sorted(
    (f for f in os.listdir(input_folder) if f.lower().endswith(exts)),
    key=lambda s: [int(t) if t.isdigit() else t.lower() for t in re.split(r'(\d+)', s)]
)

for i, filename in enumerate(files, 1):
    src = os.path.join(input_folder, filename)
    dst = os.path.join(output_folder, os.path.splitext(filename)[0] + ".tif")
    try:
        with Image.open(src) as img:
            dpi = img.info.get('dpi', (300, 300))
            # 透明处理
            if img.mode in ('RGBA', 'LA') or (img.mode == 'P' and 'transparency' in img.info):
                img = img.convert('RGBA')
                bg = Image.new('RGB', img.size, (255, 255, 255))
                bg.paste(img, mask=img.split()[-1])
                img = bg
            gray = ImageOps.autocontrast(img.convert('L'))
            t = otsu(np.asarray(gray))
            bw = gray.point(lambda x: 0 if x < t else 255, '1')
            bw.save(dst, compression="group4", dpi=dpi)
        if i % 50 == 0:
            print(f"{i}/{len(files)}")
    except Exception as e:
        print(f"[FAIL] {filename}: {e}")

```



## 三、整合压缩后图片为新pdf

```
import os
import img2pdf

def fast_combine_tifs_to_pdf(folder_path, output_pdf_name):
    # 1. 获取并排序文件
    files = [f for f in os.listdir(folder_path) if f.lower().endswith(('.tif', '.tiff'))]
    files.sort(key=natural_key)
    
    # 构建完整路径列表
    file_paths = [os.path.join(folder_path, f) for f in files]

    if not file_paths:
        print("未找到 TIF 文件")
        return

    print(f"正在使用流式写入合并 {len(file_paths)} 张 CCITT G4 TIF 图片...")

    # 2. 直接使用 img2pdf 写入文件
    # img2pdf.convert 会自动处理 CCITT Group 4 嵌入，不需要手动解码
    try:
        with open(output_pdf_name, "wb") as f:
            f.write(img2pdf.convert(file_paths))
        print(f"合并成功！输出文件：{output_pdf_name}")
    except Exception as e:
        print(f"合并过程中出错: {e}")

# --- 配置参数 ---
target_folder = r'.\sample_folder_compressed'
output_name = 'sample_compressed.pdf'

fast_combine_tifs_to_pdf(target_folder, output_name)
```
