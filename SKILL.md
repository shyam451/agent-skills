---
name: unstructured-artifact-processor
description: Process and extract structured data from any unstructured artifact type including emails (.eml, .msg), Word documents (.docx, .doc with embedded objects), Excel spreadsheets (.xlsx, .xls with embedded files), PDFs (with images and tables), archives (ZIP, RAR, 7z with nested contents), multimedia files (images, audio, video), and compound documents containing multiple embedded artifacts. Handles recursive extraction of nested content (documents with embedded ZIPs, ZIPs within ZIPs, embedded OLE objects). Use for requests like "extract data from this email", "process these documents", "what's in this archive", "extract all content from this file", "read and query these unstructured documents", "parse this artifact", "extract embedded files", "process multimedia metadata". Outputs unified JSON with content, metadata, embedded artifacts, and source citations.
---

# Unstructured Artifact Processor

Extract, parse, and query data from any unstructured document type with recursive embedded content handling and unified JSON output.

## Supported Artifact Types

| Category | Extensions | Key Capabilities |
|----------|------------|------------------|
| Email | .eml, .msg | Headers, body (HTML/plain), attachments, threading |
| Documents | .docx, .doc, .odt, .rtf | Text, tables, images, tracked changes, embedded OLE |
| Spreadsheets | .xlsx, .xls, .xlsm, .csv, .tsv | Sheets, formulas, charts, embedded objects |
| PDFs | .pdf | Text, tables, images, forms, annotations, embedded files |
| Archives | .zip, .rar, .7z, .tar, .gz | Recursive extraction, nested content, file manifest |
| Images | .png, .jpg, .tiff, .bmp, .gif | OCR text, EXIF metadata, embedded data |
| Audio/Video | .mp3, .mp4, .wav, .avi, .mov | Metadata, duration, codec info, embedded artwork |
| Structured Data | .json, .xml, .yaml, .html | Parse and normalize content |

## Quick Start Workflow

```
1. IDENTIFY → Detect artifact type (extension + magic bytes)
2. EXTRACT → Apply type-specific extraction
3. RECURSE → Process any embedded/nested artifacts
4. NORMALIZE → Convert to unified JSON schema
5. CITE → Track source locations for all content
```

## Processing Workflow

### Step 1: Artifact Identification

Determine artifact type using extension and content inspection:

```python
import magic
import mimetypes
from pathlib import Path

def identify_artifact(filepath):
    """Identify artifact type using extension and magic bytes."""
    ext = Path(filepath).suffix.lower()
    mime = magic.from_file(filepath, mime=True)
    
    # Map to artifact category
    type_map = {
        # Email
        '.eml': 'email', '.msg': 'email',
        # Documents  
        '.docx': 'word', '.doc': 'word_legacy', '.odt': 'odt', '.rtf': 'rtf',
        # Spreadsheets
        '.xlsx': 'excel', '.xls': 'excel_legacy', '.xlsm': 'excel_macro',
        '.csv': 'csv', '.tsv': 'tsv',
        # PDF
        '.pdf': 'pdf',
        # Archives
        '.zip': 'archive', '.rar': 'archive', '.7z': 'archive',
        '.tar': 'archive', '.gz': 'archive', '.tar.gz': 'archive',
        # Images
        '.png': 'image', '.jpg': 'image', '.jpeg': 'image',
        '.tiff': 'image', '.bmp': 'image', '.gif': 'image',
        # Audio/Video
        '.mp3': 'audio', '.wav': 'audio', '.mp4': 'video',
        '.avi': 'video', '.mov': 'video', '.mkv': 'video',
        # Structured
        '.json': 'json', '.xml': 'xml', '.yaml': 'yaml', '.html': 'html',
    }
    return type_map.get(ext, 'unknown'), mime
```

### Step 2: Type-Specific Extraction

Select extraction method based on artifact type. See `references/extraction_methods.md` for detailed implementations.

**Email Processing:**
```python
import email
from email import policy

def extract_email(filepath):
    """Extract email content and attachments."""
    with open(filepath, 'rb') as f:
        msg = email.message_from_binary_file(f, policy=policy.default)
    
    result = {
        'type': 'email',
        'headers': {
            'from': msg['from'],
            'to': msg['to'],
            'subject': msg['subject'],
            'date': msg['date'],
            'message_id': msg['message-id'],
        },
        'body': {'plain': None, 'html': None},
        'attachments': []
    }
    
    # Extract body and attachments
    for part in msg.walk():
        content_type = part.get_content_type()
        if content_type == 'text/plain' and not result['body']['plain']:
            result['body']['plain'] = part.get_content()
        elif content_type == 'text/html' and not result['body']['html']:
            result['body']['html'] = part.get_content()
        elif part.get_filename():
            result['attachments'].append({
                'filename': part.get_filename(),
                'content_type': content_type,
                'size': len(part.get_payload(decode=True) or b''),
                'data': part.get_payload(decode=True)  # For recursive processing
            })
    return result
```

