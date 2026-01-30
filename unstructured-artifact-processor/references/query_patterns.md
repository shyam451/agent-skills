# Query Patterns Reference

Advanced patterns for querying and filtering extracted artifact data.

## Basic Queries

### Text Search

```python
def search_text(artifact, pattern, case_sensitive=False):
    """Search for text across all content."""
    results = []
    
    def search_in_value(value, path):
        if isinstance(value, str):
            text = value if case_sensitive else value.lower()
            search_pattern = pattern if case_sensitive else pattern.lower()
            if search_pattern in text:
                results.append({
                    'path': path,
                    'text': value,
                    'match_start': text.find(search_pattern)
                })
        elif isinstance(value, list):
            for i, item in enumerate(value):
                search_in_value(item, f"{path}[{i}]")
        elif isinstance(value, dict):
            for key, val in value.items():
                search_in_value(val, f"{path}.{key}")
    
    search_in_value(artifact, 'root')
    return results
```

### Field Extraction

```python
def get_field(artifact, path):
    """Get field by dot-notation path with array indices."""
    import re
    
    parts = re.split(r'\.|\[|\]', path)
    parts = [p for p in parts if p]
    
    value = artifact
    for part in parts:
        if value is None:
            return None
        if part.isdigit():
            value = value[int(part)] if int(part) < len(value) else None
        else:
            value = value.get(part) if isinstance(value, dict) else None
    
    return value
```

### Type Filtering

```python
def filter_by_type(artifact, artifact_type):
    """Filter embedded artifacts by type."""
    matches = []
    
    def find_type(data, path='root'):
        if isinstance(data, dict):
            if data.get('artifact_type') == artifact_type:
                matches.append({'path': path, 'data': data})
            for key, value in data.items():
                find_type(value, f"{path}.{key}")
        elif isinstance(data, list):
            for i, item in enumerate(data):
                find_type(item, f"{path}[{i}]")
    
    find_type(artifact)
    return matches
```

## Advanced Queries

### Deep Content Search

```python
def deep_search(artifact, query_fn):
    """Search with custom query function."""
    results = []
    
    def traverse(data, path='root'):
        if query_fn(data, path):
            results.append({'path': path, 'data': data})
        
        if isinstance(data, dict):
            for key, value in data.items():
                traverse(value, f"{path}.{key}")
        elif isinstance(data, list):
            for i, item in enumerate(data):
                traverse(item, f"{path}[{i}]")
    
    traverse(artifact)
    return results

# Find all files larger than 1MB
large_files = deep_search(
    archive_data,
    lambda d, p: isinstance(d, dict) and d.get('size', 0) > 1048576
)
```

### Table Extraction

```python
def extract_all_tables(artifact):
    """Extract all tables from any artifact type."""
    tables = []
    
    def find_tables(data, source='unknown'):
        if isinstance(data, dict):
            if 'rows' in data and isinstance(data['rows'], list):
                tables.append({
                    'source': source,
                    'headers': data.get('headers', []),
                    'rows': data['rows']
                })
            
            if 'tables' in data:
                for i, table in enumerate(data['tables']):
                    find_tables(table, f"{source}.tables[{i}]")
            
            for key, value in data.items():
                if key not in ('rows', 'headers', 'tables'):
                    find_tables(value, f"{source}.{key}")
        
        elif isinstance(data, list):
            for i, item in enumerate(data):
                find_tables(item, f"{source}[{i}]")
    
    find_tables(artifact)
    return tables
```

### Attachment Discovery

```python
def get_all_attachments(artifact, recursive=True):
    """Get all attachments across nested artifacts."""
    attachments = []
    
    def find_attachments(data, depth=0, parent_path=''):
        if isinstance(data, dict):
            # Check for attachments array
            for key in ('attachments', 'embedded_objects', 'embedded_artifacts'):
                if key in data and isinstance(data[key], list):
                    for i, att in enumerate(data[key]):
                        att_info = {
                            'name': att.get('filename') or att.get('name'),
                            'type': att.get('content_type') or att.get('type'),
                            'size': att.get('size', 0),
                            'depth': depth,
                            'parent_path': parent_path,
                        }
                        attachments.append(att_info)
                        
                        if recursive and 'extracted_content' in att:
                            find_attachments(
                                att['extracted_content'], 
                                depth + 1,
                                f"{parent_path}/{att_info['name']}"
                            )
            
            for key, value in data.items():
                if key not in ('attachments', 'embedded_objects'):
                    find_attachments(value, depth, parent_path)
        
        elif isinstance(data, list):
            for item in data:
                find_attachments(item, depth, parent_path)
    
    find_attachments(artifact)
    return attachments
```

## Aggregation Queries

### Content Statistics

