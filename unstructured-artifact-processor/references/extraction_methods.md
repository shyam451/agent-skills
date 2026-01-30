# Extraction Methods Reference

Complete implementations for each artifact type.

## Email Extraction

### EML Files (RFC 822)

```python
import email
from email import policy
from email.utils import parsedate_to_datetime
import base64

def extract_eml_complete(filepath):
    """Complete EML extraction with threading support."""
    with open(filepath, 'rb') as f:
        msg = email.message_from_binary_file(f, policy=policy.default)
    
    result = {
        'type': 'email',
        'format': 'eml',
        'headers': extract_email_headers(msg),
        'body': extract_email_body(msg),
        'attachments': extract_email_attachments(msg),
        'threading': extract_threading_info(msg),
        'metadata': {
            'size_bytes': f.seek(0, 2),
            'has_attachments': bool(msg.get_payload()) and msg.is_multipart()
        }
    }
    return result

def extract_email_headers(msg):
    """Extract all email headers."""
    return {
        'from': msg['from'],
        'to': msg['to'],
        'cc': msg['cc'],
        'bcc': msg['bcc'],
        'subject': msg['subject'],
        'date': msg['date'],
        'message_id': msg['message-id'],
        'in_reply_to': msg['in-reply-to'],
        'references': msg['references'],
        'content_type': msg['content-type'],
        'x_mailer': msg['x-mailer'],
        'x_priority': msg['x-priority'],
        'return_path': msg['return-path'],
        'received': msg.get_all('received', []),
    }

def extract_email_body(msg):
    """Extract email body (plain and HTML)."""
    body = {'plain': None, 'html': None, 'raw_parts': []}
    
    if msg.is_multipart():
        for part in msg.walk():
            content_type = part.get_content_type()
            disposition = str(part.get('Content-Disposition', ''))
            
            if 'attachment' not in disposition:
                if content_type == 'text/plain' and not body['plain']:
                    body['plain'] = part.get_content()
                elif content_type == 'text/html' and not body['html']:
                    body['html'] = part.get_content()
                body['raw_parts'].append({
                    'content_type': content_type,
                    'size': len(str(part.get_payload()))
                })
    else:
        content_type = msg.get_content_type()
        content = msg.get_content()
        if content_type == 'text/plain':
            body['plain'] = content
        elif content_type == 'text/html':
            body['html'] = content
    
    return body

def extract_email_attachments(msg):
    """Extract all email attachments with metadata."""
    attachments = []
    
    for part in msg.walk():
        filename = part.get_filename()
        if filename:
            payload = part.get_payload(decode=True)
            attachments.append({
                'filename': filename,
                'content_type': part.get_content_type(),
                'size': len(payload) if payload else 0,
                'content_id': part.get('Content-ID'),
                'data': payload,  # Binary data for recursive processing
                'encoding': part.get('Content-Transfer-Encoding'),
            })
    
    return attachments

def extract_threading_info(msg):
    """Extract email threading/conversation info."""
    return {
        'message_id': msg['message-id'],
        'in_reply_to': msg['in-reply-to'],
        'references': (msg['references'] or '').split(),
        'thread_index': msg.get('Thread-Index'),
    }
```

### MSG Files (Outlook)

