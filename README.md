# ZipDocumentStore
A basic No-SQL document store database based off of the python stdlib ZipFile implementation.

## Issue: Can't update or delete entries without recreating the whole file.
[zipfile doesn't support removing or updating files](https://bugs.python.org/issue6818)
[zipfile doesn't support removal of items](https://bugs.python.org/issue40175)
[possible fix](https://github.com/gambl/zipextended/blob/master/zipextended/zipfileextended.py)
[pull request to add removal](https://github.com/python/cpython/pull/19358)



## The Plan

### 1. Faster Reads by skipping the need to write to disk

Have alternative to [shutil copyfileobj](https://github.com/python/cpython/blob/2f9ebe6fd06f21ea5e708d63f2596bdffcb139b2/Lib/shutil.py#L197) -- which is used within [ZipFile _extract_member](https://github.com/python/cpython/blob/2f9ebe6fd06f21ea5e708d63f2596bdffcb139b2/Lib/zipfile.py#L1650) -- write text files directly into python memory.


Downloading & reading a ZIP file in memory using Python example 

``` python
import requests
import io
import zipfile
def download_extract_zip(url):
    """
    Download a ZIP file and extract its contents in memory
    yields (filename, file-like object) pairs
    """
    response = requests.get(url)
    with zipfile.ZipFile(io.BytesIO(response.content)) as thezip:
        for zipinfo in thezip.infolist():
            with thezip.open(zipinfo) as thefile:
                yield zipinfo.filename, thefile
```



``` python

import io

output = io.StringIO()
output.write('First line.\n')
print('Second line.', file=output)

# Retrieve file contents -- this will be
# 'First line.\nSecond line.\n'
contents = output.getvalue()

# Close object and discard memory buffer --
# .getvalue() will now raise an exception.
output.close()

```

### 2. Faster Writes by keeping zip file in memory?


[StringIO](https://docs.python.org/3.8/library/io.html#io.StringIO) for writing file into memory?


``` python
# Python 2
import zipfile
import StringIO

class InMemoryZip(object):
    def __init__(self):
        # Create the in-memory file-like object
        self.in_memory_zip = StringIO.StringIO()

    def append(self, filename_in_zip, file_contents):
        '''Appends a file with name filename_in_zip and contents of 
        file_contents to the in-memory zip.'''
        # Get a handle to the in-memory zip in append mode
        zf = zipfile.ZipFile(self.in_memory_zip, "a", zipfile.ZIP_DEFLATED, False)

        # Write the file to the in-memory zip
        zf.writestr(filename_in_zip, file_contents)

        # Mark the files as having been created on Windows so that
        # Unix permissions are not inferred as 0000
        for zfile in zf.filelist:
            zfile.create_system = 0        

        return self

    def read(self):
        '''Returns a string with the contents of the in-memory zip.'''
        self.in_memory_zip.seek(0)
        return self.in_memory_zip.read()

    def writetofile(self, filename):
        '''Writes the in-memory zip to a file.'''
        f = file(filename, "w")
        f.write(self.read())
        f.close()

```
