## 1. File Upload and Download Strategies

File uploads and downloads require special handling for large files, resumability, security, and performance.

```
Small files (<10 MB):
  → Direct upload to server
  → Simple, no special handling

Large files (>100 MB):
  → Chunked upload
  → Resumable upload
  → Direct to storage (presigned URLs)
```

**Challenges:** Large file size, network interruptions, timeout issues, security, bandwidth costs.

---

## 2. Direct Upload to Server

### Simple upload

```
Client → Server → Storage (S3)

POST /upload
Content-Type: multipart/form-data

Server:
  1. Receive file
  2. Validate (size, type, virus scan)
  3. Upload to S3
  4. Return URL

Pros:
  ✅ Simple
  ✅ Server can validate before storage

Cons:
  ❌ Server bandwidth (file passes through server)
  ❌ Server memory (file buffered in memory)
  ❌ Timeout for large files
```

### Implementation

```python
from flask import Flask, request
import boto3

app = Flask(__name__)
s3 = boto3.client('s3')

@app.route('/upload', methods=['POST'])
def upload_file():
    file = request.files['file']
    
    # Validate
    if file.content_length > 100 * 1024 * 1024:  # 100 MB
        return {"error": "File too large"}, 400
    
    # Upload to S3
    s3.upload_fileobj(
        file,
        'my-bucket',
        f'uploads/{file.filename}'
    )
    
    return {"url": f"https://my-bucket.s3.amazonaws.com/uploads/{file.filename}"}
```

---

## 3. Direct Upload to Storage (Presigned URLs)

Client uploads directly to S3, bypassing the server.

### Flow

```
1. Client → Server: Request upload URL
2. Server → Client: Presigned URL (valid for 15 minutes)
3. Client → S3: Upload file directly using presigned URL
4. Client → Server: Notify upload complete

Pros:
  ✅ No server bandwidth (direct to S3)
  ✅ Scalable (S3 handles load)
  ✅ Fast (no server bottleneck)

Cons:
  ❌ Can't validate before upload
  ❌ Client needs S3 access
```

### Implementation

```python
# Server: Generate presigned URL
@app.route('/upload-url', methods=['POST'])
def get_upload_url():
    filename = request.json['filename']
    content_type = request.json['content_type']
    
    # Generate presigned URL (valid for 15 minutes)
    url = s3.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': 'my-bucket',
            'Key': f'uploads/{filename}',
            'ContentType': content_type
        },
        ExpiresIn=900  # 15 minutes
    )
    
    return {"upload_url": url}

# Client: Upload directly to S3
import requests

# 1. Get presigned URL
response = requests.post('https://api.example.com/upload-url', json={
    'filename': 'photo.jpg',
    'content_type': 'image/jpeg'
})
upload_url = response.json()['upload_url']

# 2. Upload file directly to S3
with open('photo.jpg', 'rb') as f:
    requests.put(upload_url, data=f, headers={'Content-Type': 'image/jpeg'})

# 3. Notify server
requests.post('https://api.example.com/upload-complete', json={
    'filename': 'photo.jpg'
})
```

---

## 4. Chunked Upload

Split large files into chunks and upload separately.

### Why chunked upload?

```
100 MB file, single upload:
  → Network interruption at 90 MB
  → Restart from 0 MB
  → Wasted 90 MB of bandwidth

100 MB file, 10 MB chunks:
  → Network interruption at chunk 9
  → Resume from chunk 9
  → Only re-upload 10 MB
```

### Multipart upload (S3)

```
1. Initiate multipart upload
   POST /bucket/key?uploads
   → Returns upload_id

2. Upload parts (in parallel)
   PUT /bucket/key?partNumber=1&uploadId=xyz
   PUT /bucket/key?partNumber=2&uploadId=xyz
   PUT /bucket/key?partNumber=3&uploadId=xyz
   → Each part returns ETag

3. Complete multipart upload
   POST /bucket/key?uploadId=xyz
   Body: List of part numbers and ETags
   → S3 assembles parts into final file

Benefits:
  - Parallel uploads (faster)
  - Resume on failure (re-upload only failed parts)
  - Upload files > 5 GB (S3 single PUT limit)
```

### Implementation