```python
import extract_msg
import olefile

def extract_msg_complete(filepath):
    """Complete MSG extraction with properties."""
    msg = extract_msg.Message(filepath)
    
    result = {
        'type': 'email',
        'format': 'msg',
        'headers': {
            'from': msg.sender,
            'to': msg.to,
            'cc': msg.cc,
            'bcc': msg.bcc,
            'subject': msg.subject,
            'date': str(msg.date),
            'message_id': msg.messageId,
            'importance': msg.importance,
            'priority': msg.priority,
        },
        'body': {
            'plain': msg.body,
            'html': msg.htmlBody,
            'rtf': msg.rtfBody.decode('utf-8', errors='ignore') if msg.rtfBody else None,
        },
        'attachments': [],
        'properties': extract_msg_properties(msg),
    }
    
    for att in msg.attachments:
        result['attachments'].append({
            'filename': att.longFilename or att.shortFilename,
            'content_type': att.mimetype,
            'size': len(att.data) if att.data else 0,
            'data': att.data,
            'is_embedded': att.cid is not None,
            'content_id': att.cid,
        })
    
    return result

def extract_msg_properties(msg):
    """Extract extended MSG properties."""
    return {
        'categories': msg.categories,
        'sender_email': msg.senderEmail,
        'received_time': str(msg.receivedTime) if msg.receivedTime else None,
        'sent_time': str(msg.sentTime) if msg.sentTime else None,
        'conversation_topic': msg.conversationTopic,
    }
```

## Document Extraction

### DOCX (Word 2007+)