```python
def get_statistics(artifact):
    """Calculate statistics across artifact."""
    stats = {
        'total_text_length': 0,
        'total_tables': 0,
        'total_images': 0,
        'total_attachments': 0,
        'total_pages': 0,
        'file_types': {},
    }
    
    def aggregate(data):
        if isinstance(data, dict):
            # Count pages
            if 'pages' in data.get('content', {}):
                stats['total_pages'] += len(data['content']['pages'])
            
            # Count text
            for key in ('text', 'plain', 'html'):
                if key in data and isinstance(data[key], str):
                    stats['total_text_length'] += len(data[key])
            
            # Count tables
            if 'tables' in data:
                stats['total_tables'] += len(data['tables'])
            
            # Count images
            if 'images' in data:
                stats['total_images'] += len(data['images'])
            
            # Count attachments
            for key in ('attachments', 'embedded_objects'):
                if key in data:
                    stats['total_attachments'] += len(data[key])
            
            # Track file types
            if 'artifact_type' in data:
                t = data['artifact_type']
                stats['file_types'][t] = stats['file_types'].get(t, 0) + 1
            
            for value in data.values():
                aggregate(value)
        
        elif isinstance(data, list):
            for item in data:
                aggregate(item)
    
    aggregate(artifact)
    return stats
```

### Flatten Text

```python
def flatten_text(artifact, separator='\n\n'):
    """Combine all text content into single string."""
    texts = []
    
    def collect_text(data):
        if isinstance(data, dict):
            # Collect from common text fields
            for key in ('text', 'plain', 'body'):
                if key in data and isinstance(data[key], str):
                    texts.append(data[key])
            
            # Collect from paragraphs
            if 'paragraphs' in data:
                for para in data['paragraphs']:
                    if isinstance(para, dict) and 'text' in para:
                        texts.append(para['text'])
                    elif isinstance(para, str):
                        texts.append(para)
            
            # Collect from pages
            if 'pages' in data.get('content', {}):
                for page in data['content']['pages']:
                    if 'text' in page:
                        texts.append(page['text'])
            
            for value in data.values():
                collect_text(value)
        
        elif isinstance(data, list):
            for item in data:
                collect_text(item)
    
    collect_text(artifact)
    return separator.join(t for t in texts if t and t.strip())
```

## Export Utilities

### To DataFrame

```python
import pandas as pd

def artifact_to_dataframe(artifact, content_type='tables'):
    """Convert artifact content to pandas DataFrame."""
    
    if content_type == 'tables':
        tables = extract_all_tables(artifact)
        if tables:
            dfs = []
            for t in tables:
                if t['rows']:
                    df = pd.DataFrame(
                        t['rows'],
                        columns=t['headers'] if t['headers'] else None
                    )
                    df.attrs['source'] = t['source']
                    dfs.append(df)
            return dfs
        return []
    
    elif content_type == 'attachments':
        attachments = get_all_attachments(artifact)
        return pd.DataFrame(attachments)
    
    elif content_type == 'metadata':
        metadata = artifact.get('metadata', {})
        return pd.DataFrame([metadata])
```

### To Markdown

```python
def artifact_to_markdown(artifact):
    """Convert artifact to markdown representation."""
    lines = []
    
    # Title
    filename = artifact.get('source_file', 'Unknown')
    artifact_type = artifact.get('artifact_type', 'unknown')
    lines.append(f"# {filename}")
    lines.append(f"**Type:** {artifact_type}\n")
    
    # Metadata
    if 'metadata' in artifact:
        lines.append("## Metadata")
        for key, value in artifact['metadata'].items():
            if value:
                lines.append(f"- **{key}:** {value}")
        lines.append("")
    
    # Content
    text = flatten_text(artifact)
    if text:
        lines.append("## Content")
        lines.append(text[:2000] + "..." if len(text) > 2000 else text)
        lines.append("")
    
    # Tables
    tables = extract_all_tables(artifact)
    if tables:
        lines.append("## Tables")
        for i, table in enumerate(tables):
            lines.append(f"### Table {i+1} ({table['source']})")
            if table['headers']:
                lines.append("| " + " | ".join(str(h) for h in table['headers']) + " |")
                lines.append("| " + " | ".join("---" for _ in table['headers']) + " |")
            for row in table['rows'][:10]:
                lines.append("| " + " | ".join(str(c) for c in row) + " |")
            if len(table['rows']) > 10:
                lines.append(f"*... and {len(table['rows']) - 10} more rows*")
            lines.append("")
    
    # Attachments
    attachments = get_all_attachments(artifact)
    if attachments:
        lines.append("## Attachments")
        for att in attachments:
            size_kb = att['size'] / 1024
            lines.append(f"- **{att['name']}** ({att['type']}, {size_kb:.1f} KB)")
        lines.append("")
    
    return "\n".join(lines)
```

## Batch Processing

```python
def process_multiple_artifacts(filepaths, query_fn=None):
    """Process multiple artifacts with optional filtering."""
    results = []
    
    for filepath in filepaths:
        try:
            extracted = process_artifact(filepath)
            
            if query_fn:
                matches = deep_search(extracted, query_fn)
                if matches:
                    results.append({
                        'file': filepath,
                        'matches': matches
                    })
            else:
                results.append({
                    'file': filepath,
                    'data': extracted
                })
        except Exception as e:
            results.append({
                'file': filepath,
                'error': str(e)
            })
    
    return results
```
