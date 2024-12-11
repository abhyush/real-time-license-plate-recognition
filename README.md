# Real-Time Automatic Number Plate Recognition (ANPR)

This project is a real-time Automatic Number Plate Recognition (ANPR) system designed to detect and track vehicle license plates. It uses **YOLO** for object detection, **AWS Kinesis** for video streaming, and AWS services like **S3**, **DynamoDB**, and **Lambda** for data storage, processing, and visualization. This system is scalable and reliable, making it ideal for applications in traffic monitoring, toll collection, and vehicle tracking.

---

## Project Flow

### **1. Video Input (Producer Stage)**
- **Input**: A live video stream or a pre-recorded video file.
- **AWS Kinesis Video Streams**: Streams video data to handle real-time or offline processing.
- **EC2 Instance**:
  - A lightweight **t2.small** instance is used to stream video to Kinesis.
  - **GStreamer** pipelines are configured to encode and transmit the video stream.

### **2. Object Detection and Tracking (Consumer #1)**
- **Frame Extraction**: Extracts frames from the Kinesis video stream.
- **YOLO (You Only Look Once)**: Detects vehicles and license plates in the frames.
- **SORT (Simple Online and Realtime Tracking)**: Tracks vehicles across frames.
- **Data Storage**:
  - **S3**: Cropped images of detected plates are uploaded.
  - **DynamoDB**: Metadata, such as plate number and timestamp, is stored.

### **3. Data Processing (Lambda Function)**
- **Trigger**: Uploading a new image to S3 triggers an AWS Lambda function.
- **OCR Processing**:
  - Lambda retrieves the image and performs OCR to extract the license plate number.
  - Updates DynamoDB with the plate number and related metadata.

### **4. Visualization and Reporting (Consumer #2)**
- **Visualization Scripts**:
  - Retrieve data from S3 and DynamoDB.
  - Generate annotated frames and dashboards for real-time monitoring.
- **Event Handling**:
  - **SQS (Simple Queue Service)** ensures smooth communication between components.
- **Reporting**:
  - Generate traffic or vehicle logs and store them in S3 or DynamoDB.

---

## How to Set Up the Project

### **1. Prerequisites**
- **AWS Account**: Ensure you have access to AWS services like EC2, S3, DynamoDB, Lambda, SQS, and Kinesis.
- **Tools**:
  - Python 3.x
  - AWS CLI
  - GStreamer installed on your EC2 instance.

---

### **2. Setting Up AWS Components**

#### **Step 1: Kinesis Video Streams**
1. Log in to [AWS](https://aws.amazon.com/).
2. Navigate to **Kinesis Video Streams** and create a video stream.
3. Use the stream name (e.g., `anpr-video-stream`) for configuration in your EC2 producer instance.

#### **Step 2: EC2 Instance for Producer**
1. Launch a **t2.small** EC2 instance.
2. SSH into the instance and execute:
   ```bash
   sudo apt update
   sudo apt install gstreamer1.0-tools gstreamer1.0-plugins-{base,good,bad,ugly} -y
   git clone https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp.git
   cd amazon-kinesis-video-streams-producer-sdk-cpp
   mkdir build && cd build
   cmake .. -DBUILD_DEPENDENCIES=OFF -DBUILD_GSTREAMER_PLUGIN=ON
   make && sudo make install
   ```
3. Stream video to Kinesis:
   ```bash
   gst-launch-1.0 -v filesrc location="<path-to-video>.mp4" ! qtdemux name=demux ! queue ! h264parse ! video/x-h264,stream-format=avc,alignment=au ! kvssink stream-name="<stream-name>" access-key="<your-access-key>" secret-key="<your-secret-key>" aws-region="<region-name>" streaming-type=offline
   ```

#### **Step 3: EC2 Instance for Object Detection (Consumer #1)**
1. Launch a **g4dn.xlarge** or similar GPU-enabled EC2 instance.
2. SSH into the instance and install dependencies:
   ```bash
   sudo apt update
   sudo apt install python3-virtualenv python3-tk ffmpeg -y
   virtualenv venv --python=python3
   source venv/bin/activate
   git clone https://github.com/computervisioneng/amazon-kinesis-video-streams-consumer-library-for-python.git
   cd amazon-kinesis-video-streams-consumer-library-for-python
   pip install -r requirements.txt
   pip install ultralytics
   ```
3. Configure YOLO and SORT in the consumer script to process frames from the Kinesis stream.

#### **Step 4: S3 and DynamoDB**
1. **S3 Bucket**:
   - Create an S3 bucket (e.g., `anpr-detected-plates`) to store cropped plate images.
2. **DynamoDB Table**:
   - Create a table with primary key attributes for plate numbers and timestamps.

#### **Step 5: Lambda for OCR**
1. Create a Lambda function triggered by S3 uploads.
2. Install OCR libraries (e.g., `pytesseract`) in the Lambda deployment package.
3. Add IAM permissions for DynamoDB, S3, and CloudWatch logs.
4. Code example:
   ```python
   import boto3
   from pytesseract import image_to_string
   
   def lambda_handler(event, context):
       s3 = boto3.client('s3')
       dynamodb = boto3.resource('dynamodb')
       table = dynamodb.Table('anpr-metadata')
       
       bucket = event['Records'][0]['s3']['bucket']['name']
       key = event['Records'][0]['s3']['object']['key']
       
       obj = s3.get_object(Bucket=bucket, Key=key)
       img_data = obj['Body'].read()
       
       plate_text = image_to_string(img_data)
       table.put_item(Item={'PlateNumber': plate_text, 'Timestamp': 'current-timestamp'})
   ```

#### **Step 6: SQS for Communication**
1. Create a FIFO queue in SQS to handle detection and processing events.
2. Configure the queue to integrate with Lambda or visualization scripts.

---

### **3. Visualization (Consumer #2)**
1. Clone and modify the visualization scripts (e.g., `process_queue.py` and `main_plot.py`).
2. Use these scripts to visualize annotated video frames and metadata from S3 and DynamoDB.
3. Execute:
   ```bash
   python process_queue.py
   python main_plot.py
   ```

---

## How to Use
1. Stream video using the producer EC2 instance.
2. Run the object detection consumer to process frames.
3. Use the visualization scripts to monitor real-time results.
4. View processed data in DynamoDB or S3 for reporting.

---

## Features
- Real-time license plate detection and tracking.
- Scalable storage with S3 and DynamoDB.
- Event-driven architecture using Lambda and SQS.
- Visualization of detected plates and tracking data.

---

## License
This project is licensed under the MIT License.