```python
import zipfile
from lxml import etree
import json

WORD_NS = {
    'w': 'http://schemas.openxmlformats.org/wordprocessingml/2006/main',
    'wp': 'http://schemas.openxmlformats.org/drawingml/2006/wordprocessingDrawing',
    'a': 'http://schemas.openxmlformats.org/drawingml/2006/main',
    'r': 'http://schemas.openxmlformats.org/officeDocument/2006/relationships',
    'cp': 'http://schemas.openxmlformats.org/package/2006/metadata/core-properties',
    'dc': 'http://purl.org/dc/elements/1.1/',
}

def extract_docx_complete(filepath):
    """Complete DOCX extraction."""
    with zipfile.ZipFile(filepath) as zf:
        result = {
            'type': 'word_document',
            'format': 'docx',
            'content': extract_docx_content(zf),
            'metadata': extract_docx_metadata(zf),
            'images': extract_docx_images(zf),
            'embedded_objects': extract_docx_embedded(zf),
            'styles': extract_docx_styles(zf),
            'comments': extract_docx_comments(zf),
            'tracked_changes': extract_docx_changes(zf),
        }
    return result

def extract_docx_content(zf):
    """Extract text content with structure."""
    doc_xml = zf.read('word/document.xml')
    tree = etree.fromstring(doc_xml)
    
    content = {
        'paragraphs': [],
        'tables': [],
        'headers': [],
        'footers': [],
    }
    
    # Extract paragraphs with style info
    for para in tree.findall('.//w:p', WORD_NS):
        para_data = {
            'text': '',
            'style': None,
            'runs': []
        }
        
        # Get paragraph style
        pStyle = para.find('.//w:pStyle', WORD_NS)
        if pStyle is not None:
            para_data['style'] = pStyle.get('{%s}val' % WORD_NS['w'])
        
        # Extract runs (formatted text segments)
        for run in para.findall('.//w:r', WORD_NS):
            run_text = ''.join(t.text or '' for t in run.findall('.//w:t', WORD_NS))
            para_data['text'] += run_text
            para_data['runs'].append({
                'text': run_text,
                'bold': run.find('.//w:b', WORD_NS) is not None,
                'italic': run.find('.//w:i', WORD_NS) is not None,
            })
        
        if para_data['text'].strip():
            content['paragraphs'].append(para_data)
    
    # Extract tables
    for table in tree.findall('.//w:tbl', WORD_NS):
        table_data = {'rows': []}
        for row in table.findall('.//w:tr', WORD_NS):
            row_data = []
            for cell in row.findall('.//w:tc', WORD_NS):
                cell_text = ''.join(
                    t.text or '' 
                    for t in cell.findall('.//w:t', WORD_NS)
                )
                row_data.append(cell_text)
            table_data['rows'].append(row_data)
        content['tables'].append(table_data)
    
    return content

def extract_docx_metadata(zf):
    """Extract document metadata."""
    metadata = {}
    
    try:
        core_xml = zf.read('docProps/core.xml')
        tree = etree.fromstring(core_xml)
        
        metadata['title'] = tree.findtext('.//dc:title', namespaces=WORD_NS)
        metadata['creator'] = tree.findtext('.//dc:creator', namespaces=WORD_NS)
        metadata['subject'] = tree.findtext('.//dc:subject', namespaces=WORD_NS)
        metadata['created'] = tree.findtext('.//dcterms:created', 
                                            namespaces={'dcterms': 'http://purl.org/dc/terms/'})
        metadata['modified'] = tree.findtext('.//dcterms:modified',
                                             namespaces={'dcterms': 'http://purl.org/dc/terms/'})
    except KeyError:
        pass
    
    try:
        app_xml = zf.read('docProps/app.xml')
        tree = etree.fromstring(app_xml)
        metadata['pages'] = tree.findtext('.//{*}Pages')
        metadata['words'] = tree.findtext('.//{*}Words')
        metadata['application'] = tree.findtext('.//{*}Application')
    except KeyError:
        pass
    
    return metadata

def extract_docx_images(zf):
    """Extract images from document."""
    images = []
    for name in zf.namelist():
        if name.startswith('word/media/'):
            images.append({
                'name': name.split('/')[-1],
                'path': name,
                'size': zf.getinfo(name).file_size,
                'data': zf.read(name),
            })
    return images

def extract_docx_embedded(zf):
    """Extract embedded OLE objects."""
    embedded = []
    for name in zf.namelist():
        if name.startswith('word/embeddings/'):
            embedded.append({
                'name': name.split('/')[-1],
                'path': name,
                'size': zf.getinfo(name).file_size,
                'data': zf.read(name),
            })
    return embedded

def extract_docx_comments(zf):
    """Extract document comments."""
    comments = []
    try:
        comments_xml = zf.read('word/comments.xml')
        tree = etree.fromstring(comments_xml)
        for comment in tree.findall('.//w:comment', WORD_NS):
            comments.append({
                'id': comment.get('{%s}id' % WORD_NS['w']),
                'author': comment.get('{%s}author' % WORD_NS['w']),
                'date': comment.get('{%s}date' % WORD_NS['w']),
                'text': ''.join(t.text or '' for t in comment.findall('.//w:t', WORD_NS)),
            })
    except KeyError:
        pass
    return comments

def extract_docx_changes(zf):
    """Extract tracked changes."""
    changes = {'insertions': [], 'deletions': []}
    doc_xml = zf.read('word/document.xml')
    tree = etree.fromstring(doc_xml)
    
    for ins in tree.findall('.//w:ins', WORD_NS):
        changes['insertions'].append({
            'author': ins.get('{%s}author' % WORD_NS['w']),
            'date': ins.get('{%s}date' % WORD_NS['w']),
            'text': ''.join(t.text or '' for t in ins.findall('.//w:t', WORD_NS)),
        })
    
    for delete in tree.findall('.//w:del', WORD_NS):
        changes['deletions'].append({
            'author': delete.get('{%s}author' % WORD_NS['w']),
            'date': delete.get('{%s}date' % WORD_NS['w']),
            'text': ''.join(t.text or '' for t in delete.findall('.//w:delText', WORD_NS)),
        })
    
    return changes

def extract_docx_styles(zf):
    """Extract document styles."""
    styles = []
    try:
        styles_xml = zf.read('word/styles.xml')
        tree = etree.fromstring(styles_xml)
        for style in tree.findall('.//w:style', WORD_NS):
            styles.append({
                'id': style.get('{%s}styleId' % WORD_NS['w']),
                'type': style.get('{%s}type' % WORD_NS['w']),
                'name': style.findtext('.//w:name/@w:val', namespaces=WORD_NS),
            })
    except KeyError:
        pass
    return styles
```

### DOC (Legacy Word)

