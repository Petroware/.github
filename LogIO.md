# Background and Examples <!-- omit from toc -->
The coding examples below are shown using the Java API. The .NET API is equivalent.

- [DLIS](#dlis)
- [LIS](#lis)
- [LAS](#las)
- [BIT](#bit)
- [XTF](#xtf)
- [SPWLA](#spwla)
- [ASC](#asc)
- [WITSML](#witsml)
- [JSON Well Log Format](#json-well-log-format)


## DLIS 

*Digital Log Interchange Standard* (DLIS) is a digital well log format introduced as Recommended Practice 66 (RP 66) by the [American Petroleum Institute](https://www.api.org/) in 1991. RP 66 exists in two versions, [V1](http://w3.energistics.org/RP66/V1/Toc/main.html) and [V2](http://w3.energistics.org/RP66/V2/Toc/main.html). V2 is not commonly used. DLIS is a binary (big-endian) data format.

DLIS is an *old* format in all possible ways. Available DLIS software is limited and to a large degree *orphaned* technology. The existence of programming tools for DLIS are absent. There is no easily accessible DLIS format description available, dialects exists, and DLIS is in general very hard to digest.

But still, DLIS is the main storage and communication medium for wellbore logging information. Since logging technology has evolved substantially since the introduction of the format, the amount of information contained may be vast; multi-GB volumes are commonplace.

Using Log I/O, reading and writing DLIS files becomes trivial. Note that a physical DLIS file consist of a file header and one or more logical DLIS files, each containing a number of *sets* (meta-data), one or more *frames* containing curve data, and possibly a set of *encrypted* records. A frame contains an index curve and a set of measurement curves. Each curve may be single- or multi-dimensional.

A complete example program for reading a DLIS file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.dlis.DlisCurve;
import no.petroware.logio.dlis.DlisFile;
import no.petroware.logio.dlis.DlisFileReader;
import no.petroware.logio.dlis.DlisFrame;
import no.petroware.logio.dlis.DlisSet;
import no.petroware.logio.dlis.DlisUtil;

/**
 * Class for demonstrating the Log I/O DLIS read API.
 */
public final class DlisReadTest
{
  public static void main(String[] arguments)
    throws IOException, InterruptedException
  {
    File file = new File("path/to/file.DLIS");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the DLIS file including bulk data. Report the performance.
    //
    DlisFileReader reader = new DlisFileReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    List<DlisFile> dlisFiles = reader.read(true, false, null);
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    //
    // Write the DLIS file header
    //
    System.out.println();
    System.out.println("-- File header:");
    System.out.println(dlisFiles.get(0).getHeader());

    //
    // Report number of subfiles. Loop over each of them.
    //
    System.out.println();
    System.out.println("The file contains " + dlisFiles.size() + " subfile(s):");
    for (DlisFile dlisFile : dlisFiles) {
      //
      // Report subfile ID
      //
      System.out.println();
      System.out.println("File ID: " + dlisFile.getFileId());
      System.out.println();

      //
      // Report the meta-data we can conveniently extract through DlisUtil
      //
      System.out.println("-- Meta-data as reported by DlisUtil:");
      System.out.println("Well name.........: " + DlisUtil.getWellName(dlisFile));
      System.out.println("Run number........: " + DlisUtil.getRunNumber(dlisFile));
      System.out.println("Field name........: " + DlisUtil.getFieldName(dlisFile));
      System.out.println("Producer name.....: " + DlisUtil.getProducerName(dlisFile));
      System.out.println("Company...........: " + DlisUtil.getCompany(dlisFile));
      System.out.println("Country...........: " + DlisUtil.getCountry(dlisFile));
      System.out.println("Start date........: " + DlisUtil.getStartDate(dlisFile));
      System.out.println("Bit size..........: " + DlisUtil.getBitSize(dlisFile));
      System.out.println("MD................: " + DlisUtil.getMd(dlisFile));
      System.out.println("Index type........: " + DlisUtil.getIndexType(dlisFile));
      System.out.println("Header identifier.: " + DlisUtil.getHeaderIdentifier(dlisFile));
      double[] interval = DlisUtil.getInterval(dlisFile);
      System.out.println("Interval..........: " + interval[0] + " - " + interval[1]);

      //
      // Report all the meta-data
      //
      System.out.println();
      System.out.println("-- Meta-data details:");
      System.out.println();
      System.out.println("The subfile contains " + dlisFile.getSets().size() + " set(s):");
      System.out.println();
      for (DlisSet set : dlisFile.getSets()) {
        System.out.println("Set type: " + set.getType() + " (Name = " + set.getName() + ")");
        // This will write all the set content as a matrix
        System.out.println(set);
      }

      //
      // Encrypted records
      //
      System.out.println();
      System.out.println("-- The subfile contains " + dlisFile.getEncryptedRecords().size() + " encrypted record(s).");

      //
      // Report number of frames in this sub file. Loop over each of them.
      //
      System.out.println();
      System.out.println("-- Curve information:");
      System.out.println();
      System.out.println("The subfile contains " + dlisFile.getFrames().size() + " frame(s):");
      for (DlisFrame frame : dlisFile.getFrames()) {
        int nCurves = frame.getCurves().size();
        int nValues = nCurves > 0 ? frame.getCurves().get(0).getNValues() : 0;

        //
        // Report frame ID and #curves and #values
        //
        System.out.println();
        System.out.println("Frame " + frame.getName() +
                          " (" + nCurves + " curves of " + nValues + " values each):");
        System.out.println();

        //
        // For each curve, report name, dimension, unit, type and description, ...
        //
        for (DlisCurve curve : frame.getCurves()) {
          System.out.println("  " + curve.getName() +
                            "[" + curve.getNDimensions() + "], " +
                            curve.getUnit() + ", " +
                            curve.getValueType() +
                            "(" + curve.getRepresentationCode() + "), " +
                            curve.getDescription());

          //
          // ... and the first few values from the first couple of dimensions:
          //
          for (int dimension = 0; dimension < curve.getNDimensions(); dimension++) {
            System.out.println("    ");
            for (int index = 0; index < curve.getNValues(); index++) {
              System.out.println(curve.getValue(dimension, index) + "; ");
              if (index == 2)
                break; // Write a few values only
            }
            System.out.println("...");

            if (dimension == 2) { // Write a few dimensions only
              System.out.println("    :");
              break;
            }
          }
        }
      }
    }
  }
}
```

Creating DLIS files from scratch includes populating **DlisFile** instances with *sets*, *frames* and *curves* and writing them to disk using the **DlisFileWriter** class.

Note that the write API is by design *low-level* and there are many possible ways for a client to create an *inconsistent* DLIS file instance. The design is chosen for simplicity and flexibility. Each client is likely to implement one DLIS writer component anyway, and following the simple procedure below will ensure consistent DLIS instances:

```java
// Create an empty DLIS file instance
DlisFile dlisFile = new DlisFile();

//
// Add the mandatory FILE-HEADER set
//

// Create the set
List<DlisComponent> fileHeaderAttributes = new ArrayList<>();
fileHeaderAttributes.add(DlisComponent.newAttributeComponent("SEQUENCE-NUMBER", DlisRepresentationCode.ASCII, null));
fileHeaderAttributes.add(DlisComponent.newAttributeComponent("ID", DlisRepresentationCode.ASCII, null));
DlisSet fileHeaderSet = new DlisSet("FILE-HEADER", null, fileHeaderAttributes);

// Create an object and add to the set
DlisComponent fileHeaderObject = DlisComponent.newObjectComponent(0, "0");
List<DlisComponent> fileHeaderValues = new ArrayList<>();
fileHeaderValues.add(DlisComponent.newValueComponent("1", DlisRepresentationCode.ASCII, null)); // SEQUENCE-NUMBER
fileHeaderValues.add(DlisComponent.newValueComponent("logSetName", DlisRepresentationCode.ASCII, null)); // ID
fileHeaderSet.addObject(fileHeaderObject, fileHeaderValues);

// Add set to file
dlisFile.addSet(fileHeaderSet);

//
// Add the mandatory ORIGIN set
//

// Create set
List<DlisComponent> originAttributes = new ArrayList<>();
originAttributes.add(DlisComponent.newAttributeComponent("FILE-ID", DlisRepresentationCode.ASCII, null));
originAttributes.add(DlisComponent.newAttributeComponent("WELL-NAME", DlisRepresentationCode.ASCII, null));
originAttributes.add(DlisComponent.newAttributeComponent("FIELD-NAME", DlisRepresentationCode.ASCII, null));
originAttributes.add(DlisComponent.newAttributeComponent("COMPANY", DlisRepresentationCode.ASCII, null));
DlisSet originSet = new DlisSet("ORIGIN", null, originAttributes);

// Create an object and add to set
DlisComponent originObject = DlisComponent.newObjectComponent(0, "DLIS_DEFINING_ORIGIN");
List<DlisComponent> originValues = new ArrayList<>();
originValues.add(DlisComponent.newValueComponent("fileId", DlisRepresentationCode.ASCII, null)); // FILE-ID
originValues.add(DlisComponent.newValueComponent("25/1-A-4", DlisRepresentationCode.ASCII, null)); // WELL-NAME
originValues.add(DlisComponent.newValueComponent("Frigg", DlisRepresentationCode.ASCII, null)); // FIELD-NAME
originValues.add(DlisComponent.newValueComponent("Mobil", DlisRepresentationCode.ASCII, null)); // COMPANY
originSet.addObject(originObject, originValues);

// Add set to file
dlisFile.addSet(originSet);

//
// Add the mandatory CHANNEL set describing all curves
//

// Create set
List<DlisComponent> channelAttributes = new ArrayList<DlisComponent>();
channelAttributes.add(DlisComponent.newAttributeComponent("LONG-NAME", DlisRepresentationCode.ASCII,  null));
channelAttributes.add(DlisComponent.newAttributeComponent("PROPERTIES", DlisRepresentationCode.IDENT,  null));
channelAttributes.add(DlisComponent.newAttributeComponent("REPRESENTATION-CODE", DlisRepresentationCode.USHORT, null));
channelAttributes.add(DlisComponent.newAttributeComponent("UNITS", DlisRepresentationCode.UNITS,  null));
channelAttributes.add(DlisComponent.newAttributeComponent("DIMENSION", DlisRepresentationCode.UVARI,  null));
DlisSet channelSet = new DlisSet("CHANNEL", null, channelAttributes);

// Create objects for each curve
DlisComponent channelObject1 = DlisComponent.newObjectComponent(0, "MD");
List<DlisComponent> channelValues1 = new ArrayList<>();
channelValues1.add(DlisComponent.newValueComponent("Measured depth", DlisRepresentationCode.ASCII, null)); // LONG-NAME
channelValues1.add(DlisComponent.newValueComponent("BASIC",  DlisRepresentationCode.IDENT, null)); // PROPERTIES
channelValues1.add(DlisComponent.newValueComponent(DlisRepresentationCode.FDOUBL, DlisRepresentationCode.USHORT, null)); // REPRESENTATION-CODE
channelValues1.add(DlisComponent.newValueComponent("m", DlisRepresentationCode.UNITS, null)); // UNITS
channelValues1.add(DlisComponent.newValueComponent(1, DlisRepresentationCode.UVARI, null)); // DIMENSION
channelSet.addObject(channelObject1, channelValues1);

DlisComponent channelObject2 = DlisComponent.newObjectComponent(0, "GR");
List<DlisComponent> channelValues2 = new ArrayList<>();
channelValues2.add(DlisComponent.newValueComponent("Gamma ray", DlisRepresentationCode.ASCII, null)); // LONG-NAME
channelValues2.add(DlisComponent.newValueComponent("BASIC",  DlisRepresentationCode.IDENT, null)); // PROPERTIES
channelValues2.add(DlisComponent.newValueComponent(DlisRepresentationCode.FDOUBL, DlisRepresentationCode.USHORT, null)); // REPRESENTATION-CODE
channelValues2.add(DlisComponent.newValueComponent("gAPI", DlisRepresentationCode.UNITS, null)); // UNITS
channelValues2.add(DlisComponent.newValueComponent(1, DlisRepresentationCode.UVARI, null)); // DIMENSION
channelSet.addObject(channelObject2, channelValues2);

// Add set to file
dlisFile.addSet(channelSet);

//
// Add any other sets containing meta-data
//
:
:

//
// Add the mandatory FRAME set describing all the containing frames
// and what curves they contains
//

// Create set
List<DlisComponent> frameAttributes = new ArrayList<>();
frameAttributes.add(DlisComponent.newAttributeComponent("DESCRIPTION", DlisRepresentationCode.ASCII, null));
frameAttributes.add(DlisComponent.newAttributeComponent("CHANNELS", DlisRepresentationCode.OBNAME, null));
frameAttributes.add(DlisComponent.newAttributeComponent("INDEX-TYPE", DlisRepresentationCode.IDENT, null));
frameAttributes.add(DlisComponent.newAttributeComponent("DIRECTION", DlisRepresentationCode.IDENT, null));
frameAttributes.add(DlisComponent.newAttributeComponent("SPACING", DlisRepresentationCode.FDOUBL, "m"));
frameAttributes.add(DlisComponent.newAttributeComponent("INDEX-MIN", DlisRepresentationCode.FDOUBL, "m"));
frameAttributes.add(DlisComponent.newAttributeComponent("INDEX-MAX", DlisRepresentationCode.FDOUBL, "m"));
DlisSet frameSet = new DlisSet("FRAME", null, frameAttributes);

// Create a frame object
DlisComponent frameObject = DlisComponent.newObjectComponent(0, "s10");
List<DlisComponent> frameValues = new ArrayList<>();
frameValues.add(DlisComponent.newValueComponent("", DlisRepresentationCode.ASCII, null)); // DESCRIPTION
frameValues.add(DlisComponent.newValueComponent(new ArrayList<Object>(Arrays.asList("MD", "GR")), DlisRepresentationCode.OBNAME, null)); // CHANNELS
frameValues.add(DlisComponent.newValueComponent("BOREHOLE-DEPTH", DlisRepresentationCode.IDENT, null)); // INDEX-TYPE
frameValues.add(DlisComponent.newValueComponent("INCREASING", DlisRepresentationCode.IDENT, null)); // DIRECTION
frameValues.add(DlisComponent.newValueComponent(0.5, DlisRepresentationCode.FDOUBL, "m")); // SPACING
frameValues.add(DlisComponent.newValueComponent(1450.0, DlisRepresentationCode.FDOUBL, "m")); // INDEX-MIN
frameValues.add(DlisComponent.newValueComponent(1451.0, DlisRepresentationCode.FDOUBL, "m")); // INDEX-MAX
frameSet.addObject(frameObject, frameValues);

// Add set to file
dlisFile.addSet(frameSet);

//
// Create one frame per frame objects in the FRAME set and populate with curves and data.
// The curves must already have been defined in the CHANNEL set above.
//
DlisFrame frame = new DlisFrame(frameSet, frameObject);

DlisCurve curve1 = new DlisCurve("MD", "m", "Measured depth", DlisRepresentationCode.FDOUBL, 1);
curve1.addValue(1450.0);
curve1.addValue(1450.5);
curve1.addValue(1451.0);
frame.addCurve(curve1);

DlisCurve curve2 = new DlisCurve("GR", "API", "Gamma ray", DlisRepresentationCode.FDOUBL, 1);
curve2.addValue(46.89);
curve2.addValue(30.65);
curve2.addValue(43.02);
frame.addCurve(curve2);

// Add frame to file
dlisFile.addFrame(frame);

//
// Finally write the DLIS file instance to disk
//
DlisFileWriter fileWriter = new DlisFileWriter(new File("/path/to/file.DLIS");
fileWriter.write(dlisFile);
```

## LIS
*Log Interchange Standard* (LIS) is the predecessor to DLIS and was developed by [Schlumberger](https://www.slb.com/) in the late 1970s.

Like DLIS, [LIS](http://w3.energistics.org/LIS/lis-79.pdf) is a binary (big-endian) format. It it is based on the VAX binary information standard and has an even more awkward syntax than DLIS.

But still, LIS files are still being produced and immense volumes of historical logging information exists in this format. Log I/O is a convenient platform for being able to manage and maintain this information.

A physical LIS file consists of one or more logical LIS files, each containing a set of *records* (meta-data) of different types as well as an index curve and a set of measurement curves. Each curve may have one or several samples per depth measure, and in addition the curves may be single- or multi-dimensional.

A complete example program for reading a LIS file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.lis.LisCurve;
import no.petroware.logio.lis.LisFile;
import no.petroware.logio.lis.LisFileReader;
import no.petroware.logio.lis.LisRecord;
import no.petroware.logio.lis.LisUtil;

/**
 * Class for demonstrating the Log I/O LIS read API.
 */
public final class LisReadTest
{
  public static void main(String[] arguments)
    throws IOException, InterruptedException
  {
    File file = new File("path/to/file.LIS");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the LIS file including bulk data. Report the performance.
    //
    LisFileReader reader = new LisFileReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    List<LisFile> lisFiles = reader.read(true, false, null);
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    // Loop over all logical files
    for (LisFile lisFile : lisFiles) {
      System.out.println("Sub file name: " + lisFile.getName());

      //
      // Report the meta-data we can conveniently extract through LisUtil
      //
      System.out.println("-- Meta-data as reported by LisUtil:");
      System.out.println("Well name........: " + LisUtil.getWellName(lisFile));
      System.out.println("Field name.......: " + LisUtil.getFieldName(lisFile));
      System.out.println("Run number.......: " + LisUtil.getRunNumber(lisFile));
      System.out.println("Company..........: " + LisUtil.getCompany(lisFile));
      System.out.println("Service company..: " + LisUtil.getServiceCompany(lisFile));
      System.out.println("Country..........: " + LisUtil.getCountry(lisFile));
      System.out.println("Rig name.........: " + LisUtil.getRigName(lisFile));
      System.out.println("Date.............: " + LisUtil.getDate(lisFile));
      System.out.println("Bit size.........: " + LisUtil.getBitSize(lisFile));

      //
      // Report all the meta-data
      //
      System.out.println();
      System.out.println("-- Meta-data details:");
      System.out.println();
      System.out.println("The subfile contains " + lisFile.getRecords().size() + " record(s):");
      System.out.println();
      for (LisRecord record : lisFile.getRecords())
        System.out.println(record);

      //
      // Report #curves and #values
      //
      int nCurves = lisFile.getCurves().size();
      int nValues = nCurves > 0 ? lisFile.getCurves().get(0).getNValues() : 0;

      System.out.println();
      System.out.println("The file contains " + nCurves + " curves of " + nValues + " values each.");
      System.out.println();

      //
      // For each curve, report curve info, ...
      //
      for (LisCurve curve : lisFile.getCurves()) {
        System.out.println("  " + curve.getName() +
                           "[" + curve.getNDimensions() + "], " +
                           curve.getUnit() + ", " +
                           curve.getValueType() +
                           "(" + curve.getRepresentationCode() + "), " +
                           curve.getDescription());

        //
        // ... and the first few values from the first couple of dimensions:
        //
        for (int sample = 0; sample < curve.getNSamples(), sample++) {
          for (int dimension = 0; dimension < curve.getNDimensions(); dimension++) {
            System.out.print("    ");
            for (int index = 0; index < curve.getNValues(); index++) {
              System.out.print(curve.getValue(sample, dimension, index) + "; ");
              if (index == 10)
                break; // Write a few values only
            }
            System.out.println("...");

            if (dimension == 10) { // Write a few dimensions only
              System.out.println("    :");
              break;
            }
          }
        }
      }
    }
  }
}
```

Creating LIS files from scratch includes populating **LisFile** instances with *records* and *curves* and writing them to a disk using the **LisFileWriter** class.

Note that the write API is by design *low-level* and there are many possible ways for a client to create an *inconsistent* LIS file instance. The design is chosen for simplicity and flexibility. Each client is likely to implement one LIS writer module anyway, and following the simple procedure below will ensure consistent LIS instances:

```java
/
// Create an empty LisFile instance
//
LisFile lisFile = new LisFile();

//
// Add a reel header record
//
LisRecord reelHeaderRecord = new LisReelHeaderRecord("Serv", new Date(), "Orig", "Reel", 1, null, "Created by ");
lisFile.addRecord(reelHeaderRecord);

//
// Add a tape header record
//
LisRecord tapeHeaderRecord = new LisTapeHeaderRecord("Serv", new Date(), "Orig", "Reel", 1, null, "Created by ");
lisFile.addRecord(tapeHeaderRecord);

//
// Add a file header record
//
LisRecord fileHeaderRecord = new LisFileHeaderRecord("", "ServID", "", new Date(), 99999, "LI", null);
lisFile.addRecord(fileHeaderRecord);

//
// Add a well site data record
//
LisWellSiteDataRecord wellSiteDataRecord = new LisWellSiteDataRecord("Well site data", new ArrayList<String>() {"Name", "Value", "Unit", "Description"});
wellSiteDataRecord.addRow(new List<String>() {"WELL", "25/1-A-4", null, "Well name"});
wellSiteDataRecord.addRow(new List<String>() {"FIELD", "Frigg", null, "Field name"});
wellSiteDataRecord.addRow(new List<String>() {"MD", "2450.5", "m", "Measured depth"});
:
lisFile.addRecord(wellSiteDataRecord);

//
// Add the mandatory data format specification
//
List<LisDataFormatSpecificationRecord.Entry> entries = new ArrayList<>();
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.DATA_RECORD_TYPE, 0));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.DATUM_SPEC_BLOCK_SUB_TYPE, 1));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.UP_DOWN_FLAG, 255));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.FRAME_SPACING, 0.5));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.UNITS_OF_FRAME_SPACING, "m"));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.ABSENT_VALUE, -999.25));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.DEPTH_RECORDING_MODE, 1));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.UNITS_OF_DEPTH, "m"));
entries.add(new LisDataFormatSpecificationRecord.Entry(LisDataFormatSpecificationRecord.Entry.Type.TERMINATOR, 0));

List<LisDataFormatSpecificationRecord.Datum> datums = new ArrayList<>();
datums.add(new LisDataFormatSpecificationRecord.Datum("GR", "gAPI", 1, LisRepresentationCode.FSINGL, 4);
datums.add(new LisDataFormatSpecificationRecord.Datum("NEU", null, 1, LisRepresentationCode.FSINGL, 4);

lisFile.addRecord(new LisDataFormatSpecificationRecord(entries, datums);

//
// Add curve data
//
LisCurve depthCurve = lisFile.getCurves().get(0);
depthCurve.addValue(0, 0, 4350.00);
depthCurve.addValue(0, 0, 4350.50);
depthCurve.addValue(0, 0, 4351.00);
depthCurve.addValue(0, 0, 4351.50);
depthCurve.addValue(0, 0, 4352.00);

LisCurve gammaRayCurve = lisFile.getCurves().get(1);
gammaRayCurve.addValue(0, 0, 12.3);
gammaRayCurve.addValue(0, 0, 8.43);
gammaRayCurve.addValue(0, 0, null);  // No-value
gammaRayCurve.addValue(0, 0, 4.1);
gammaRayCurve.addValue(0, 0, 7.29);

LisCurve neutronCurve = lisFile.getCurves().get(2);
neutronCurve.addValue(0, 0, 1.1);
neutronCurve.addValue(0, 0, 2.02);
neutronCurve.addValue(0, 0, 1.13);
neutronCurve.addValue(0, 0, 0.89);
neutronCurve.addValue(0, 0, 3.11);

//
// Add file trailer
//
LisRecord fileTrailerRecord = new LisFileTrailerRecord("", "ServID", "", new Date(), 99999, "LI", null);
lisFile.addRecord(fileTrailerRecord);

//
// Add tape trailer
//
LisRecord tapeTrailerRcord = new LisTapeTrailerRecord("Serv", new Date(), "Orig", "Tape", 1, null, "Created by ");
lisFile.addRecord(tapeTrailerRcord);

//
// Add reel trailer
//
LisRecord reelTrailerRecord = new LisReelTrailerRecord("Serv", new Date(), "Orig", "Reel", 1, null, "Created by ");
lisFile.addRecord(reelTrailerRecord);

//
// Write to file
//
LisFileWriter writer = new LisFileWriter(new File("path/to/file.LIS");
writer.write(lisFile);
```

## LAS
*The Log ASCII Standard* (LAS) was created by the [Canadian Well Logging Society](https://www.cwls.org/) (CWLS) in the late 1980s.

LAS popuplarity is due to its simple syntax and the fact that it contains human readable ASCII text. Consequently it is easier to work with than the black-box DLIS and LIS files. The ASCII format has a price however; It requires a lot more storage space than equivalent DLIS files, so LAS is not useful for very large log volumes. Also, the easily accessible text format combined with an ambiguous format description has caused many LAS dialects and semantic interpretations over the years.

Three different versions of LAS exists, [1.2](https://www.cwls.org/wp-content/uploads/2014/09/LAS12_Standards.txt), [2.0](https://www.cwls.org/wp-content/uploads/2014/09/LAS_20_Update_Jan2014.pdf) and [3.0](https://www.cwls.org/wp-content/uploads/2014/09/LAS_3_File_Structure.pdf). Current version is 3.0, published in 2000, but version 2.0 is by far the most common. Log I/O supports all LAS versions, as well as converting between them. Note that LAS does not support multi-frame nor multi-dimensional curves like DLIS. LAS 3.0 supports multiple subfiles per physical disk file.

A complete example program for reading a LAS file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.las.LasCurve;
import no.petroware.logio.las.LasFile;
import no.petroware.logio.las.LasFileReader;
import no.petroware.logio.las.LasUtil;

/**
 * Class for demonstrating the Log I/O LAS read API.
 */
public final class LasReadTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    File file = new File("path/to/file.LAS");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the LAS file including bulk data. Report the performance.
    //
    LasFileReader reader = new LasFileReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    List<LasFile> lasFiles = reader.read(true);
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    //
    // Report number of subfiles (LAS 3.0). Loop over each of them.
    //
    System.out.println();
    System.out.println("The file contains " + lasFiles.size() + " subfile(s):");
    for (LasFile lasFile : lasFiles) {
      System.out.println("File name: " + lasFile.getName());

      //
      // Report the meta-data we can conveniently extract through LasUtil
      //
      System.out.println("-- Meta-data as reported by LasUtil:");
      System.out.println("Well name.........: " + LasUtil.getWellName(lasFile));
      System.out.println("Field name........: " + LasUtil.getFieldName(lasFile));
      System.out.println("Rig name..........: " + LasUtil.getRigName(lasFile));
      System.out.println("Company...........: " + LasUtil.getCompany(lasFile));
      System.out.println("Service company...: " + LasUtil.getServiceCompany(lasFile));
      System.out.println("Country...........: " + LasUtil.getCountry(lasFile));
      System.out.println("Run number........: " + LasUtil.getRunNumber(lasFile));
      System.out.println("Bit size..........: " + LasUtil.getBitSize(lasFile));
      System.out.println("Date..............: " + LasUtil.getDate(lasFile));
      System.out.println("MD................: " + LasUtil.getMd(lasFile));
      double[] interval = LasUtil.getInterval(lasFile);
      System.out.println("Interval..........: " + interval[0] + " - " + interval[1]);

      // Loop over all curves
      for (LasCurve curve :lasFile.getCurves()) {

        // Write curve information ...
        System.out.println("  " + curve.getName() +
                           curve.getUnit() + ", " +
                           curve.getValueType() +
                           curve.getDescription());

        //
        // ... and the first few curve values
        //
        for (int index = 0; index < curve.getNValues(); index++) {
          System.out.print(curve.getValue(index) + ", ");
          if (index == 10) { // Write a few values only
            System.out.println("...");
            break;
          }
        }
      }
    }
  }
}
```

Creating LAS files from scratch includes populating **LasFile** instances with *sections* and *curves* and writing them to disk using the **LasFileWriter** class.

The mandatory **~VERSION** section will be auto-populated. The mandatory **~WELL** section must be created by the client, but parameters associated with the curve data (**STRT**, **STOP**, **STEP** and **NULL**) will be auto-populated *if* the section is present.

The optional parameter sections must be provided by the client.

The definition section(s) (**~CURVE**) will be auto-populated based on provided curves.

A complete example program for writing a LAS file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;

import no.petroware.logio.las.LasCurve;
import no.petroware.logio.las.LasFile;
import no.petroware.logio.las.LasFileWriter;
import no.petroware.logio.las.LasParameterRecord;
import no.petroware.logio.las.LasSection;
import no.petroware.logio.las.LasVersion;

/**
 * Class for demonstrating the Log I/O LAS write API.
 */
public final class LasWriteTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    //
    // Create LasFile instance
    //
    LasFile lasFile = new LasFile("WLC_COMPOSITE", LasVersion.VERSION_2_0);

    //
    // Create the mandatory WELL section
    //
    LasSection wellSection = new LasSection("Well");
    wellSection.addRecord(new LasParameterRecord("WELL", null, "16/2-16", "Well name"));
    wellSection.addRecord(new LasParameterRecord("CTRY", null, "Norway", "Country"));
    wellSection.addRecord(new LasParameterRecord("COMP", null, "Lundin", "Company"));
    wellSection.addRecord(new LasParameterRecord("FLD", null, "Johan Sverdrup", "Field name"));
    wellSection.addRecord(new LasParameterRecord("SRVC", null, "Schlumberger", "Service company"));
    lasFile.addSection(wellSection);

    //
    // Create an optional parameter section
    //
    LasSection parameterSection = new LasSection("Parameters");
    parameterSection.addRecord(new LasParameterRecord("Rshl", "OHMM", "2.0000", "Resistivity shale"));
    parameterSection.addRecord(new LasParameterRecord("TLI", null, "149.5000", "Top Logged Interval"));
    parameterSection.addRecord(new LasParameterRecord("VshCutoff", "V/V", "0.2500", "Shale Volume Cutoff"));
    lasFile.addSection(parameterSection);

    //
    // Create and populate curves
    //
    LasCurve depthCurve = new LasCurve("DEPT", "m", "Measured depth", Double.class);
    depthCurve.addValue(4350.00);
    depthCurve.addValue(4350.50);
    depthCurve.addValue(4351.00);
    depthCurve.addValue(4351.50);
    depthCurve.addValue(4352.00);
    lasFile.addCurve(depthCurve);

    LasCurve gammaRayCurve = new LasCurve("GR", "gAPI", "Gamma ray", Double.class);
    gammaRayCurve.addValue(12.3);
    gammaRayCurve.addValue(8.43);
    gammaRayCurve.addValue(null);  // No-value
    gammaRayCurve.addValue(4.1);
    gammaRayCurve.addValue(7.29);
    lasFile.addCurve(gammaRayCurve);

    //
    // Write instance to disk
    //
    LasFileWriter.write(new File("/path/to/file.LAS"), lasFile);
  }
}
```

The program above will produce the following LAS file:

```
~Version
VERS. 2.0 : CWLS LAS ASCII Standard - Version 2.0
WRAP. NO  : One line per depth step

~Well
WELL.  16/2-16        : Well name
CTRY.  Norway         : Country
COMP.  Lundin         : Company
FLD .  Johan Sverdrup : Field name
SRVC.  Schlumberger   : Service company
STRT.m         4350.0 : Start depth
STOP.m         4352.0 : Stop depth
STEP.m            0.5 : Step
NULL.         -999.25 : No-value

~Parameters
Rshl     .OHMM   2.0000 : Resistivity shale
TLI      .     149.5000 : Top Logged Interval
VshCutoff.V/V    0.2500 : Shale Volume Cutoff

~Curves
DEPT.m     : Measured depth
GR  .gAPI  : Gamma ray

~Ascii
4350.0 12.30000
4350.5  8.43000
4351.0  -999.25
4351.5  4.10000
4352.0  7.29000

```

## BIT
*The Basic Information Tape* (BIT) is a binary well log format created by [Dresser Atlas](https://en.wikipedia.org/wiki/Dresser_Industries) in in the 1970's.

Each physical BIT disk file can contain one or more logical BIT files. Each logical file is composed by a simple *General Heading* record followed by a number of *Data* records holding data for a maximum of 20 curves (not including the index channel of depth or time). All curve data is 4-byte floating point in the IBM System/360 representation.

The BIT format is no longer in common use, but vast amounts of historic logging data still exists in this format. The format is quite simple, but no public description of it is available.

A complete example program for reading a BIT file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.bit.BitCurve;
import no.petroware.logio.bit.BitFile;
import no.petroware.logio.bit.BitFileHeader;
import no.petroware.logio.bit.BitFileReader;

/**
 * Class for demonstrating the Log I/O BIT read API.
 */
public final class BitReadTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    File file = new File("path/to/file.BIT");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the BIT file including bulk data. Report the performance.
    //
    BitFileReader reader = new BitFileReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    List<BitFile> bitFiles = reader.read(true);
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    //
    // Report number of subfiles. Loop over each of them.
    //
    System.out.println();
    System.out.println("The file contains " + bitFiles.size() + " subfile(s):");
    for (BitFile bitFile : bitFiles) {
      System.out.println("File name: " + bitFile.getName());
      System.out.println();

      //
      // Get file header containing logging meta-data
      //
      LisGeneralHeadingRecord header = bitFile.getHeader();
      System.out.println("Well name.....: " + header.getWellName());
      System.out.println("Company.......: " + header.getCompanyName());
      System.out.println("Time..........: " + header.getTime();

      //
      // Loop over all curves
      //
      for (BitCurve curve : bitFile.getCurves()) {
        System.out.println("  " + curve.getName() + " " +
                           curve.getQuantity() +
                           curve.getUnit() + " " +
                           curve.getValueType() + " " +
                           curve.getDescription());

        //
        // ... and list the first few values
        //
        System.out.println("    ");
        for (int index = 0; index < curve.getNValues(); index++) {
          System.out.println(curve.getValue(index) + "; ");
          if (index == 10) {
            break;
            System.out.println("...");
          }
        }
      }
    }
  }
}
```

Creating BIT files from scratch includes populating **BitFile** instances with a proper header and associated curve data and writing them to disk using the **BitFileWriter** class.

A complete example program for writing a BIT file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.bit.BitCurve;
import no.petroware.logio.bit.BitFile;
import no.petroware.logio.bit.BitFileWriter;
import no.petroware.logio.bit.BitGeneralHeadingRecord;

/**
 * Class for demonstrating the Log I/O BIT write API.
 */
public final class BitWriteTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    //
    // Create BIT file with appropriate header
    //
    BitGeneralHeadingRecord header = new BitGeneralHeadingRecord(0.0,
                                                                 "Halliburton",
                                                                 "34/2-16B T2",
                                                                 new Date(),
                                                                 new List<String>() {"GR", "NEU"},
                                                                 startDepth,
                                                                 4350.0,
                                                                 4352.0,
                                                                 0.5,
                                                                 "",
                                                                 "1");
      header.setDepthUnit("m");

      BitFile bitFile = new BitFile(header);

      //
      // Curve data
      //
      BitCurve gammaRayCurve = bitFile.getCurves().get(1); // Index is 0
      gammaRayCurve.addValue(12.3);
      gammaRayCurve.addValue(8.43);
      gammaRayCurve.addValue(null);  // No-value
      gammaRayCurve.addValue(4.1);
      gammaRayCurve.addValue(7.29);

      BitCurve neutronCurve = bitFile.getCurves().get(2);
      neutronCurve.addValue(0.89);
      neutronCurve.addValue(1.13);
      neutronCurve.addValue(2.32);
      neutronCurve.addValue(0.71);
      neutronCurve.addValue(0.99);

      //
      // Write instance to disk
      //
      BitFileWriter writer = new BitFileWriter(new File("/path/to/file.BIT"));
      writer.write(bitFile);
    }
  }
}
```

## XTF
The *XTF* format is a binary well log format created by [Western Atlas](https://en.wikipedia.org/wiki/Western_Atlas) in the 1990's. The format does not define its own binary encoding, but reflects the coding of the originating system (i.e. IEEE, VAX, IBM, big-endian, little-endian etc.).

An XTF file is divided into blocks of 4096 bytes each and keeps a bitmap index defining valid blocks. Consequently, information in an XTF file can be extended, replaced or removed without rewriting the entire file.

An XTF file consists of a predefined *file header*, a *well header*, a number of *curves*, each having a *curve header* and single- or multi-dimensional data. Optionally the file can contain predefined *data types* such as *well variables*, *zones*, *section headers* or client defined types.

Unllike most other well log formats the curves of an XTF file does not have a common index. Each curve of the file is evenly sampled and keeps reference to its own start index value and sample rate.

A complete example program for reading an XTF file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;

import no.petroware.logio.xtf.XtfCurve;
import no.petroware.logio.xtf.XtfCurveHeader;
import no.petroware.logio.xtf.XtfFile.
import no.petroware.logio.xtf.XtfFileHeader;
import no.petroware.logio.xtf.XtfFileReader;
import no.petroware.logio.xtf.XtfWellHeader;

/**
 * Class for demonstrating the Log I/O XTF read API.
 */
public final class XtfReadTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    File file = new File("path/to/file.XTF");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the XTF file. Report the performance.
    //
    XtfFileReader reader = new XtfFileReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    XtfFile xtfFile = reader.read();
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    //
    // Capture the file header
    //
    XtfFileHeader fileHeader = xtfFile.getFileHeader();
    System.out.println(fileHeader);

    //
    // Capture the well header
    //
    XtfWellHeader wellHeader = xtfFile.getWellHeader();
    System.out.println(wellHeader);

    //
    // Loop over all curves
    //
    for (XtfCurve curve : xtfFile.getCurves()) {

      // Write the curve header
      XtfCurveHeader curveHeader = curve.getHeader();
      System.out.println(curveHeader);

      // Write the curve values from first dimension
      // Note that the actual type of the values are obtained from
      // curve.getValueType()
      for (int index = 0; index < curve.getNValues(); index++) {
        Object value = curve.getValue(0, index);
        System.out.print(value + ",")
      }
    }
  }
}
```

## SPWLA
SPWLA is a format for core log data. It was developed by the *Aberdeen Well Log Analysis Society* (a chapter of the [Society of Petrophysics and Well Log Analysts](https://www.spwla.org/)) in 1985.

SPWLA files are text based and coded as ASCII or EBCDIC (IBM 500). Log I/O handles both types transparently.

An SPWLA file consist of a set of measurements for a predefined set of properties associated with *cores* and *plugs* along a borehole.

The code example below indicates the few simple steps necessary for accessing an SPWLA file using the Log I/O library:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.spwla.SpwlaCore;
import no.petroware.logio.spwla.SpwlaFile;
import no.petroware.logio.spwla.SpwlaFileReader;
import no.petroware.logio.spwla.SpwlaLogSet;
import no.petroware.logio.spwla.SpwlaPlug;
import no.petroware.logio.spwla.SpwlaWellSurvey;
import no.petroware.logio.spwla.SpwlaWellTest;

/**
 * Class for demonstrating the Log I/O SPWLA read API.
 */
public final class SpwlaReadTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    File file = new File("path/to/file.SPWLA");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the SPWLA file. Report the performance.
    //
    SpwlaFileReader reader = new SpwlaFileReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    List<SpwlaFile> spwlaFiles = reader.read();
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    //
    // Report number of subfiles. Loop over each of them.
    //
    System.out.println();
    System.out.println("The file contains " + spwlaFiles.size() + " subfile(s):");
    for (SpwlaFile spwlaFile : spwlaFiles) {

      // Report file name
      System.out.println("Name: " + spwlaFile.getName());

      // Write well header
      System.out.println(spwlaFile.getWellHeader());

      //
      // Loop over all cores
      //
      for (SpwlaCore core : spwlaFile.getCores()) {
        System.out.println(core);

        // Loop over all plugs. A plug has a top and a bottom depth
        // some properties associated with the plug, and then log measurements
        // organized in a SpwlaLogSet, being a compund of SPWLA log curves.
        for (SpwlaPlug plug : core.getPlugs()) {
          System.out.println(plug);
        }
      }

      //
      // Loop over all well surveys
      //
      for (SpwlaWellSurvey wellSurvey : spwlaFile.getWellSurveys()) {
        System.out.println(wellSurvey);
      }

      //
      // Loop over all well test
      //
      for (SpwlaWellTest wellTest : spwlaFile.getWellTests()) {
        System.out.println(wellTest);
      }

      //
      // Loop over all log sets
      //
      for (SpwlaLogSet logSet : spwlaFile.getLogSets()) {
        System.out.println(logSet);
      }
    }
  }
}
```

## ASC
*ASC* is a collective term used in the industry for denoting *any* ASCII based file that may contain well log information.

The Log I/O ASC reader uses advanced pattern recognising technology in order to automatically digest the vast number of different inputs it may be passed. It will handle column based, token based or delimiter (CSV) based bulk input through a *best effort* approach, and it also handles provided header information. The ASC reader may be used to read any spreadsheet files saved as CSV.

The associated ASC *writer* may produce pretty printed column based or CSV output as well as condensed CSV output if required.

A complete example program for reading an ASC file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;

import no.petroware.logio.asc.AscCurve;
import no.petroware.logio.asc.AscFile;
import no.petroware.logio.asc.AscFileReader;
import no.petroware.logio.asc.AscUtil;

/**
 * Class for demonstrating the Log I/O ASC read API.
 */
public final class AscReadTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    File file = new File("path/to/file.ASC");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the ASC file including bulk data. Report the performance.
    //
    AscFileReader reader = new AscFileReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    AscFile ascFile = reader.read(true);
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    // Report file name
    System.out.println("File name: " + ascFile.getName());

    //
    // Report the meta-data we can conveniently extract through AscUtil
    //
    System.out.println("-- Meta-data as reported by AscUtil:");
    System.out.println("Well name.........: " + Asctil.getWellName(ascFile));
    System.out.println("Wellbore name.....: " + Asctil.getWellboreName(ascFile));
    System.out.println("Bit size..........: " + Asctil.getBitSize(ascFile));
    System.out.println("Run number........: " + Asctil.getRunNumber(ascFile));
    System.out.println("Field name........: " + Asctil.getFieldName(ascFile));
    System.out.println("Rig name..........: " + Asctil.getRigName(ascFile));
    System.out.println("Company...........: " + Asctil.getCompany(ascFile));
    System.out.println("Service company...: " + Asctil.getServiceCompany(ascFile));
    System.out.println("Country...........: " + Asctil.getCountry(ascFile));
    System.out.println("Drill permit #....: " + Asctil.getDrillPermitNumber(ascFile));
    System.out.println("Date..............: " + Asctil.getDate(ascFile));
    Object[] interval = AscUtil.getInterval(ascFile);
    System.out.println("Interval..........: " + interval[0] + " - " + interval[1]);

    //
    // Report all meta-data
    //
    System.out.println("-- All meta-data:");
    for (String key : ascFile.getMetaDataKeys()) {
      System.out.println(key + " = " + ascFile.getMetaData(key));

    // Loop over all curves
    for (AscCurve curve : ascFile.getCurves()) {

      // Write curve information ...
      System.out.println("  " + curve.getName() +
                         curve.getUnit() + ", " +
                         curve.getValueType() +
                         curve.getDescription());

      //
      // ... and the first few curve values
      //
      for (int index = 0; index < curve.getNValues(); index++) {
        System.out.print(curve.getValue(index) + ", ");
        if (index == 10) { // Write a few values only
          System.out.println("...");
          break;
        }
      }
    }
  }
}
```

Creating ASC files from scratch includes populating **AscFile** instances with *meta-data* and *curves* and writing them to disk using the **AscFileWriter** class.

A complete example program for writing an ASC file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;

import no.petroware.logio.asc.AscCurve;
import no.petroware.logio.asc.AscFile;
import no.petroware.logio.asc.AscFileWriter;

/**
 * Class for demonstrating the Log I/O ASC write API.
 */
public final class AscWriteTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    //
    // Create AscFile instance
    //
    AscFile ascFile = new LasFile("WLC_COMPOSITE", ",");

    //
    // Add meta-data
    //
    ascFile.addMetaData("Well", "16/2-16");
    ascFile.addMetaData("Country", "Norway");
    ascFile.addMetaData("Operator", "Lundin");
    ascFile.addMetaData("Field", "Johan Sverdrup");
    :

    AscCurve depthCurve = new AscCurve("Depth", "Length", "m", "Meausred depth", Double.class);
    depthCurve.addValue(4350.00);
    depthCurve.addValue(4350.50);
    depthCurve.addValue(4351.00);
    depthCurve.addValue(4351.50);
    depthCurve.addValue(4352.00);
    ascFile.addCurve(depthCurve);

    AscCurve gammaRayCurve = new AscCurve("GR", null, "gAPI", "Gamma ray", Double.class);
    gammaRayCurve.addValue(12.3);
    gammaRayCurve.addValue(8.43);
    gammaRayCurve.addValue(null);  // No-value
    gammaRayCurve.addValue(4.1);
    gammaRayCurve.addValue(7.29);
    ascFile.addCurve(gammaRayCurve);

    //
    // Write instance to disk
    //
    AscFileWriter.write(new File("/path/to/file.LAS"), ascFile);
  }
}
```

The program above will produce the following ASC file:

```
Well:     16/2-16
Country:  Norway
Operator: Lundin
Field:    Johan Sverdrup

Depth,     GR
m,         gAPI
4350.000,  12.30
4350.500,   8.43
4351.000,
4351.500,   4.10
4352.000,   7.29
```

## WITSML
WITSML is a data interchange standard primarily used with real-time operations in the petroleum industry. WITSML was created in 2000 and is maintained by the [Energistics](https://energistics.org/) consortium.

WITSML exists in different versions, the most common being [WITSML 1.3](http://w3.energistics.org/schema/witsml_v1.3.1_data/doc/WITSML_Schema_docu.htm) from 2006, [WITSML 1.4](http://w3.energistics.org/schema/witsml_v1.4.1_data/doc/witsml_schema_overview.html) from 2011 and [WITSML 2.0](http://w3.energistics.org/energyML/data/witsml/v2.0/doc/witsml_schema_overview.html) from 2017.

WITSML data is human readable text encoded as XML. The WITSML data model defines more than 20 different object types, but the most common is the well log. Log I/O can be used to read files stored in the WITSML format and also be used to produce the XML stream for transmitting well logs from WITSML servers.

A complete example program for reading a WITSML log file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.witsml.WitsmlCurve;
import no.petroware.logio.witsml.WitsmlFile;
import no.petroware.logio.witsml.WitsmlReader;

/**
 * Class for demonstrating the Log I/O WITSML read API.
 */
public final class WitsmlReadTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    File file = new File("path/to/file.xml");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the WITSML log file including bulk data. Report the performance.
    //
    WitsmlReader reader = new WitsmlReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    List<WitsmlLog> witsmlLog = reader.read(true, true, null);
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    //
    // Report number of logs. Loop over each of them.
    //
    System.out.println();
    System.out.println("The file contains " + witsmlLogs.size() + " log(s):");
    for (WitsmlLog witsmlLog : witsmlLogs) {
      //
      // Report some of the meta data
      //
      System.out.println("-- Well known metadata:");
      System.out.println("Log name..........: " + witsmlLog.getName());
      System.out.println("Log description...: " + witsmlLog.getDescription());
      System.out.println("Well name.........: " + witsmlLog.getWellName());
      System.out.println("Wellbore name.....: " + witsmlLog.getWellboreName());
      System.out.println("Creation date.....: " + witsmlLog.getCreationDate());
      System.out.println("Service company...: " + witsmlLog.getServiceCompany());
      System.out.println("Run number........: " + witsmlLog.getRunNumber());

      //
      // Report #curves and #values
      //
      int nCurves = witsmlLog.getNCurves();
      int nValues = witsmlLog.getNValues();

      System.out.println();
      System.out.println("The file contains " + nCurves + " curves of " + nValues + " values each.");
      System.out.println();

      //
      // For each curve, report curve info, ...
      //
      for (WitsmlCurve curve : witsmlLog.getCurves()) {
         System.out.println(" " + curve.getName() +
                           "[" + curve.getNDimensions() + "], " +
                           curve.getUnit() + ", " +
                           curve.getValueType() +
                           curve.getDescription());

        //
        // ... and the first few values from the first couple of dimensions:
        //
        for (int dimension = 0; dimension < curve.getNDimensions(); dimension++) {
          System.out.print(" ");
          for (int index = 0; index < curve.getNValues(); index++) {
          System.out.print(curve.getValue(dimension, index) + "; ");
            if (index == 10)
              break; // Write a few values only
          }
          System.out.println("...");

          if (dimension == 10) { // Write a few dimensions only
            System.out.println(" :");
            break;
          }
        }
      }
    }
  }
}
```

Creating WITSML log files from scratch includes populating **WitsmlLog** instances with metadata and curves and writing them to disk using the **WitsmlWriter** class.

A complete example program for writing a WITSML log file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;

import no.petroware.logio.json.JsonCurve;
import no.petroware.logio.json.JsonLog;
import no.petroware.logio.json.JsonWriter;

/**
* Class for demonstrating the Log I/O WITSML write API.
*/
public final class WitsmlWriteTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    //
    // Create WitsmlLog instance
    //
    WitsmlLog witsmlLog = new WitsmlLog();

    //
    // Add metadata
    //
    witsmlLog.setName("EcoScope data");
    witsmlLog.setWellName("35/12-6S");
    witsmlLog.setServiceCompany("Logtek AS");

    //
    // Create and populate curves
    //
    WitsmlCurve mdCurve = new WitsmlCurve("MD", "Measured depth", "length", "m", Double.class, 1);
    mdCurve.addValue(2907.79);
    mdCurve.addValue(2907.80);
    mdCurve.addValue(2907.81);
    mdCurve.addValue(2907.82);
    witsmlLog.addCurve(mdCurve);

    WitsmlCurve rsCurve = new WitsmlCurve("A40H", "Resistivity", "electrical resistivity", "ohm.m", Double.class, 1);
    rsCurve.addValue(29.955);
    rsCurve.addValue(28.892);
    rsCurve.addValue(null);
    rsCurve.addValue(31.451);
    witsmlLog.addCurve(rsCurve);

    //
    // Write to WITSML 1.4 file, human readable with 2 space indentation
    //
    WitsmlWriter writer = new WitsmlWriter(new File("/path/to/file.xml"), WitsmlVersion.VERSION_1_4, true, 2);
    writer.write(witsmlLog);
    writer.close();
  }
}
```

The program above will produce the following WITSML file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<logs version="1.4.1.1" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.witsml.org/schemas/1series">
  <log>
    <nameWell>35/12-6S</nameWell>
    <name>EcoScope data</name>
    <serviceCompany>Logtek AS</serviceCompany>
    <indexType>measured depth</indexType>
    <indexCurve>MD</indexCurve>
    <direction>increasing</direction>
    <startIndex uom="m">2907.79</startIndex>
    <endIndex uom="m">2907.82</endIndex>
    <logCurveInfo uid="MD">
      <mnemonic>MD</mnemonic>
      <unit>m</unit>
      <curveDescription>Measured depth</curveDescription>
      <classWitsml>length</classWitsml>
      <typeLogData>double</typeLogData>
      <minIndex uom="m">2907.79</minIndex>
      <maxIndex uom="m">2907.82</maxIndex>
    </logCurveInfo>
    <logCurveInfo uid="A40H">
      <mnemonic>A40H</mnemonic>
      <unit>ohm.m</unit>
      <curveDescription>Resistivity</curveDescription>
      <classWitsml>electrical resistivity</classWitsml>
      <typeLogData>double</typeLogData>
      <minIndex uom="m">2907.79</minIndex>
      <maxIndex uom="m">2907.82</maxIndex>
    </logCurveInfo>
    <logData>
      <mnemonicList>MD,A40H</mnemonicList>
      <unitList>m,ohm.m</unitList>
      <data>2907.79,29.955</data>
      <data>2907.8,28.892</data>
      <data>2907.81,</data>
      <data>2907.82,31.451</data>
    </logdata>
  </log>
</logs>
```

## JSON Well Log Format
The [JSON Well Log Format](https://jsonwelllogformat.org/) is a modern well log format created by [Petroware](https://petroware.no/) in 2019 in order to overcome the deficiencies of existing formats and to accomodate for more efficient storage, sharing, transmission and cloud based analytics.
JSON Well Log Format is based on the *JavaScript Object Notation* open standard ([RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259)). It is lightweigt and human readable, it has full [UTF-8](https://en.wikipedia.org/wiki/UTF-8) support, built-in *no-value* support, and a compact type system. Log sets can be depth or time indexed and curves can be single or multi-dimensional. The format defines some key metadata properties as *well known*, otherwise clients may include any information according to the JSON syntax.

JSON Well Log Format files can be generated or parsed by standard JSON components in any software environment. Log I/O includes a convenience *Well Log* API on top of the standard libraries to make it even more comfortable to work with. Log I/O also contains a *validator* for the JSON Well Log Format.

A complete example program for reading a JSON file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;
import java.util.List;

import no.petroware.logio.json.JsonCurve;
import no.petroware.logio.json.JsonLog;
import no.petroware.logio.json.JsonReader;

/**
* Class for demonstrating the Log I/O JSON read API.
*/
public final class JsonReadTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    File file = new File("path/to/file.JSON");

    //
    // Write some meta information about the disk file
    //
    System.out.println("File: " + file.getName());
    if (!file.exists()) {
      System.out.println("File not found.");
      return;
    }
    System.out.println("Size: " + file.length());

    //
    // Read the JSON file including bulk data. Report the performance.
    //
    JsonReader reader = new JsonReader(file);
    System.out.println("Reading...");
    long time0 = System.currentTimeMillis();
    List<JsonLog> jsonLogs = reader.read(true, true, null);
    long time = System.currentTimeMillis() - time0;
    double speed = Math.round(file.length() / time / 1000.0); // MB/s
    System.out.println("Done in " + time + "ms." + "( " + speed + "MB/s)");

    //
    // Report number of sub files. Loop over each of them.
    //
    System.out.println();
    System.out.println("The file contains " + jsonLogs.size() + " subfile(s):");
    for (JsonLog jsonLog : jsonLogs) {
      //
      // Report the well known metadata
      //
      System.out.println("-- Well known metadata:");
      System.out.println("Log name..........: " + jsonLog.getName());
      System.out.println("Log description...: " + jsonLog.getDescription());
      System.out.println("Well name.........: " + jsonLog.getWell());
      System.out.println("Well ID...........: " + jsonLog.getWellId());
      System.out.println("Wellbore name.....: " + jsonLog.getWellbore());
      System.out.println("Field.............: " + jsonLog.getField());
      System.out.println("Country...........: " + jsonLog.getCountry());
      System.out.println("Logging date......: " + jsonLog.getDate());
      System.out.println("Operator..........: " + jsonLog.getOperator());
      System.out.println("Service company...: " + jsonLog.getServiceCompany());
      System.out.println("Run number........: " + jsonLog.getRunNumber());
      System.out.println("Start index.......: " + jsonLog.getStartIndex());
      System.out.println("End index.........: " + jsonLog.getEndIndex());
      System.out.println("Step..............: " + jsonLog.getStep());

      //
      // Report all the meta-data
      //
      System.out.println();
      System.out.println("-- Meta-data details:");
      for (String key : jsonLog.getProperties()) {
        Object property = jsonLog.getProperty(key);
        System.out.println(property);
      }

      //
      // Report #curves and #values
      //
      int nCurves = jsonLog.getCurves().size();
      int nValues = nCurves > 0 ? jsonLog.getCurves().get(0).getNValues() : 0;

      System.out.println();
      System.out.println("The file contains " + nCurves + " curves of " + nValues + " values each.");
      System.out.println();

      //
      // For each curve, report curve info, ...
      //
      for (JsonCurve curve : jsonLog.getCurves()) {
        System.out.println(" " + curve.getName() +
                           "[" + curve.getNDimensions() + "], " +
                           curve.getUnit() + ", " +
                           curve.getValueType() +
                           curve.getDescription());

        //
        // ... and the first few values from the first couple of dimensions:
        //
        for (int dimension = 0; dimension < curve.getNDimensions(); dimension++) {
          System.out.print(" ");
          for (int index = 0; index < curve.getNValues(); index++) {
           System.out.print(curve.getValue(dimension, index) + "; ");
            if (index == 10)
              break; // Write a few values only
          }
          System.out.println("...");

          if (dimension == 10) { // Write a few dimensions only
            System.out.println(" :");
            break;
          }
        }
      }
    }
  }
}
```

Creating JSON Well Log Format files from scratch includes populating **JsonLog** instances with metadata and curves and writing them to disk using the **JsonWriter** class.

A complete example program for writing a JSON file with Log I/O is shown below:

```java
import java.io.File;
import java.io.IOException;

import no.petroware.logio.json.JsonCurve;
import no.petroware.logio.json.JsonLog;
import no.petroware.logio.json.JsonWriter;

/**
* Class for demonstrating the Log I/O JSON write API.
*/
public final class JsonWriteTest
{
  public static void main(String[] arguments)
    throws IOException
  {
    //
    // Create JsonFile instance
    //
    JsonLog jsonLog = new JsonLog();

    //
    // Add metadata
    //
    jsonLog.setName("EcoScope data");
    jsonLog.setWell("35/12-6S");
    jsonLog.setField("Fram");
    jsonLog.setOperator("Wellesley Petroleum");

    //
    // Create and populate curves
    //
    JsonCurve mdCurve = new JsonCurve("MD", "Measured depth", "length", "m", Double.class, 1);
    mdCurve.addValue(2907.79);
    mdCurve.addValue(2907.80);
    mdCurve.addValue(2907.81);
    mdCurve.addValue(2907.82);
    jsonLog.addCurve(mdCurve);

    JsonCurve rsCurve = new JsonCurve("A40H", "Resistivity", "electrical resistivity", "ohm.m", Double.class, 1);
    rsCurve.addValue(29.955);
    rsCurve.addValue(28.892);
    rsCurve.addValue(null);
    rsCurve.addValue(31.451);
    jsonLog.addCurve(rsCurve);

    // Specify metadata for index
    jsonLog.setStartIndex(jsonLog.getActualStartIndex());
    jsonLog.setEndIndex(jsonLog.getActualEndIndex());
    jsonLog.setStep(jsonLog.getActualStep());

    //
    // Write to file, human readable with 2 space indentation
    //
    JsonWriter writer = new JsonWriter(new File("/path/to/file.json"), true, 2);
    writer.write(jsonLog);
    writer.close();
  }
}
```

The program above will produce the following JSON file:

```json
[
  {
    "header": {
      "name": "EcoScope Data" ,
      "well": "35/12-6S",
      "field": "Fram",
      "operator": "Wellesley Petroleum",
      "startIndex": 2907.79,
      "endIndex": 2907.82,
      "step": 0.01
    },
    "curves": [
      {
        "name": "MD",
        "description": "Measured depth",
        "quantity": "length",
        "unit": "m",
        "valueType": "float",
        "dimensions": 1
      },
      {
        "name": "A40H",
        "description": "Resistivity",
        "quantity": "electrical resistivity",
        "unit": "ohm.m",
        "valueType": "float",
        "dimensions": 1
      }
    ]
    "data": [
      [2907.79, 29.955],
      [2907.80, 28.892],
      [2907.81,   null],
      [2907.82, 31.451],
    ]
  }
]
```