**Document Processing (Word):**
```python
import zipfile
from lxml import etree

def extract_docx(filepath):
    """Extract Word document content including embedded objects."""
    with zipfile.ZipFile(filepath) as zf:
        # Main document text
        doc_xml = zf.read('word/document.xml')
        tree = etree.fromstring(doc_xml)
        
        # Namespaces
        ns = {'w': 'http://schemas.openxmlformats.org/wordprocessingml/2006/main'}
        
        result = {
            'type': 'word_document',
            'text_content': [],
            'tables': [],
            'images': [],
            'embedded_objects': [],
            'metadata': {}
        }
        
        # Extract paragraphs
        for para in tree.findall('.//w:p', ns):
            text = ''.join(t.text or '' for t in para.findall('.//w:t', ns))
            if text.strip():
                result['text_content'].append(text)
        
        # Extract embedded OLE objects
        if 'word/embeddings/' in zf.namelist():
            for name in zf.namelist():
                if name.startswith('word/embeddings/'):
                    result['embedded_objects'].append({
                        'name': name,
                        'data': zf.read(name)
                    })
        
        # Extract images
        for name in zf.namelist():
            if name.startswith('word/media/'):
                result['images'].append({
                    'name': name,
                    'data': zf.read(name)
                })
    
    return result
```

**Archive Processing (Recursive):**
```python
import zipfile
import tempfile
import os

def extract_archive(filepath, max_depth=5, current_depth=0):
    """Recursively extract archive contents."""
    if current_depth >= max_depth:
        return {'type': 'archive', 'error': 'max_depth_exceeded'}
    
    result = {
        'type': 'archive',
        'contents': [],
        'nested_artifacts': []
    }
    
    with zipfile.ZipFile(filepath) as zf:
        for info in zf.infolist():
            entry = {
                'name': info.filename,
                'size': info.file_size,
                'compressed_size': info.compress_size,
                'is_dir': info.is_dir()
            }
            
            if not info.is_dir():
                # Extract to temp and process recursively
                with tempfile.TemporaryDirectory() as tmpdir:
                    extracted_path = zf.extract(info, tmpdir)
                    artifact_type, _ = identify_artifact(extracted_path)
                    
                    if artifact_type != 'unknown':
                        # Recursively process nested artifact
                        nested = process_artifact(extracted_path, 
                                                  max_depth, current_depth + 1)
                        entry['nested_content'] = nested
            
            result['contents'].append(entry)
    
    return result
```

### Step 3: Recursive Embedded Content Processing

Handle documents with embedded files at any nesting level:

```python
def process_artifact(filepath, max_depth=5, current_depth=0):
    """Main processing function with recursive handling."""
    artifact_type, mime = identify_artifact(filepath)
    
    # Route to appropriate extractor
    extractors = {
        'email': extract_email,
        'word': extract_docx,
        'excel': extract_xlsx,
        'pdf': extract_pdf,
        'archive': lambda f: extract_archive(f, max_depth, current_depth),
        'image': extract_image,
        'audio': extract_audio_metadata,
        'video': extract_video_metadata,
        'csv': extract_csv,
        'json': extract_json,
        'xml': extract_xml,
    }
    
    extractor = extractors.get(artifact_type, extract_generic)
    result = extractor(filepath)
    
    # Process any embedded artifacts recursively
    if 'embedded_objects' in result:
        for obj in result['embedded_objects']:
            if 'data' in obj and current_depth < max_depth:
                with tempfile.NamedTemporaryFile(delete=False) as tmp:
                    tmp.write(obj['data'])
                    tmp.flush()
                    obj['extracted_content'] = process_artifact(
                        tmp.name, max_depth, current_depth + 1
                    )
                    os.unlink(tmp.name)
    
    if 'attachments' in result:
        for att in result['attachments']:
            if 'data' in att and current_depth < max_depth:
                with tempfile.NamedTemporaryFile(delete=False) as tmp:
                    tmp.write(att['data'])
                    tmp.flush()
                    att['extracted_content'] = process_artifact(
                        tmp.name, max_depth, current_depth + 1
                    )
                    os.unlink(tmp.name)
    
    return result
```

### Step 4: Unified Output Schema

All artifacts normalize to this structure:

```json
{
  "artifact_type": "email|word|excel|pdf|archive|image|audio|video|...",
  "source_file": "original_filename.ext",
  "processing_timestamp": "2026-01-30T12:00:00Z",
  "metadata": {
    "created": "...",
    "modified": "...",
    "author": "...",
    "size_bytes": 12345,
    "mime_type": "...",
    "custom_properties": {}
  },
  "content": {
    "text": ["paragraph 1", "paragraph 2"],
    "tables": [{"headers": [...], "rows": [...]}],
    "structured_data": {}
  },
  "embedded_artifacts": [
    {
      "name": "attachment.xlsx",
      "type": "excel",
      "citation": {"location": "email_attachment", "index": 0},
      "extracted_content": { /* recursive structure */ }
    }
  ],
  "citations": [
    {"field": "content.text[0]", "source": "page_1", "coordinates": {...}}
  ],
  "processing_notes": {
    "extraction_method": "...",
    "warnings": [],
    "errors": []
  }
}
```