```python
import olefile
import subprocess

def extract_doc_legacy(filepath):
    """Extract legacy .doc file."""
    result = {
        'type': 'word_document',
        'format': 'doc',
        'content': {'text': '', 'tables': []},
        'metadata': {},
        'embedded_objects': [],
    }
    
    # Use antiword or catdoc for text extraction
    try:
        text = subprocess.run(
            ['antiword', filepath],
            capture_output=True, text=True, timeout=30
        )
        result['content']['text'] = text.stdout
    except FileNotFoundError:
        try:
            text = subprocess.run(
                ['catdoc', filepath],
                capture_output=True, text=True, timeout=30
            )
            result['content']['text'] = text.stdout
        except FileNotFoundError:
            pass
    
    # Extract OLE metadata
    if olefile.isOleFile(filepath):
        ole = olefile.OleFileIO(filepath)
        result['metadata'] = dict(ole.get_metadata().__dict__)
        
        # List embedded objects
        for entry in ole.listdir():
            if 'ObjectPool' in entry or 'Package' in entry:
                result['embedded_objects'].append({
                    'name': '/'.join(entry),
                    'size': ole.get_size('/'.join(entry)),
                })
        ole.close()
    
    return result
```

## Spreadsheet Extraction

### XLSX (Excel 2007+)

```python
import openpyxl
import zipfile
from lxml import etree

def extract_xlsx_complete(filepath):
    """Complete XLSX extraction."""
    result = {
        'type': 'spreadsheet',
        'format': 'xlsx',
        'sheets': [],
        'metadata': {},
        'embedded_objects': [],
        'charts': [],
        'named_ranges': {},
    }
    
    wb = openpyxl.load_workbook(filepath, data_only=True)
    
    # Extract sheets
    for sheet_name in wb.sheetnames:
        ws = wb[sheet_name]
        sheet_data = {
            'name': sheet_name,
            'rows': [],
            'merged_cells': list(str(m) for m in ws.merged_cells.ranges),
            'dimensions': ws.dimensions,
        }
        
        for row in ws.iter_rows(values_only=True):
            sheet_data['rows'].append(list(row))
        
        result['sheets'].append(sheet_data)
    
    # Extract metadata and embedded objects via zipfile
    with zipfile.ZipFile(filepath) as zf:
        result['metadata'] = extract_xlsx_metadata(zf)
        result['embedded_objects'] = extract_xlsx_embedded(zf)
    
    # Named ranges
    for name in wb.defined_names.definedName:
        result['named_ranges'][name.name] = str(name.value)
    
    wb.close()
    return result

def extract_xlsx_metadata(zf):
    """Extract XLSX metadata."""
    metadata = {}
    try:
        core_xml = zf.read('docProps/core.xml')
        tree = etree.fromstring(core_xml)
        metadata['creator'] = tree.findtext('.//{*}creator')
        metadata['created'] = tree.findtext('.//{*}created')
        metadata['modified'] = tree.findtext('.//{*}modified')
    except KeyError:
        pass
    return metadata

def extract_xlsx_embedded(zf):
    """Extract embedded objects from XLSX."""
    embedded = []
    for name in zf.namelist():
        if 'embeddings' in name or 'oleObject' in name:
            embedded.append({
                'name': name.split('/')[-1],
                'path': name,
                'size': zf.getinfo(name).file_size,
                'data': zf.read(name),
            })
    return embedded
```

### CSV/TSV

```python
import pandas as pd
import csv

def extract_csv_complete(filepath, delimiter=','):
    """Extract CSV/TSV with encoding detection."""
    import chardet
    
    with open(filepath, 'rb') as f:
        raw = f.read()
        encoding = chardet.detect(raw)['encoding']
    
    df = pd.read_csv(filepath, encoding=encoding, delimiter=delimiter)
    
    return {
        'type': 'spreadsheet',
        'format': 'csv' if delimiter == ',' else 'tsv',
        'encoding': encoding,
        'columns': list(df.columns),
        'row_count': len(df),
        'data': df.to_dict(orient='records'),
        'dtypes': {col: str(dtype) for col, dtype in df.dtypes.items()},
    }
```

