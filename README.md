# Excel Streaming Reader

If you've used [Apache POI](http://poi.apache.org) in the past to read in Excel files, you probably noticed that it's not very memory efficient. Reading in an entire workbook will cause a severe memory usage spike, which can wreak havoc on a server. 

There are plenty of good reasons for why Apache has to read in the whole workbook, but most of them have to do with the fact that the library allows you to read and write with random addresses. If (and only if) you just want to read the contents of an Excel file in a fast and memory effecient way, you probably don't need this ability. Unfortunately, there is nothing available in the POI library for reading a streaming workbook (though there is a [streaming writer](http://poi.apache.org/spreadsheet/how-to.html#sxssf)).

Well, that's what this project is for!

# Include

This should be available from the Maven Central repository. Just add this to your POM:

```
<dependencies>
  <dependency>
    <groupId>com.thundermoose</groupId>
    <artifactId>xlsx-streamer</artifactId>
    <version>0.1</version>
  </dependency>
</dependencies>  
```

# Usage

This library is very specific in how it is meant to be used. You should initialize it like so:

```java
import com.thundermoose.xlsx.StreamingReader;

InputStream is = new FileInputStream(new File("/path/to/workbook.xlsx"));
StreamingReader reader = StreamingReader.builder()
        .rowCacheSize(100)    // number of rows to keep in memory
        .bufferSize(4096)     // buffer size to use when reading InputStream to file
        .sheetIndex(0)        // index of sheet to use (starting from 0)
        .read(is);            // InputStream or File for XLSX file
```

Once you've done this, you can then iterate through the rows and cells like so:

```java
for (Row r : reader) {
  for (Cell c : r) {
    System.out.println(c.getStringCellValue());
  }
}
```

The StreamingReader is an autoclosable resource, and it's important that you close it to free the filesystem resource it consumed. With Java 7, you can do this:

```java
try (
  InputStream is = new FileInputStream(new File("/path/to/workbook.xlsx"));
  StreamingReader reader = StreamingReader.builder()
          .rowCacheSize(100)
          .bufferSize(4096)
          .sheetIndex(0)
          .read(is);
) {
  for (Row r : reader) {
    for (Cell c : r) {
      System.out.println(c.getStringCellValue());
    }
  }
}
```

You may access cells randomly within a row, as the entire row is cached. **However**, there is no way to randomly access rows. As this is a streaming implementation, only a small number of rows are kept in memory at any given time.

# Supported Methods

Not all POI Cell and Row functions are supported. The most basic ones are (`Cell.getStringCellValue()`, `Cell.getColumnIndex()`, etc.), but don't be surprised if you get a `NotSupportedException` on the more advanced ones.

I'll try to add more support as time goes on, but some items simply can't be read in a streaming fashion. Methods that require dependent values will not have said dependencies available at the point in the stream in which they are read.

This is a brief and very generalized list of things that are not supported for reads:

* Functions
* Macros
* Styled cells (the styles are kept at the end of the ZIP file)

# Logging

This library uses SLF4j logging. This is a rare use case, but you can plug in your logging provider and get some potentially useful output. Below is an example of doing this with log4j:

**pom.xml dependencies**

```
<dependencies>
  <dependency>
    <groupId>com.thundermoose</groupId>
    <artifactId>xlsx-streamer</artifactId>
    <version>0.1</version>
  </dependency>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.6</version>
  </dependency>
  <dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
  </dependency>
</dependencies>
```

**log4j.properties**

```
log4j.rootLogger=DEBUG, A1
log4j.appender.A1=org.apache.log4j.ConsoleAppender
log4j.appender.A1.layout=org.apache.log4j.PatternLayout
log4j.appender.A1.layout.ConversionPattern=%d{ISO8601} [%c] %p: %m%n

log4j.category.com.thundermoose=DEBUG
```

# Implementation Details

This library will take a provided `InputStream` and output it to the file system. The stream is piped safely through a configurable-sized buffer to prevent large usage of memory. Once the file is created, it is then streamed into memory from the file system.

The reason for needing the stream being outputted in this manner has to do with how ZIP files work. Because the XLSX file format is basically a ZIP file, it's not possible to find all of the entries without reading the entire InputStream.

This is a problem that can't really be gotten around for POI, as it needs a complete list of ZIP entries. The default implementation of reading from an `InputStream` in POI is to read the entire stream directly into memory. This library works by reading out the stream into a temporary file. As part of the auto-close action, the temporary file is deleted.

There are two methods for initializing the
