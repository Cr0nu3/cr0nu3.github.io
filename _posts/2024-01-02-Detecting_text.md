---
title: Detecting Text & Getting Similarity between images
description: >-
  Machine Learning Project
author: Cronus
date: 2024-01-02 10:51:00 
categories: [Project, Edu Helper]
tags: [Machine Learning]
pin: false
image:
  path: /assets/img/ML/detect.jpeg
  width: 300
  height: 300
---

The program flow in EDU_Siri operates as follows:
```
Insert desired video -> Detect motion & capture frames -> Compare similarity between captured images -> Perform text recognition
```

For details on motion detection and screen capture, refer to the previous post.
In the previous implementation, when the user specifies a directory, the program creates a folder with the designated name and stores the captured frames inside it. The next step is to compare the similarity between consecutive images in this folder.

## 1. Checking Directory & Comparing Image Similarity
```python
# Check all images within the directory
def file_listing(path) -> str:
    files = os.listdir(path)

    for i in range(1, len(files)):
        comparePic(path, files[i-1], files[i])

    print("\n+=====================================+")
    print("| Organizing class materials is Done! |")
    print("+=====================================+")
    
# Compare similarity between images
def comparePic(path, image1, image2) -> str:
    image1_path = path + "/" + image1
    image2_path = path + "/" + image2

    pre = cv.imread(image1_path)
    post = cv.imread(image2_path)
    result = pre.copy()

    grayPre = cv.cvtColor(pre, cv.COLOR_BGR2GRAY)
    grayPost = cv.cvtColor(post, cv.COLOR_BGR2GRAY)

    (Similarity, diff) = compare_ss(grayPre, grayPost, full=True)
    diff = (diff * 255).astype('uint8')

    if Similarity < 0.95:
        PicToText(post)
```

The algorithm used for image similarity comparison is quite straightforward.
	1.	Load images into pre and post
	2.	Convert images to grayscale using cv.cvtColor() with COLOR_BGR2GRAY
	3.	Use compare_ss() to perform structural similarity comparison on the entire image (full=True)
	4.	Extract the similarity score (Similarity)

![1.png](assets/img/ML/detect1.png)
Structural Similarity Index (SSIM) measures the structural similarity between two images:
- 1 → Completely identical
- -1 → Completely different

Since lecture slides contain textual information, even minor textual changes need to be detected. After extensive testing, 0.95 was determined to be the optimal threshold for triggering text recognition.

## 2. Setting Up Dependencies for Text Recognition
Before performing text recognition, install the necessary dependencies:
```bash
pip install pillow
pip install pytesseract
```

We use pytesseract since it supports both English and Korean text recognition.

```python
def PicToText(path) -> str:
    # Initialize Config Parser & Read Config File
    config = configparser.ConfigParser()
    config.read(os.path.dirname(os.path.realpath(__file__)) + os.sep + 'env' + os.sep + 'property.ini')

    ocrToStr(path, 'eng')
```
On line 4, the program loads the property.ini configuration file based on the operating system’s path.

```ini
# /env/property.ini
[Bigdata Ocr Extract]
Version= 1.0

[Path]
OcrTxtPath= \\resource\\ocr_result_txt  # Path where extracted text files are saved
```

Typically, a path is specified for saving extracted text files. However, in EDU_Siri, the extracted text is formatted and saved into a designated file, so this path setting can be ignored.

For more details on configparser, refer to the official [Python documentation](https://docs.python.org/3/library/configparser.html).


## 3. Detecting Text & Saving Output
```python
def ocrToStr(fullPath, lang='eng'):
    img = Image.fromarray(fullPath)
    
    # preserve_interword_spaces: Adjusts word spacing for better accuracy
    # psm (Page Segmentation Mode): Defines the region from which text is extracted
    # psm modes: https://github.com/tesseract-ocr/tesseract/wiki/Command-Line-Usage

    # Extract text from the image
    outText = image_to_string(img, lang=lang, config='--psm 1 -c preserve_interword_spaces=1')
    strToText(outText)
    
def strToText(outText):
    with open('./course_material/ppt_content.txt', 'a', encoding='utf-8') as f:
        f.write(outText)
        f.write("==========================================================\n")
```
- The extracted text is saved to a designated file.
- The function uses psm 1 mode, which considers the entire image for text extraction.
- The result is stored in ppt_content.txt, ensuring all extracted text is systematically saved.

## Full Code
```python
def comparePic(path, image1, image2) -> str:
    image1_path = path + "/" + image1
    image2_path = path + "/" + image2

    pre = cv.imread(image1_path)
    post = cv.imread(image2_path)
    result = pre.copy()

    grayPre = cv.cvtColor(pre, cv.COLOR_BGR2GRAY)
    grayPost = cv.cvtColor(post, cv.COLOR_BGR2GRAY)

    (Similarity, diff) = compare_ss(grayPre, grayPost, full=True)
    diff = (diff * 255).astype('uint8')

    if Similarity < 0.95:
        PicToText(post)

# Function 1: Perform OCR (Optical Character Recognition)
def ocrToStr(fullPath, lang='eng'):
    img = Image.fromarray(fullPath)

    # preserve_interword_spaces: Adjusts word spacing
    # psm (Page Segmentation Mode): Defines text extraction boundaries
    outText = image_to_string(img, lang=lang, config='--psm 1 -c preserve_interword_spaces=1')
    strToText(outText)

def strToText(outText):
    with open('./course_material/ppt_content.txt', 'a', encoding='utf-8') as f:
        f.write(outText)
        f.write("==========================================================\n")

# Main function for text recognition
def PicToText(path) -> str:
    # Initialize Config Parser & Read Config File
    config = configparser.ConfigParser()
    config.read(os.path.dirname(os.path.realpath(__file__)) + os.sep + 'env' + os.sep + 'property.ini')

    ocrToStr(path, 'eng')

# Scan directory and compare image similarity
def file_listing(path) -> str:
    files = os.listdir(path)

    for i in range(1, len(files)):
        comparePic(path, files[i-1], files[i])
    
    print("\n+=====================================+")
    print("| Organizing class materials is Done! |")
    print("+=====================================+")
```