## PDF Extraction

```python
import pdfplumber
from pypdf import PdfReader

def extract_pdf_complete(filepath):
    """Complete PDF extraction."""
    result = {
        'type': 'pdf',
        'content': {'pages': [], 'tables': []},
        'metadata': {},
        'images': [],
        'forms': {},
        'embedded_files': [],
        'annotations': [],
    }
    
    # Text and tables with pdfplumber
    with pdfplumber.open(filepath) as pdf:
        for i, page in enumerate(pdf.pages):
            page_data = {
                'number': i + 1,
                'text': page.extract_text() or '',
                'tables': [],
                'dimensions': {'width': page.width, 'height': page.height},
            }
            
            tables = page.extract_tables()
            for table in tables:
                if table:
                    page_data['tables'].append({
                        'headers': table[0] if table else [],
                        'rows': table[1:] if len(table) > 1 else [],
                    })
            
            result['content']['pages'].append(page_data)
    
    # Metadata and forms with pypdf
    reader = PdfReader(filepath)
    
    if reader.metadata:
        result['metadata'] = {
            'title': reader.metadata.get('/Title'),
            'author': reader.metadata.get('/Author'),
            'subject': reader.metadata.get('/Subject'),
            'creator': reader.metadata.get('/Creator'),
            'producer': reader.metadata.get('/Producer'),
            'creation_date': str(reader.metadata.get('/CreationDate')),
        }
    
    # Form fields
    if reader.get_form_text_fields():
        result['forms'] = reader.get_form_text_fields()
    
    # Embedded files
    if '/Names' in reader.trailer['/Root']:
        names = reader.trailer['/Root']['/Names']
        if '/EmbeddedFiles' in names:
            result['embedded_files'] = extract_pdf_embedded_files(reader)
    
    return result

def extract_pdf_embedded_files(reader):
    """Extract embedded files from PDF."""
    embedded = []
    # Implementation depends on PDF structure
    return embedded
```

## Archive Extraction

```python
import zipfile
import tarfile
import py7zr
import rarfile

def extract_archive_complete(filepath, max_depth=5, current_depth=0):
    """Universal archive extraction."""
    ext = filepath.lower().split('.')[-1]
    
    if ext == 'zip':
        return extract_zip(filepath, max_depth, current_depth)
    elif ext in ('tar', 'gz', 'tgz', 'bz2'):
        return extract_tar(filepath, max_depth, current_depth)
    elif ext == '7z':
        return extract_7z(filepath, max_depth, current_depth)
    elif ext == 'rar':
        return extract_rar(filepath, max_depth, current_depth)
    
    return {'type': 'archive', 'error': 'unsupported_format'}

def extract_zip(filepath, max_depth, current_depth):
    """Extract ZIP archive."""
    result = {
        'type': 'archive',
        'format': 'zip',
        'contents': [],
        'total_files': 0,
        'total_size': 0,
    }
    
    with zipfile.ZipFile(filepath) as zf:
        for info in zf.infolist():
            entry = {
                'name': info.filename,
                'size': info.file_size,
                'compressed_size': info.compress_size,
                'is_dir': info.is_dir(),
                'modified': str(info.date_time),
                'compression': info.compress_type,
            }
            
            result['total_files'] += 1
            result['total_size'] += info.file_size
            
            # Recursive processing for nested files
            if not info.is_dir() and current_depth < max_depth:
                entry['nested_content'] = process_nested_file(
                    zf.read(info), info.filename, max_depth, current_depth
                )
            
            result['contents'].append(entry)
    
    return result

def process_nested_file(data, filename, max_depth, current_depth):
    """Process nested file from archive."""
    import tempfile
    import os
    
    ext = filename.lower().split('.')[-1]
    
    with tempfile.NamedTemporaryFile(suffix=f'.{ext}', delete=False) as tmp:
        tmp.write(data)
        tmp.flush()
        
        try:
            from . import process_artifact
            result = process_artifact(tmp.name, max_depth, current_depth + 1)
        finally:
            os.unlink(tmp.name)
    
    return result
```