## Querying Extracted Data

After extraction, query the unified JSON:

```python
def query_artifact(extracted_data, query_type, **kwargs):
    """Query extracted artifact data."""
    
    if query_type == 'text_search':
        # Search across all text content
        pattern = kwargs.get('pattern', '')
        results = []
        for i, text in enumerate(extracted_data.get('content', {}).get('text', [])):
            if pattern.lower() in text.lower():
                results.append({'index': i, 'text': text})
        return results
    
    elif query_type == 'get_attachments':
        # List all embedded artifacts
        return extracted_data.get('embedded_artifacts', [])
    
    elif query_type == 'get_tables':
        # Extract all tables
        return extracted_data.get('content', {}).get('tables', [])
    
    elif query_type == 'get_metadata':
        # Return metadata
        return extracted_data.get('metadata', {})
    
    elif query_type == 'flatten_text':
        # Get all text as single string
        texts = extracted_data.get('content', {}).get('text', [])
        return '\n\n'.join(texts)
```

## Special Handling

### Documents with Embedded ZIPs

Word/Excel documents may contain embedded ZIP files (e.g., data packages):

```python
def handle_embedded_zip_in_docx(docx_result):
    """Process ZIP files embedded as OLE objects in Word docs."""
    for obj in docx_result.get('embedded_objects', []):
        if obj['name'].endswith('.zip') or is_zip_data(obj['data']):
            with tempfile.NamedTemporaryFile(suffix='.zip', delete=False) as tmp:
                tmp.write(obj['data'])
                tmp.flush()
                obj['zip_contents'] = extract_archive(tmp.name)
                os.unlink(tmp.name)
```

### Multimedia Metadata

Extract metadata without processing full media:

```python
from mutagen import File as MutagenFile
from PIL import Image
from PIL.ExifTags import TAGS

def extract_image_metadata(filepath):
    """Extract image metadata including EXIF and OCR."""
    result = {'type': 'image', 'metadata': {}, 'ocr_text': None}
    
    with Image.open(filepath) as img:
        result['metadata']['format'] = img.format
        result['metadata']['size'] = img.size
        result['metadata']['mode'] = img.mode
        
        # EXIF data
        exif = img._getexif()
        if exif:
            result['metadata']['exif'] = {
                TAGS.get(k, k): v for k, v in exif.items()
            }
    
    # OCR (optional)
    try:
        import pytesseract
        result['ocr_text'] = pytesseract.image_to_string(Image.open(filepath))
    except Exception:
        pass
    
    return result

def extract_audio_video_metadata(filepath):
    """Extract audio/video metadata."""
    audio = MutagenFile(filepath)
    if audio:
        return {
            'type': 'audio' if hasattr(audio.info, 'channels') else 'video',
            'metadata': {
                'duration': getattr(audio.info, 'length', None),
                'bitrate': getattr(audio.info, 'bitrate', None),
                'sample_rate': getattr(audio.info, 'sample_rate', None),
                'channels': getattr(audio.info, 'channels', None),
                'tags': dict(audio.tags) if audio.tags else {}
            }
        }
```

### MSG Files (Outlook)

```python
import extract_msg

def extract_msg_file(filepath):
    """Extract Outlook MSG file."""
    msg = extract_msg.Message(filepath)
    return {
        'type': 'email',
        'headers': {
            'from': msg.sender,
            'to': msg.to,
            'cc': msg.cc,
            'subject': msg.subject,
            'date': msg.date,
        },
        'body': {
            'plain': msg.body,
            'html': msg.htmlBody,
        },
        'attachments': [
            {
                'filename': att.longFilename or att.shortFilename,
                'data': att.data,
                'size': len(att.data) if att.data else 0
            }
            for att in msg.attachments
        ]
    }
```

## Error Handling

```python
def safe_process_artifact(filepath, max_depth=5):
    """Process artifact with comprehensive error handling."""
    try:
        result = process_artifact(filepath, max_depth)
        result['processing_notes'] = {'status': 'success'}
        return result
    except zipfile.BadZipFile:
        return {'error': 'corrupt_archive', 'source_file': filepath}
    except PermissionError:
        return {'error': 'access_denied', 'source_file': filepath}
    except UnicodeDecodeError:
        return {'error': 'encoding_error', 'source_file': filepath}
    except Exception as e:
        return {
            'error': 'processing_failed',
            'error_type': type(e).__name__,
            'message': str(e),
            'source_file': filepath
        }
```

## Dependencies

Install required packages:

```bash
pip install python-magic lxml openpyxl pdfplumber pytesseract pillow \
    mutagen extract-msg pandas python-docx --break-system-packages
```

For MSG files: `pip install extract-msg --break-system-packages`
For OCR: Ensure tesseract is installed: `apt-get install tesseract-ocr`

## References

- **extraction_methods.md**: Detailed extraction implementations for each type
- **output_schemas.md**: Complete JSON schemas for each artifact type
- **query_patterns.md**: Advanced querying and filtering examples
