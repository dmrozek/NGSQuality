# NGSQuality
DLL libraries for Azure Data Lake-based DNA Next Generation Sequencing quality control and cleaning

## NGS data extraction
Reading data from files located in the NGS data lake is implemented in the EXTRACT expression of the U-SQL language. The EXTRACT expression consists of a list of attributes extracted, a FROM clause followed by a file path, and a USING clause followed by an instance of the extractor that defines how the files should be read. 
The library that we developed allows extraction from three file formats used to store raw NGS data. With the library, we can read data from FASTQ file format, dedicated to storing NGS raw data. Additionally, we designed a dedicated row-oriented format for processing NGS data on the Azure Data Lake platform, which improves performance of the processing. 

NGS data extraction with EXTRACT phrase
```SQL
@SRR_1 = EXTRACT id int, name string, sequence string,
            optional string, quality string
 FROM @forwardFilePath
 USING new NGSQualityControl.Domain.Extractors.FastqExtractor();

@SRR_2 = EXTRACT id int, name string, sequence string,
            optional string, quality string
 FROM @reverseFilePath
 USING new NGSQualityControl.Domain.Extractors.FastqExtractor();

@SRR_1_2 = SELECT r1.name AS name_r1,  r2.name AS name_r2,
      r1.sequence AS sequence_r1, r2.sequence AS sequence_r2,
      r1.optional AS optName_r1, r2.optional AS optName_r2,
      r1.quality AS qualScore_r1, r2.quality AS qualScore_r2
    FROM @SRR_1 AS r1 JOIN @SRR_2 AS r2  
         ON r1.id == r2.id;
```

NGS data extraction with wrapping functions
```SQL
// extracting from two FASTQ files, for the paired-end sequencing
@SRR988072 = ExtractPairedEndSequences(
   @"/SRR988072_Compressed/SRR988072_1.gz",
   @"/SRR988072_Compressed/SRR988072_2.gz"
);

// extracting from a FASTQ file, for the single-read sequencing
@SRR988072 =  ExtractSingleEndSequences(
   @"/SRR988075_FULL/SRR988075_2.fastq"
);
```

## NGS Data Processing: Improving NGS Data Quality

NGS data processing covers applying a set of transformations for the rowset generated in the Exctract phase. Improving NGS data quality is implemented in the U-SQL and performed through a set of transformations implemented in the Process phase of the EPS process. The set of transformations is modeled based on the capabilities of the Trimmomatic tool for two modes: single-read and paired-end. 

The following commands for improving data quality have been implemented in our tool:

* ILLUMINACLIP -- removes Illuminina adapters from sequence reads,
* SLIDINGWINDOW -- removes nucleotides using the sliding window method; starts scanning at the 5' end and cuts off the read when the average quality in the window falls below the threshold value, 
* MAXINFO -- removes nucleotides with an adaptive method, by balancing the read length and error level to maximize the quality of each read,
* LEADING -- removes nucleotides from the beginning of the sequence as long as their quality is lower than the specified threshold, 
* TRAILING -- removes nucleotides from the end of the sequence as long as their quality is below the specified threshold,
* CROP -- reduces reads to a specified length, 
* HEADCROP -- deletes the specified number of nucleotides from the beginning of the read, 
* TAILCROP -- deletes the specified number of nucleotides from the end of the sequence,
* MINLEN -- deletes the read if its length is shorter than the specified value,
* AVGQUAL -- deletes the sequence if the average quality of its nucleotides is lower than the specified threshold.

NGS data transformations are performed by invoking dedicated processors for the U-SQL queries that are used for parallel processing in the Data Lake environment. We developed two data processors that allow cleaning the NGS reads:
* `FastqPairedEndTrimmerProcessor` (wrapped by the `ProcessPairedEnd` processing function) - allows processing sequence reads obtained as a result of paired-end sequencing. 
* `FastqSingleEndTrimmerProcessor` (wrapped by the `ProcessSingleEnd` processing function) - allows processing sequence reads obtained as a result of single-read sequencing. 

Cleaning NGS data with the developed processors.
```SQL
@SRRSingleEnd_result =
    PROCESS @SRRSingleEnd //processed rowset
    PRODUCE name string, sequence string,
            optionalName string, qualityScore string
    USING new NGSQualityControl.Domain.Processors.
FastqSingleEndTrimmerProcessor(@command, @illuminaAdaptors, 
 (NGSQualityControl.Helper.Infrastructure.QualityEncodingType)@qualityType); 

@SRRPairedEnd_result =
    PROCESS @SRRPairedEnd_1_2 //processed rowset
    PRODUCE name_r1 string, name_r2 string,
            sequence_r1 string, sequence_r2 string,
            optionalName_r1 string, optionalName_r2 string,
            qualityScore_r1 string, qualityScore_r2 string
    USING new NGSQualityControl.Domain.Processors.
FastqPairedEndTrimmerProcessor(@command, @keepUnpaired, @illuminaAdaptors, (NGSQualityControl.Helper.Infrastructure.QualityEncodingType)@qualityType);
```

Cleaning NGS data with the wrapping functions for developed processors
```SQL
@SRR988074_result = ProcessPairedEnd(
   @SRR988074,
   @"ILLUMINACLIP:2:30:10 TAILCROP:10 LEADING:20 TRAILING:20 SLIDINGWINDOW:4:20 MINLEN:30",
   DEFAULT,
   @Res_Lookup,
   DEFAULT
); 

@SRR988075_result = ProcessSingleEnd(
   @SRR988075,
   @"ILLUMINACLIP:2:30:10 TAILCROP:10 LEADING:20 TRAILING:20 MINLEN:30 SLIDINGWINDOW:4:20",
   @Res_Lookup,
   DEFAULT
);
```

## NGS Data Storing

Storing data completes the EPS process for the NGS data. The Store phase implemented in U-SQL covers saving the output of processing scripts to a file in the Data Lake or a database. The data is written to the file using one of the dedicated outputters that we have developed for various formats that NGS data can be stored in. Five different output interfaces have been prepared for this purpose:
* `FastaOutputter` -- saves data to a file in the FASTA format,
* `FastqOutputter` (with the \lstinline|SavePairedEndRowsetDecompressed| and the\lstinline|SaveSingleEnd-RowsetDecompressed| functions) -- saves data to a file in the FASTQ format,
* `FastqGzipOutputter` (with the `SavePairedEndRowsetCompressed` and the  `SaveSingleEndRowsetCompressed` functions) -- saves data to a compressed FASTQ file.
* `FormattedFastqOutputter` (with the `SaveFormattedPairedEndRowsetDecompressed` and the `SaveFormattedSingleEndRowsetDecompressed` functions) -- saves data to a file in the row-oriented version of the FASTQ format; as an argument, it uses a Boolean value that specifies whether the rowset being stored contains readings resulting from paired-end sequencing or only readings from single-read sequencing,
* `FormattedGzipFastqOutputter` (with the `SaveFormattedPairedEndRowsetCompressed` and the `SaveFormattedSingleEndRowsetCompressed` functions) - stores data to the compressed, row-oriented version of the FASTQ format.

