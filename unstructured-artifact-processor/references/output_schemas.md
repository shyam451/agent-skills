# Output Schemas Reference

JSON schemas for each artifact type's extracted data.

## Base Schema

All artifacts include these common fields:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "BaseArtifact",
  "type": "object",
  "required": ["artifact_type", "source_file", "processing_timestamp"],
  "properties": {
    "artifact_type": {
      "type": "string",
      "enum": ["email", "word", "excel", "pdf", "archive", "image", "audio", "video", "csv", "json", "xml", "unknown"]
    },
    "source_file": {
      "type": "string",
      "description": "Original filename"
    },
    "processing_timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "size_bytes": {"type": "integer"},
        "mime_type": {"type": "string"},
        "created": {"type": "string"},
        "modified": {"type": "string"}
      }
    },
    "processing_notes": {
      "type": "object",
      "properties": {
        "status": {"type": "string", "enum": ["success", "partial", "failed"]},
        "warnings": {"type": "array", "items": {"type": "string"}},
        "errors": {"type": "array", "items": {"type": "string"}}
      }
    }
  }
}
```

## Email Schema

```json
{
  "artifact_type": "email",
  "format": "eml|msg",
  "headers": {
    "from": "sender@example.com",
    "to": "recipient@example.com",
    "cc": "cc@example.com",
    "bcc": "bcc@example.com",
    "subject": "Email subject",
    "date": "2026-01-30T10:00:00Z",
    "message_id": "<unique-id@domain>",
    "in_reply_to": "<parent-id@domain>",
    "references": ["<ref1>", "<ref2>"],
    "importance": "normal|high|low",
    "x_mailer": "Client name"
  },
  "body": {
    "plain": "Plain text content...",
    "html": "<html>...</html>",
    "rtf": "RTF content (MSG only)"
  },
  "attachments": [
    {
      "filename": "document.pdf",
      "content_type": "application/pdf",
      "size": 12345,
      "content_id": "cid:image001 (for inline)",
      "is_embedded": false,
      "extracted_content": { /* recursive artifact */ }
    }
  ],
  "threading": {
    "message_id": "<id>",
    "in_reply_to": "<parent>",
    "references": ["<ref1>"],
    "thread_index": "base64-thread-index"
  }
}
```

## Word Document Schema

```json
{
  "artifact_type": "word",
  "format": "docx|doc",
  "content": {
    "paragraphs": [
      {
        "text": "Paragraph content",
        "style": "Heading1|Normal|etc",
        "runs": [
          {
            "text": "formatted text",
            "bold": true,
            "italic": false,
            "underline": false,
            "font": "Arial",
            "size": 12
          }
        ]
      }
    ],
    "tables": [
      {
        "rows": [
          ["Cell 1", "Cell 2"],
          ["Cell 3", "Cell 4"]
        ],
        "headers": ["Header 1", "Header 2"],
        "merged_cells": []
      }
    ],
    "headers": [{"text": "Header content"}],
    "footers": [{"text": "Footer content"}]
  },
  "metadata": {
    "title": "Document Title",
    "author": "Author Name",
    "creator": "Application",
    "created": "2026-01-30T10:00:00Z",
    "modified": "2026-01-30T12:00:00Z",
    "pages": 10,
    "words": 5000,
    "revision": 5
  },
  "images": [
    {
      "name": "image1.png",
      "path": "word/media/image1.png",
      "size": 12345,
      "dimensions": {"width": 800, "height": 600},
      "extracted_content": { /* optional OCR */ }
    }
  ],
  "embedded_objects": [
    {
      "name": "oleObject1.bin",
      "type": "xlsx|pdf|etc",
      "size": 54321,
      "extracted_content": { /* recursive artifact */ }
    }
  ],
  "comments": [
    {
      "id": "1",
      "author": "Reviewer",
      "date": "2026-01-30T11:00:00Z",
      "text": "Comment content",
      "anchor_text": "Referenced text"
    }
  ],
  "tracked_changes": {
    "insertions": [
      {
        "author": "Editor",
        "date": "2026-01-30T11:30:00Z",
        "text": "inserted text"
      }
    ],
    "deletions": [
      {
        "author": "Editor",
        "date": "2026-01-30T11:30:00Z",
        "text": "deleted text"
      }
    ]
  }
}
```

## Excel Spreadsheet Schema

```json
{
  "artifact_type": "excel",
  "format": "xlsx|xls|xlsm",
  "sheets": [
    {
      "name": "Sheet1",
      "dimensions": "A1:Z100",
      "rows": [
        ["Header1", "Header2", "Header3"],
        ["Data1", "Data2", "Data3"]
      ],
      "merged_cells": ["A1:B1", "C1:D1"],
      "formulas": {
        "C2": "=A2+B2",
        "C3": "=SUM(A2:B3)"
      },
      "formatting": {
        "header_row": 1,
        "data_types": {
          "A": "string",
          "B": "number",
          "C": "formula"
        }
      }
    }
  ],
  "metadata": {
    "creator": "Author",
    "created": "2026-01-30T10:00:00Z",
    "modified": "2026-01-30T12:00:00Z",
    "application": "Microsoft Excel"
  },
  "named_ranges": {
    "DataRange": "Sheet1!$A$1:$C$100",
    "TotalCell": "Sheet1!$D$101"
  },
  "charts": [
    {
      "name": "Chart1",
      "type": "bar|line|pie",
      "data_range": "Sheet1!$A$1:$B$10"
    }
  ],
  "embedded_objects": [
    {
      "name": "oleObject1.bin",
      "type": "unknown",
      "size": 12345,
      "extracted_content": { /* recursive */ }
    }
  ],
  "pivot_tables": [
    {
      "name": "PivotTable1",
      "source_range": "Sheet1!$A$1:$D$100",
      "location": "Sheet2!$A$1"
    }
  ]
}
```

## PDF Schema

```json
{
  "artifact_type": "pdf",
  "content": {
    "pages": [
      {
        "number": 1,
        "text": "Page content text...",
        "tables": [
          {
            "headers": ["Col1", "Col2"],
            "rows": [["a", "b"], ["c", "d"]]
          }
        ],
        "dimensions": {"width": 612, "height": 792},
        "annotations": [
          {
            "type": "highlight|comment|link",
            "content": "annotation text",
            "rect": [100, 200, 300, 220]
          }
        ]
      }
    ],
    "full_text": "Combined text from all pages..."
  },
  "metadata": {
    "title": "Document Title",
    "author": "Author Name",
    "subject": "Subject",
    "creator": "Creating Application",
    "producer": "PDF Producer",
    "creation_date": "2026-01-30T10:00:00Z",
    "modification_date": "2026-01-30T12:00:00Z",
    "page_count": 10,
    "encrypted": false
  },
  "forms": {
    "field_name": "filled_value",
    "checkbox_field": true,
    "dropdown_field": "selected_option"
  },
  "images": [
    {
      "page": 1,
      "index": 0,
      "size": 54321,
      "dimensions": {"width": 400, "height": 300}
    }
  ],
  "embedded_files": [
    {
      "name": "attachment.xlsx",
      "size": 12345,
      "extracted_content": { /* recursive */ }
    }
  ],
  "bookmarks": [
    {
      "title": "Chapter 1",
      "page": 1,
      "children": [
        {"title": "Section 1.1", "page": 3}
      ]
    }
  ]
}
```

## Archive Schema

```json
{
  "artifact_type": "archive",
  "format": "zip|rar|7z|tar|gz",
  "summary": {
    "total_files": 25,
    "total_dirs": 5,
    "total_size_uncompressed": 1048576,
    "total_size_compressed": 524288,
    "compression_ratio": 0.5
  },
  "contents": [
    {
      "name": "folder/document.docx",
      "size": 12345,
      "compressed_size": 6000,
      "is_dir": false,
      "modified": "2026-01-30T10:00:00Z",
      "compression": "deflate",
      "crc32": "abc12345",
      "nested_content": { /* recursive artifact */ }
    },
    {
      "name": "folder/",
      "is_dir": true,
      "modified": "2026-01-30T09:00:00Z"
    }
  ],
  "nested_archives": [
    {
      "name": "inner.zip",
      "depth": 1,
      "extracted_content": { /* recursive archive schema */ }
    }
  ],
  "encryption": {
    "encrypted": false,
    "method": null
  }
}
```

## Image Schema

```json
{
  "artifact_type": "image",
  "format": "png|jpg|tiff|bmp|gif",
  "metadata": {
    "width": 1920,
    "height": 1080,
    "mode": "RGB|RGBA|L",
    "dpi": [300, 300],
    "is_animated": false,
    "n_frames": 1,
    "bit_depth": 8
  },
  "exif": {
    "Make": "Camera Make",
    "Model": "Camera Model",
    "DateTime": "2026:01:30 10:00:00",
    "ExposureTime": "1/125",
    "FNumber": "2.8",
    "ISOSpeedRatings": 400,
    "FocalLength": "50mm",
    "Software": "Editing Software"
  },
  "gps": {
    "GPSLatitude": 37.7749,
    "GPSLongitude": -122.4194,
    "GPSAltitude": 10.5,
    "GPSTimeStamp": "10:00:00"
  },
  "ocr_text": "Extracted text from image...",
  "embedded_data": {
    "icc_profile": true,
    "xmp": true
  }
}
```

## Audio Schema

```json
{
  "artifact_type": "audio",
  "format": "mp3|wav|flac|aac|ogg",
  "metadata": {
    "duration_seconds": 245.5,
    "bitrate": 320000,
    "sample_rate": 44100,
    "channels": 2,
    "bits_per_sample": 16,
    "codec": "mp3"
  },
  "tags": {
    "title": "Song Title",
    "artist": "Artist Name",
    "album": "Album Name",
    "year": "2026",
    "track": "1/12",
    "genre": "Rock",
    "composer": "Composer",
    "comment": "Comment text"
  },
  "embedded_artwork": [
    {
      "type": "front_cover",
      "mime": "image/jpeg",
      "size": 54321,
      "dimensions": {"width": 500, "height": 500}
    }
  ],
  "chapters": [
    {
      "title": "Chapter 1",
      "start_time": 0,
      "end_time": 60.5
    }
  ]
}
```

## Video Schema

```json
{
  "artifact_type": "video",
  "format": "mp4|avi|mkv|mov|webm",
  "metadata": {
    "duration_seconds": 3600.5,
    "bitrate": 5000000,
    "file_size": 2147483648
  },
  "video_stream": {
    "codec": "h264",
    "width": 1920,
    "height": 1080,
    "frame_rate": 29.97,
    "bit_depth": 8,
    "color_space": "yuv420p"
  },
  "audio_streams": [
    {
      "codec": "aac",
      "channels": 2,
      "sample_rate": 48000,
      "bitrate": 128000,
      "language": "eng"
    }
  ],
  "subtitle_streams": [
    {
      "codec": "srt",
      "language": "eng"
    }
  ],
  "tags": {
    "title": "Video Title",
    "artist": "Creator",
    "date": "2026-01-30",
    "description": "Video description"
  },
  "chapters": [
    {
      "title": "Introduction",
      "start_time": 0,
      "end_time": 120
    }
  ]
}
```

## CSV/TSV Schema

```json
{
  "artifact_type": "csv",
  "format": "csv|tsv",
  "encoding": "utf-8",
  "delimiter": ",",
  "columns": ["col1", "col2", "col3"],
  "row_count": 1000,
  "column_types": {
    "col1": "string",
    "col2": "integer",
    "col3": "float"
  },
  "data": [
    {"col1": "value1", "col2": 123, "col3": 45.67},
    {"col1": "value2", "col2": 456, "col3": 89.01}
  ],
  "statistics": {
    "col2": {"min": 1, "max": 1000, "mean": 500.5},
    "col3": {"min": 0.1, "max": 99.9, "mean": 50.0}
  }
}
```

## Citation Schema

All schemas support field-level citations:

```json
{
  "citations": [
    {
      "field_path": "content.pages[0].text",
      "source": {
        "type": "page",
        "page_number": 1,
        "location": "top-left",
        "bounding_box": {
          "x": 72,
          "y": 72,
          "width": 468,
          "height": 648
        }
      },
      "confidence": 0.95,
      "extraction_method": "pdfplumber"
    },
    {
      "field_path": "headers.from",
      "source": {
        "type": "header",
        "header_name": "From",
        "raw_value": "John Doe <john@example.com>"
      },
      "confidence": 1.0,
      "extraction_method": "email_parser"
    }
  ]
}
```
