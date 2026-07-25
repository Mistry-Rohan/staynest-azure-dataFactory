# Azure Data Factory Raw-to-Bronze Pipeline

This project contains an Azure Data Factory (ADF) pipeline that discovers CSV
files in an Azure Data Lake Storage Gen2 `raw` folder and copies them into a
`bronze` folder.

## Pipeline summary

- **Data factory:** `cbAssignment9asf`
- **Pipeline:** `pipeline1`
- **Source:** `staynest/raw`
- **Destination:** `staynest/bronze`
- **Input format:** Comma-delimited text with a header row
- **Execution model:** Up to three files are copied in parallel

The supplied metadata output shows these source files:

- `bookings.csv`
- `customers.csv`
- `hotels.csv`

The sample copies of these files are in
`data_factory_assignment_assets/data/`.

## How the pipeline works

```text
staynest/raw
     |
     v
getMetaDataActivity
  Reads: childItems
     |
     v
ForEachFileInRawFolder
  Items: @activity('getMetaDataActivity').output.childItems
  Parallel batch count: 3
     |
     v
copyDataActivity
  Source file: @item().name
     |
     v
staynest/bronze
```

1. `getMetaDataActivity` uses `ds_raw_folder` to return the child items in
   `staynest/raw`.
2. `ForEachFileInRawFolder` loops over the returned files. It is not sequential
   and has a `batchCount` of `3`.
3. `copyDataActivity` passes the current file name (`@item().name`) to the
   parameterized `ds_source` dataset.
4. The Copy activity reads the comma-delimited file and writes it through
   `ds_sink` into `staynest/bronze`. Output values are quoted, and the Copy
   activity is configured with a `.txt` file extension.

## Linked service

| Property | Value |
| --- | --- |
| Name | `LS_cbAssignmentStaynest` |
| Type | Azure Data Lake Storage Gen2 (`AzureBlobFS`) |
| Storage URL | `https://cbassignment9.dfs.core.windows.net/` |
| Authentication | Storage account key |
| Integration runtime | `AutoResolveIntegrationRuntime` |

The account key is represented by the secure ARM template parameter
`LS_cbAssignmentStaynest_accountKey`. Its value is intentionally blank in the
parameter files and must be supplied during deployment or configured in ADF.

## Datasets

| Dataset | Purpose | File system | Path/file | Important settings |
| --- | --- | --- | --- | --- |
| `ds_raw_folder` | Lets Get Metadata list the source folder | `staynest` | `raw` | Delimited text, comma delimiter, first row is header |
| `ds_source` | Reads the current file selected by the ForEach activity | `staynest` | `raw/@dataset().fileName` | String parameter `fileName`, comma delimiter, first row is header |
| `ds_sink` | Writes copied data to the bronze layer | `staynest` | `bronze` | Delimited text, comma delimiter, first row is header |

The schemas saved with `ds_raw_folder` and `ds_source` reflect individual
sample files, but the Copy activity uses automatic tabular translation with
type conversion. The sink dataset has no fixed schema.

## Run the pipeline with Debug

### Prerequisites

1. Open Azure Data Factory Studio for `cbAssignment9asf`, or deploy the supplied
   ARM templates to another factory.
2. Open **Manage > Linked services** and confirm
   `LS_cbAssignmentStaynest` has the correct storage URL and a valid account
   key. Test the connection.
3. Confirm the ADLS Gen2 file system `staynest` contains the `raw` and `bronze`
   folders.
4. Upload `bookings.csv`, `customers.csv`, and `hotels.csv` from
   `data_factory_assignment_assets/data/` to `staynest/raw` if they are not
   already present.

### Debug steps

1. In ADF Studio, select **Author**.
2. Open **Pipelines > pipeline1**.
3. Select **Validate all** and resolve any connection or dataset errors.
4. Select **Debug** on the pipeline toolbar. The pipeline has no input
   parameters, so no parameter values are required.
5. In the **Output** pane, verify that:
   - `getMetaDataActivity` succeeds and its output contains a `childItems`
     array.
   - `ForEachFileInRawFolder` succeeds.
   - A `copyDataActivity` run succeeds for each source file.
6. Open the storage account and verify that data was written under
   `staynest/bronze`.

Debug runs execute immediately and are not created by a trigger. Use the
activity output and error details in the ADF Output pane when troubleshooting.

## Deployment files

- `ARMTemplateForFactory.json` contains the linked service, datasets, and
  pipeline.
- `ARMTemplateParametersForFactory.json` contains the corresponding deployment
  parameters.
- `factory/` contains the factory resource ARM export.
- `linkedTemplates/` contains the split ARM templates used for linked-template
  deployment.
- `GetMetaDataOutput.json` is an example output from `getMetaDataActivity`.
- `screenshots/` contains images of the pipeline run and bronze folder.

## Notes

- `ds_sink` specifies the destination folder but does not provide a dynamic
  output file name. ADF may therefore generate output file names. To preserve
  each source name, add a `fileName` parameter to `ds_sink` and pass
  `@item().name` from `copyDataActivity`.
- The pipeline defines a string variable named `fileName`, but the current
  activities do not use it.