```python
import boto3

s3 = boto3.client('s3')

def upload_large_file(file_path, bucket, key):
    # 1. Initiate multipart upload
    response = s3.create_multipart_upload(
        Bucket=bucket,
        Key=key
    )
    upload_id = response['UploadId']
    
    # 2. Upload parts
    parts = []
    part_size = 10 * 1024 * 1024  # 10 MB
    
    with open(file_path, 'rb') as f:
        part_number = 1
        while True:
            data = f.read(part_size)
            if not data:
                break
            
            response = s3.upload_part(
                Bucket=bucket,
                Key=key,
                PartNumber=part_number,
                UploadId=upload_id,
                Body=data
            )
            
            parts.append({
                'PartNumber': part_number,
                'ETag': response['ETag']
            })
            
            part_number += 1
    
    # 3. Complete multipart upload
    s3.complete_multipart_upload(
        Bucket=bucket,
        Key=key,
        UploadId=upload_id,
        MultipartUpload={'Parts': parts}
    )
```

---

## 5. Resumable Upload (TUS Protocol)

TUS is an open protocol for resumable file uploads.

### How it works

```
1. Client: Create upload
   POST /files
   Upload-Length: 100000000
   → Server: Returns upload URL and offset (0)

2. Client: Upload chunk
   PATCH /files/abc123
   Upload-Offset: 0
   Content-Length: 10000000
   Body: [chunk data]
   → Server: Returns new offset (10000000)

3. Network interruption

4. Client: Check offset
   HEAD /files/abc123
   → Server: Upload-Offset: 10000000

5. Client: Resume from offset
   PATCH /files/abc123
   Upload-Offset: 10000000
   Content-Length: 10000000
   Body: [next chunk]
```

### Implementation (tus-py-client)

```python
from tusclient import client

# Client
my_client = client.TusClient('https://api.example.com/files/')
uploader = my_client.uploader('large_file.mp4', chunk_size=10*1024*1024)

# Upload with automatic resume
uploader.upload()

# Server (Flask + tusd)
# Use tusd server (Go implementation) or Flask-Tus
```

---

## 6. File Download

### Direct download

```
GET /files/abc123

Server:
  1. Retrieve file from S3
  2. Stream to client

Cons:
  ❌ Server bandwidth
  ❌ Server becomes bottleneck
```

### Presigned URL download

```
1. Client → Server: Request download URL
2. Server → Client: Presigned URL (valid for 1 hour)
3. Client → S3: Download directly

Pros:
  ✅ No server bandwidth
  ✅ Scalable
```

```python
# Server: Generate presigned URL
@app.route('/download-url/<file_id>')
def get_download_url(file_id):
    url = s3.generate_presigned_url(
        'get_object',
        Params={
            'Bucket': 'my-bucket',
            'Key': f'uploads/{file_id}'
        },
        ExpiresIn=3600  # 1 hour
    )
    return {"download_url": url}
```

### Range requests (partial download)

Support resumable downloads and streaming.

```
Client: GET /files/abc123
        Range: bytes=0-1023

Server: HTTP 206 Partial Content
        Content-Range: bytes 0-1023/100000
        Content-Length: 1024
        Body: [first 1024 bytes]

Client: GET /files/abc123
        Range: bytes=1024-2047

Server: HTTP 206 Partial Content
        Content-Range: bytes 1024-2047/100000
        Content-Length: 1024
        Body: [next 1024 bytes]

Use cases:
  - Resume interrupted downloads
  - Video streaming (seek to position)
  - Download specific parts
```

---

## 7. Security

### File type validation

```python
import magic

def validate_file_type(file):
    # Check MIME type (don't trust client)
    mime = magic.from_buffer(file.read(1024), mime=True)
    file.seek(0)
    
    allowed_types = ['image/jpeg', 'image/png', 'application/pdf']
    if mime not in allowed_types:
        raise ValueError("Invalid file type")
```

### File size limits

```python
# Nginx
client_max_body_size 100M;

# Flask
app.config['MAX_CONTENT_LENGTH'] = 100 * 1024 * 1024  # 100 MB

# Validate before processing
if request.content_length > 100 * 1024 * 1024:
    return {"error": "File too large"}, 413
```

### Virus scanning

```python
import clamd

def scan_file(file_path):
    cd = clamd.ClamdUnixSocket()
    result = cd.scan(file_path)
    
    if result[file_path][0] == 'FOUND':
        raise ValueError(f"Virus detected: {result[file_path][1]}")
```

### Filename sanitization

```python
import os
from werkzeug.utils import secure_filename

def sanitize_filename(filename):
    # Remove path traversal attempts
    filename = secure_filename(filename)
    
    # Generate unique filename
    unique_id = uuid.uuid4().hex
    extension = os.path.splitext(filename)[1]
    
    return f"{unique_id}{extension}"
```

---

## 8. Performance Optimization

### CDN for downloads