## Image Extraction

```python
from PIL import Image
from PIL.ExifTags import TAGS, GPSTAGS
import pytesseract

def extract_image_complete(filepath):
    """Complete image extraction with OCR."""
    result = {
        'type': 'image',
        'metadata': {},
        'exif': {},
        'gps': {},
        'ocr_text': None,
        'embedded_data': None,
    }
    
    with Image.open(filepath) as img:
        result['metadata'] = {
            'format': img.format,
            'mode': img.mode,
            'width': img.width,
            'height': img.height,
            'is_animated': getattr(img, 'is_animated', False),
            'n_frames': getattr(img, 'n_frames', 1),
        }
        
        # EXIF data
        exif_data = img._getexif()
        if exif_data:
            for tag_id, value in exif_data.items():
                tag = TAGS.get(tag_id, tag_id)
                if tag == 'GPSInfo':
                    result['gps'] = decode_gps_info(value)
                else:
                    try:
                        result['exif'][tag] = str(value)
                    except:
                        pass
        
        # OCR
        try:
            result['ocr_text'] = pytesseract.image_to_string(img)
        except Exception as e:
            result['ocr_error'] = str(e)
    
    return result

def decode_gps_info(gps_info):
    """Decode GPS EXIF data."""
    gps = {}
    for key, value in gps_info.items():
        tag = GPSTAGS.get(key, key)
        gps[tag] = value
    return gps
```

## Audio/Video Extraction

```python
from mutagen import File as MutagenFile
from mutagen.mp3 import MP3
from mutagen.mp4 import MP4

def extract_audio_complete(filepath):
    """Extract audio metadata."""
    audio = MutagenFile(filepath)
    
    result = {
        'type': 'audio',
        'format': filepath.split('.')[-1].lower(),
        'metadata': {
            'duration': getattr(audio.info, 'length', None),
            'bitrate': getattr(audio.info, 'bitrate', None),
            'sample_rate': getattr(audio.info, 'sample_rate', None),
            'channels': getattr(audio.info, 'channels', None),
        },
        'tags': {},
        'embedded_artwork': [],
    }
    
    if audio.tags:
        for key, value in audio.tags.items():
            result['tags'][key] = str(value)
    
    # Extract embedded artwork
    if isinstance(audio, MP3):
        for key in audio.tags.keys():
            if key.startswith('APIC'):
                result['embedded_artwork'].append({
                    'type': audio.tags[key].type,
                    'mime': audio.tags[key].mime,
                    'size': len(audio.tags[key].data),
                })
    
    return result

def extract_video_complete(filepath):
    """Extract video metadata."""
    import subprocess
    import json
    
    result = {
        'type': 'video',
        'format': filepath.split('.')[-1].lower(),
        'metadata': {},
        'streams': [],
    }
    
    # Use ffprobe for detailed info
    try:
        probe = subprocess.run([
            'ffprobe', '-v', 'quiet',
            '-print_format', 'json',
            '-show_format', '-show_streams',
            filepath
        ], capture_output=True, text=True, timeout=30)
        
        if probe.returncode == 0:
            data = json.loads(probe.stdout)
            result['metadata'] = data.get('format', {})
            result['streams'] = data.get('streams', [])
    except (FileNotFoundError, subprocess.TimeoutExpired):
        # Fallback to mutagen
        audio = MutagenFile(filepath)
        if audio:
            result['metadata'] = {
                'duration': getattr(audio.info, 'length', None),
                'bitrate': getattr(audio.info, 'bitrate', None),
            }
    
    return result
```
