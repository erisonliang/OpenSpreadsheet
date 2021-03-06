# OpenSpreadsheet
[![Nuget Badge](https://img.shields.io/nuget/v/OpenSpreadsheet.svg?style=for-the-badge)](https://www.nuget.org/packages/OpenSpreadsheet/)

OpenSpreadsheet is a fast and lightweight wrapper around the OpenXml spreadsheet library, employing an easy-to-use fluent interface to define relations between entities and spreadsheet rows. The library uses the Simple API for XML (SAX) method for both reading and writing.

The primary use case for OpenSpreadsheet is efficiently importing and exporting typed collections, where each row roughly corresponds to a class instance. It is not meant to offer fine-grained control of data or formatting at the cell level; if you need this level of control, check out [ClosedXml](https://github.com/ClosedXML/ClosedXML) or [EPPlus](https://github.com/JanKallman/EPPlus).


## Syntax

### Configuration

OpenSpreadsheet uses a fluent interface to map object properties to spreadsheet rows. The configuration format is modeled after the fantastic [CsvHelper](https://joshclose.github.io/CsvHelper/) library, although OpenSpreadsheet has far fewer mapping options (for now!).

**Basic Example**

Each entity to be read or written to a spreadsheet needs to have a `ClassMap` defining the relationship between the class's properties and the spreadsheet. A couple notes on the basics:
+ Classes being mapped must have either a parameterless constructor or a constructor with optional arguments.
+ Indexes are optional. When reading, OpenSpreadsheet will attempt to match the spreadsheet header with the defined mapping name, or the property name if not defined. For writing, the mapping order will be used unless the index is explicitly defined.
+ The name map is optional. When reading, the name is used to match a property to a header name if not index is defined. When writing, the name will provide the header, defaulting to the property name. 

Most configuration properties have both a read and write version, if applicable. If you need to a class to have different mappings for reading and writing operations, simply use the appropriate map method.

```c#
public class TestClassMap : ClassMap<TestClass>
{
    public TestClassMap()
    {
        Map(x => x.Surname).Index(1).Name("Employee Last Name");
        Map(x => x.GivenName).Index(2).Name("Employee First Name");
        Map(x => x.Id).Index(3).Name("Employee Id");
        Map(x => x.Address).Index(4).IgnoreWrite(true);
        Map(x => x.SSN).IndexRead(10).IndexWrite(5).CustomNumberFormat("000-00-0000");
        Map(x => x.Amount).Index(6).Style(new ColumnStyle() { NumberFormat = OpenXmlNumberingFormat.Accounting });
    }
}
````
<br />

**Constants and Defaults**
If you need to supply a constant value to a property during reading or you'd like to write a constant value (with or without an associated property), use the `Constant` map.

If you need to supply a fallback value for null values, use the `Default` map.

```c#
public class TestClassMap : ClassMap<TestClass>
{
    public TestClassMap()
    {
        Map().Index(1).Name("Date").ConstantWrite(DateTime.Today.ToString());
        Map(x => x.Id).Index(2).Name("Employee Id").Default(0);
    }
}
````
<br />

**Column Styles**

In order to customize the appearance of a style, simply create a new `ColumnStyle` instance and map it to the property using the `Style` method. If an explicit `ColumnStyle` is not specified, a default instance will be used.

```c#
public class TestClassMap : ClassMap<TestClass>
{
    public TestClassMap()
    {
        var columnStyle = new ColumnStyle()
        {
            BackgroundColor = Color.Aquamarine,
            BackgroundPatternType = DocumentFormat.OpenXml.Spreadsheet.PatternValues.Solid,
            BorderColor = Color.Red,
            BorderPlacement = BorderPlacement.Outside,
            BorderStyle = DocumentFormat.OpenXml.Spreadsheet.BorderStyleValues.Thin,
            Font = new Font("Arial", 14, FontStyle.Italic),
            ForegroundColor = Color.White,
            HoizontalAlignment = DocumentFormat.OpenXml.Spreadsheet.HorizontalAlignmentValues.Center,
            NumberFormat = OpenXmlNumberingFormat.Currency,
            VerticalAlignment = DocumentFormat.OpenXml.Spreadsheet.VerticalAlignmentValues.Center
        };

        Map(x => x.Amount).Index(1).Style(columnStyle);
        Map(x => x.SSN).Index(2).Style(new ColumnStyle() { CustomNumberFormat = "000-00-0000" });
    }
}
```
<br />

**Data Conversions**

Sometimes you need to provide some additional logic to accurately map between your spreadsheet and your class instance. In these cases, use the `ReadUsing` and `WriteUsing` to provide a delegate to be used for the mapping operation. The `ReadUsing` delegate takes a `ReaderRow` for its input parameter, which allows you to retrieve data from any cell within the row by using its header name or column index. The `WriteUsing` delegate takes the class instance as its input parameter. 

In the example blow, the `ClassMap` contains a map for the boolean property `IsExpired`. During reading, the value of `IsExpired` is determined by comparing the current date a date contained in a cell with a header named "ExpirationDate". When writing, the value written to the cell is 'T' or 'F' rather than the default `IsExpired.ToString()` of `True` or `False`.

```c#
public class TestClassMap : ClassMap<TestClass>
{
    public TestClassMap()
    {
        Map(x => x.IsExpired)
            .ReadUsing(row =>
            {
                var expirationTextValue = row.GetCellValue("ExpirationDate");
                var expirationDate = DateTime.Parse(expirationTextValue);
                return expirationDate < DateTime.Now;
            })
            .WriteUsing(x => x.IsExpired ? "T" : "F");
    }
}
````


### Writing

To write data to a new worksheet, simply call the WriteWorksheet method from your Spreadsheet, providing the type of object to be written and its associtiated map. If you want more fine-grained control over the write operation, have your Spreadsheet create a new WorksheetWriter.

```c#
using (var spreadsheet = new Spreadsheet(filepath))
{
    // write all records from the Spreadsheet (uses a WorksheetWriter behind the scenes)
    spreadsheet.WriteWorksheet<TestClass, TestClassMap>("Sheet2", records);

    // write all records using an explicit WorksheetWriter
    using (var writer = spreadsheet.CreateWorksheetWriter<TestClass, TestClassMap>("Sheet3"))
    {
        writer.WriteRecords(records);
    }

    // write individual records from the WorksheetWriter
    using (var writer = spreadsheet.CreateWorksheetWriter<TestClass, TestClassMap>("Sheet1", 0))
    {
        writer.WriteHeader();
        writer.SkipRows(3);
        writer.WriteRecord(new TestClass() { TestData = "first row" });
        writer.WriteRecord(new TestClass() { TestData = "second row" });        
        writer.WriteRecord(new TestClass() { TestData = "third row" });
        writer.SkipRow();
        writer.WriteRecord(new TestClass() { TestData = "fourth row" });
    }
}
```

To apply general worksheet styles, create a new WorksheetStyle instance and pass it as an argument to your write operations. Otherwise, a default WorksheetStyle instance will be used.

```c#
var worksheetStyle = new WorksheetStyle()
{
    HeaderBackgroundColor = Color.Chartreuse,
    HeaderBackgroundPatternType = DocumentFormat.OpenXml.Spreadsheet.PatternValues.Solid,
    HeaderFont = new Font("Comic Sans", 16, FontStyle.Strikeout),
    HeaderForegroundColor = Color.DarkBlue,
    HeaderHoizontalAlignment = DocumentFormat.OpenXml.Spreadsheet.HorizontalAlignmentValues.Center,
    HeaderRowIndex = 2,
    HeaderVerticalAlignment = DocumentFormat.OpenXml.Spreadsheet.VerticalAlignmentValues.Center,
    MaxColumnWidth = 30,
    MinColumnWidth = 10,
    ShouldAutoFilter = true,
    ShouldAutoFitColumns = true,
    ShouldFreezeTopRow = true,
    ShouldWriteHeaderRow = true,
};

using (var spreadsheet = new Spreadsheet(filepath))
{
    spreadsheet.WriteWorksheet<TestClass, TestClassMap>("Sheet1", records, worksheetStyle);
}
```

### Reading

To read data from an existing worksheet, simply call the ReadWorksheet method from your Spreadsheet, providing the type of object to be written and its associtiated map. If you want more fine-grained control over the read operation, have your Spreadsheet create a new WorksheetReader.

```c#
using (var spreadsheet = new Spreadsheet(filepath))
{
    // read all records from the Spreadsheet (uses a WorksheetReader behind the scenes)
    var recordsSheet1 = spreadsheet.ReadWorksheet<TestClass, TestClassMap>("Sheet1");

    // read all records using an explicit WorksheetReader
    using (var reader = spreadsheet.CreateWorksheetReader<TestClass, TestClassMap>("Sheet2"))
    {
        var recordsSheet2 = reader.ReadRows();
    }

    // read individual records
    using (var reader = spreadsheet.CreateWorksheetReader<TestClass, TestClassMap>("Sheet3"))
    {
        var firstRow = reader.ReadRow();
        var secondRow = reader.ReadRow();
        reader.SkipRow();
        var fourthRow = reader.ReadRow();
    }
}
```

## Performance

**Reading**

OpenSpreadsheet is significantly faster and better on memory than ClosedXml, but is generally slower than EPPlus. For reading, all three libraries are pretty performant.

| Library | Records | Fields | Runtime | Memory Used |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| [ClosedXml](https://github.com/ClosedXML/ClosedXML) | 50,000 | 3 | 971.2 ms | 211.46 MB
| [EPPlus](https://github.com/JanKallman/EPPlus) | 50,000 | 3 | 394.9 ms | 139.05 MB
| [OpenSpreadsheet](https://github.com/FolkCoder/OpenSpreadsheet) | 50,000 | 3 | 745.7 ms | 121.14 MB
| [ClosedXml](https://github.com/ClosedXML/ClosedXML) | 100,000 | 3 | 1,932.7 ms | 423.67 MB
| [EPPlus](https://github.com/JanKallman/EPPlus) | 100,000 | 3 | 807.2 ms | 277.15 MB
| [OpenSpreadsheet](https://github.com/FolkCoder/OpenSpreadsheet) | 100,000 | 3 | 1,502.6 ms | 241.69 MB
| [ClosedXml](https://github.com/ClosedXML/ClosedXML) | 250,000 | 3 | 4,747.9 ms | 1044.93 MB
| [EPPlus](https://github.com/JanKallman/EPPlus) | 250,000 | 3 | 2,003.8 ms| 686.58 MB
| [OpenSpreadsheet](https://github.com/FolkCoder/OpenSpreadsheet) | 250,000 | 3 | 3,694.0 ms | 602.89 MB
| [ClosedXml](https://github.com/ClosedXML/ClosedXML) | 500,000 | 3 | 113.359 ms | 2094.14 MB
| [EPPlus](https://github.com/JanKallman/EPPlus) | 500,000 | 3 | 75.751 ms| 1372.95 MB
| [OpenSpreadsheet](https://github.com/FolkCoder/OpenSpreadsheet) | 500,000 | 3 | 79.665 ms | 1205.57 MB


**Writing**

OpenSpreadsheet is significantly faster and memory-friendly than ClosedXml, and slightly more so than EPPlus.

| Library | Records | Fields | Runtime | Memory Used |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| [ClosedXml](https://github.com/ClosedXML/ClosedXML) | 50,000 | 30 | 12.013 s | 2459.94 MB
| [EPPlus](https://github.com/JanKallman/EPPlus) | 50,000 | 30 | 3.351 s | 1039.68 MB
| [OpenSpreadsheet](https://github.com/FolkCoder/OpenSpreadsheet) | 50,000 | 30 | 2.401 s | 1006.11 MB
| [ClosedXml](https://github.com/ClosedXML/ClosedXML) | 100,000 | 30 | 23.908 s | 4928.38 MB
| [EPPlus](https://github.com/JanKallman/EPPlus) | 100,000 | 30 | 6.658 s | 2053.81 MB
| [OpenSpreadsheet](https://github.com/FolkCoder/OpenSpreadsheet) | 100,000 | 30 | 4.865 s | 2005.31 MB
| [ClosedXml](https://github.com/ClosedXML/ClosedXML) | 250,000 | 30 | 59.999 s | 12027.75 MB
| [EPPlus](https://github.com/JanKallman/EPPlus) | 250,000 | 30 | 16.526 s | 5041.11 MB
| [OpenSpreadsheet](https://github.com/FolkCoder/OpenSpreadsheet) | 250,000 | 30 | 11.997 s | 4815.44 MB


## Future Plans
+ Automatic class mapping
+ Support for dynamic and anonymous types
+ Better handling of duplicate header names for reading
+ Greatly improve accuracy and coverage of automated tests and ClassMap validations
+ Allow default worksheet style for entire spreadsheet
+ Provide override for ReadWorksheet to accept tab position index as well as name