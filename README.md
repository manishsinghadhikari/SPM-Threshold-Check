# 📌 Step Count Analysis from Apple Health Data

This project processes **Apple Health** step count data exported from the **Health App** (iPhone) and analyzes the **Steps Per Minute (SPM)** using Python. It identifies high-intensity activity (SPM > 180 for at least 30 seconds) and visualizes the data using bar charts. 📊  

It is designed to run on **Google Colab** and requires an Apple Health **export.zip** file to be uploaded.

---

## **🛠 Features**
✔ Parses **XML data** exported from Apple Health  
✔ Categorizes step count data into different ranges  
✔ Identifies periods where **SPM > 180 for at least 30 seconds**  
✔ Generates a **bar chart visualization** of abnormal activity  
✔ Works on **Google Colab** without additional setup  

---

## **📂 Installation & Setup**

### **1️⃣ Download Step Data from Apple Health**
Follow these steps to export your **Apple Health** step data:  
1. Open the **Health App** on your **iPhone** 📱  
2. Tap on your **profile picture** (top-right corner)  
3. Select **Export All Health Data**  
4. This will generate an **export.zip** file  
5. Transfer this **export.zip** to your computer  

---

### **2️⃣ Upload `export.zip` to Google Colab**
Since the script runs on **Google Colab**, you must manually upload the exported health data:  
1. Open **Google Colab** 🔗 [https://colab.research.google.com](https://colab.research.google.com)  
2. Upload `export.zip` to Colab’s **Files** section (Left Sidebar)  
3. Extract the zip file using:

```python
import zipfile
import os

zip_path = "/content/export.zip"  # Path where you uploaded the file
extract_path = "/content/health_data"

with zipfile.ZipFile(zip_path, 'r') as zip_ref:
    zip_ref.extractall(extract_path)

print("✅ Extraction complete! Files are in:", extract_path)
```

---

### **3️⃣ Install Required Libraries**
Before running the script, install dependencies using:  

```python
!pip install seaborn pandas matplotlib
```

---

## **🚀 Running the Script**

### **Load & Process the Step Data**
Run the following script in **Google Colab** after uploading and extracting `export.zip`:

```python
import xml.etree.ElementTree as ET
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from datetime import datetime, timedelta

def process_and_visualize_steps(file_path, device_name):
    def parse_time(time_str):
        return datetime.strptime(time_str, "%Y-%m-%d %H:%M:%S %z")

    try:
        tree = ET.parse(file_path)
        records = tree.findall(".//Record[@type='HKQuantityTypeIdentifierStepCount' "
                              f"][@sourceName='{device_name}']")
    except Exception as e:
        print(f"Error: {e}")
        return pd.DataFrame()

    minute_data = {}

    for record in records:
        try:
            steps = int(record.get('value', 0))
            start = parse_time(record.get('startDate'))
            end = parse_time(record.get('endDate'))

            if start >= end:
                continue

            total_seconds = (end - start).total_seconds()
            if total_seconds == 0:
                continue

            current_min = start.replace(second=0, microsecond=0)
            while current_min < end:
                next_min = current_min + timedelta(minutes=1)
                overlap_start = max(start, current_min)
                overlap_end = min(end, next_min)
                overlap_seconds = (overlap_end - overlap_start).total_seconds()

                if overlap_seconds > 0:
                    minute_key = current_min
                    step_contribution = steps * (overlap_seconds / total_seconds)

                    if minute_key in minute_data:
                        minute_data[minute_key] += step_contribution
                    else:
                        minute_data[minute_key] = step_contribution

                current_min = next_min

        except Exception as e:
            print(f"Skipping invalid record: {e}")
            continue

    if not minute_data:
        return pd.DataFrame()

    df = pd.DataFrame(list(minute_data.items()), columns=['Minute', 'Steps'])
    df['Steps'] = df['Steps'].round().astype(int)

    bins = [0, 100, 150, 180, 200, float('inf')]
    labels = ['<100', '100-150', '150-180', '180-200', '200+']
    df['Category'] = pd.cut(df['Steps'], bins=bins, labels=labels, right=False)

    df.to_csv(f"{device_name}.csv", index=False)

    threshold = 180
    high_spm_duration = 30
    abnormal_activity_intervals = []

    for i in range(1, len(df)):
        if df['Steps'].iloc[i] > threshold:
            if (df['Minute'].iloc[i] - df['Minute'].iloc[i-1]).total_seconds() >= high_spm_duration:
                abnormal_activity_intervals.append(df['Minute'].iloc[i])

    print(f"Abnormal Activity (SPM > 180 for 30 seconds or more): {len(abnormal_activity_intervals)} occurrences")

    abnormal_df = df[df['Minute'].isin(abnormal_activity_intervals)]

    plt.figure(figsize=(14, 8))
    ax = sns.barplot(x='Minute', y='Steps', data=abnormal_df, palette='Reds')

    for p in ax.patches:
        if p.get_height() > 0:
            ax.annotate(f"{int(p.get_height())}", (p.get_x() + p.get_width() / 2., p.get_height()),
                        ha='center', va='center', fontsize=8, color='black', xytext=(0, 5),
                        textcoords='offset points')

    plt.title('Abnormal Activity (SPM > 180 for 30 seconds or more)', fontsize=16)
    plt.xlabel('Time (Minute Intervals)', fontsize=12)
    plt.ylabel('Steps per Minute', fontsize=12)
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.savefig(f'{device_name}_abnormal_activity.png', bbox_inches='tight')
    plt.show()

    return df

file_path = "/content/export/health_data/apple_health_export/export.xml"
device_name = "Manish Singh’s Apple Watch"

processed_data = process_and_visualize_steps(file_path, device_name)
```

---

## **📊 Output**
- The script will print **category-wise step distribution**  
- It will display the **number of times SPM > 180 for 30 seconds**  
- A **bar chart** will be generated to highlight high activity intervals  

---

## **📌 Notes**
⚠ Ensure you have uploaded `export.zip` to Google Colab before running the script  
⚠ Adjust the `device_name` in the script to match your **Apple Watch name**  

---

## **🤝 Contributing**
Feel free to open issues and submit pull requests for improvements! 🚀

