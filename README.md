# ESG Metrics Extraction and Standardization

## Overview

This branch provides a suite of Python scripts designed to automate the extraction of Environmental, Social, and Governance (ESG) metrics from PDF reports. The workflow includes data extraction, cleaning, and similarity matching to standardize the extracted metrics.

## Repository Structure

### dictionary
- `dictionary_new.xlsx` is a foundational input file that drives the categorization and keyword mapping logic of the script. It ensures that metrics and their attributes are correctly read, categorized, and exported for downstream processes.
  
- `qualitative_metrics_keywords.json`: An output file derived from metrics in `dictionary_new.xlsx` where the "Indicators category" is labeled as "Exposure". It provides a mapping between qualitative metrics and their associated keywords.

- `quantity_metrics_keywords.json`: An output file generated by excluding qualitative metrics from the complete metrics-to-keywords mapping (`metrics_keywords`). This file stores the relationships between non-qualitative metrics and their corresponding keywords.

### scr ([detailed explanation](https://github.com/ariahuang314/groupproject/wiki/3-Automated-ESG-Metric-Extraction-and-Processing-Workflow))
- `main.py`: Running this file directly executes the entire data extraction and processing workflow in the `scr` folder. It collaborates with multiple modules to complete the tasks: loading the required files (`load_file`), analyzing data using the language model (`load_llm_model`) and supplement model (`load_supplement_model`), processing metric data with the merging and similarity calculation modules (`merge_metric and calculate_simliarity`), saving the results to the database (`save_db`) and scoring the data (`scoring_metric`).
  
- `pdf_to_txt.py`: We convert the ESG reports to texts by using regularization and provide a pipeline to convert PDF files into clean, structured text files, particularly useful for processing documents for further text analysis.
  
- `divide_quantity_qualitative.py`: From the dictionary created, we divide metrics into qualitative and quantitative categories and save them to `.json` files.
  
- `llm_model.py`: We define a pipeline for extracting ESG-related metrics from semi-structured text using OpenAI's language model, specifically configured for environmental, social, and governance data. It provides utilities to retrieve ESG indicators, values, units, and confidence scores from raw text and then saves the results in CSV format.
  
- `supplement_model.py`: We load a sentiment analysis model to process qualitative metrics based on a JSON file of metric keywords. The function searches for keyword matches within the text file and splits matched sentences into shorter sub-sentences for analysis. Using it, we evaluate each sub-sentence to obtian sentiment and confidence score. For each detected metric, we assign a value of 1, store it in a new DataFrame alongside its confidence score and drop duplicates.
  
- `merge_similarity.py`: In this code file, we merge the quantitative and qualitative metrics from separate files and process these metrics with a BERT model to identify semantic similarities to the target labels. The outcomes are saved based on the similarity thresholds.
  
- `save_to_db`: In this file, we upload the processed data to the MySQL database. We align column names with the database schema and uses `SQLAlchemy` to insert the data into a specified table, enabling efficient data management and querying.

- `scoring_code.py`: We calculate the ESG scores and ratings for target companies. This file retrieves the latest company data from a database, compares it with industry averages and applies scoring rules to generate individual scores for Environment (E), Social (S), Governance (G) and an overall score. The results, including detailed metrics and ratings, are then stored back in the database for further analysis. 

### upload
- Upload ESG reports to this folder and the `pdf_to_txt.py` file will read their contents, converting the PDFs into text files stored in the `txt` folder for subsequent analysis.

### txt
- The TXT files converted by pdf_to_txt.py will be saved here and later read and processed by llm_model.py.

### output_metric
- `Quantity_metrics.csv`: Stores data extracted from quantitative metrics. This typically includes numerical information, such as amounts or ratios, for further processing and merging.

- `Qualitative_metrics_filter.csv`: Generated by matching qualitative metrics and their associated keywords from `qualitative_metrics_keywords.json`, performing sentiment analysis and filtering the results based on confidence and relevance. It contains filtered qualitative metrics data.This file includes metrics matched with target labels, facilitating subsequent analysis and standardization.

- `metrics.csv`: Holds the combined data of quantitative and qualitative metrics. This output file integrates all metrics and serves as the data source for similarity calculations.

- `metrics_filter.csv`: Contains high-quality metrics filtered from the merged data. After BERT embedding and similarity calculations, the data is refined and cleaned for further usage and analysis.

---------------------------------------------------------------------
# groupproject  
目前剩下的任务：  
1. 利用模型将txt文件转换为grouped_data.json格式的内容，可以在我的基础上调，换模型也行  
直接尝试表格提取工具（pdfplumber，tabula）或者正则化也可以
(可以有缺失值unit)   
2. 行业迁移  
最后整个项目跑遍，把环境下载下来  
**有代码更新记得传github,单独开branch,项目快截止了，还要前后端连接，创建docker, wiki,ppt**  
**你们三个人分一下上面的任务吧，我去算可信度**  

我觉得模型可能需要：  
少量的标签报告内容（体力活）    

## 代码内容
写在前面，所有的input output文件名有些只包含了前半部分  
eg: grouped_data.json 代表目前所有以grouped_data开头和.json结尾的文件 如: grouped_data_full.json  
后半部分不同有时只是数据量不同  

之前的output/1030.txt删除了，内容和AML一样，txt/AML.txt为清洗完全版本  

- scr/pdf_to_txt.ipynb  
将pdf转换成txt, 并清洗txt  
可以处理单个pdf文件或者整个文件夹以下的所有pdf  
  - input: data/file.pdf  
  - output: txt/file.pdf  

- scr/transform_json.ipynb  
将从人工贴的标签（利用label studio）转换为标准化json格式  
  - format:{ "metric":  "value":  "unit": }  
  - input: json/project-full.json  
  - output: json/grouped_data.json  

- scr/ESG_metric_LLM2.ipynb   
  第一个部分：将标准化json按照BIO标签转换为txt  
  BIO标签是bert学习的格式
  - format:Net B-INDICATOR profit I-INDICATOR 1.9 B-VALUE S$ B-UNIT million I-UNIT
  - input: json/grouped_data.json  
  - output: output/bio_data.json  

- scr/ESG_metric_LLM2.ipynb  
  之后部分：调用模型DeBERTa（基于bert的提升模型）  
  目前包含模型调优，模型训练，保存模型，调用调优模型  
  **但是不知道为什么输出是空的,你们可以在此基础上调优或者替换模型，我确实调不出来了**  
  - input:  
    json_filepath = '../json/grouped_data_full1.json'  
    bio_filepath = '../output/bio_data.txt'  
    raw_text_filepath = '../txt/AML.txt'   
  - output: output_metric/esg_data.csv  最好还是json,便于后续代码，当时写的时候没注意，输出格式要和grouped_data.json一致,最好加个来源  
  - output: model/fine_tuned_deberta  调优模型，因为太大了我没上传  

- scr/simiarlity.ipynb  
用于清洗LLM输出的json,只留下B组需要的指标  
计算报告中指标与B组打分指标间的相似度，**（并计算指标可信度，我正在写）**  
包含两种 高精度 和 高速度 模式的相似度  
  - input: json/grouped_data.json   **之后替换为LLM输出**   
  - output: json/filtered_data_bert.json  基于bert词嵌入  
  - output: json/filtered_data.json  基于TF-IDF向量化   

## 关于csv和xlsx
- dictionary.xlsx 是之前提供的字典，列分别为：一级二级三级标题，关键词（可能是三级标题近义词），GRI  
- output/unique_metric.csv  根据dictionary.xlsx提出的唯一三级标题  