```
Upload: Client → Server → S3
Download: Client → CloudFront (CDN) → S3

Benefits:
  - Edge caching (faster downloads)
  - Reduced S3 costs (CloudFront cheaper)
  - Better user experience
```

### Compression

```python
import gzip

# Compress before upload
with open('large_file.txt', 'rb') as f_in:
    with gzip.open('large_file.txt.gz', 'wb') as f_out:
        f_out.writelines(f_in)

# Upload compressed file
s3.upload_file('large_file.txt.gz', 'my-bucket', 'uploads/file.txt.gz')

# Client decompresses on download
```

### Parallel uploads

```python
from concurrent.futures import ThreadPoolExecutor

def upload_chunk(chunk_data, part_number):
    return s3.upload_part(
        Bucket=bucket,
        Key=key,
        PartNumber=part_number,
        UploadId=upload_id,
        Body=chunk_data
    )

# Upload chunks in parallel
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = []
    for i, chunk in enumerate(chunks):
        future = executor.submit(upload_chunk, chunk, i+1)
        futures.append(future)
    
    # Wait for all uploads
    for future in futures:
        future.result()
```

---

## 9. Common Interview Questions + Answers

### Q: How would you handle large file uploads (>1 GB)?

> "I'd use chunked uploads with S3 multipart upload. Split the file into 10 MB chunks and upload them in parallel. This allows resuming on failure by re-uploading only failed chunks, not the entire file. Use presigned URLs so clients upload directly to S3, bypassing the server to save bandwidth. Implement the TUS protocol for standardized resumable uploads. Monitor upload progress and provide feedback to users. Set appropriate timeouts and retry failed chunks with exponential backoff."

### Q: What's the difference between direct upload and presigned URL upload?

> "Direct upload means the file goes through your server to storage — the server receives the file, validates it, and uploads to S3. This uses server bandwidth and can be a bottleneck. Presigned URL upload means the client uploads directly to S3 using a temporary signed URL generated by your server. This saves server bandwidth and scales better since S3 handles the load. The trade-off is you can't validate the file before upload, only after. Use presigned URLs for large files and direct upload when you need validation before storage."

### Q: How do you implement resumable file uploads?

> "Use the TUS protocol or S3 multipart upload. With TUS, the client creates an upload session and gets an upload URL. It uploads chunks and the server tracks the offset. If the upload is interrupted, the client queries the current offset and resumes from there. With S3 multipart, initiate a multipart upload to get an upload ID, upload parts with part numbers, and complete the upload by providing the list of parts. Both approaches allow resuming by re-uploading only the missing parts, not the entire file."

### Q: How do you secure file uploads?

> "Validate file type by checking the MIME type from the file content, not the extension — use libraries like python-magic. Enforce file size limits at multiple layers: web server (nginx), application, and storage. Scan files for viruses using ClamAV or similar. Sanitize filenames to prevent path traversal attacks. Use presigned URLs with short expiration times (15 minutes). Store files with random UUIDs, not user-provided names. Implement rate limiting to prevent abuse. For sensitive files, encrypt at rest and in transit."

---

## 10. Quick Reference

```
Upload strategies:
  Direct: Client → Server → S3 (simple, server bandwidth)
  Presigned URL: Client → S3 directly (scalable, no validation)
  Chunked: Split into parts (resumable, parallel)

Presigned URLs:
  Server generates temporary signed URL
  Client uploads/downloads directly to/from S3
  Expires after specified time (15 min - 1 hour)

Multipart upload (S3):
  1. Initiate (get upload_id)
  2. Upload parts (parallel, resumable)
  3. Complete (assemble parts)
  Benefits: Resume, parallel, files > 5 GB

Resumable upload (TUS):
  Client tracks offset
  Resume from last successful offset
  Standardized protocol

File download:
  Direct: Server streams from S3 (server bandwidth)
  Presigned URL: Client downloads from S3 (scalable)
  Range requests: Partial downloads (resumable, streaming)

Security:
  Validate file type (MIME, not extension)
  Enforce size limits (nginx, app, storage)
  Virus scanning (ClamAV)
  Sanitize filenames (prevent path traversal)
  Use UUIDs for storage (not user filenames)

Performance:
  CDN for downloads (CloudFront)
  Compression (gzip)
  Parallel uploads (ThreadPoolExecutor)
  Chunked transfer encoding

Best practices:
  - Use presigned URLs for large files
  - Implement chunked/resumable uploads
  - Validate and sanitize all inputs
  - Use CDN for downloads
  - Monitor upload/download metrics
  - Set appropriate timeouts
```
