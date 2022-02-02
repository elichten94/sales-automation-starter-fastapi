# Import Prospects from a CSV File Implementation

<!-- Please fill out commented out fields -->

## Step 1: Create File Preview for Matching Columns

The first step of the feature is to generate a preview of the file to allow the user to select which column represents the email address, first name, and/or last name of the prospects in the CSV file.

To generate the preview of the csv file for the user, you can either do that by uploading the file to the back-end server, and parse it there or you can open and parse the file directly on the front-end client (in the browser). In this implementation, we are going to use the front-end to generate the preview data using a package like `FileReader`.

## Step 2: Upload CSV API Route

The second step of this feature is to upload the CSV to the back-end with the mapping data (which csv column index corresponds to the correct prospect fields in our database). Please fill in the details about this API route below:

**HTTP Method:** POST

**Path:** /api/prospects/import

**Request Payload:**

Body parameters:
| name | type | required | description |
| :---: | :---: | :---: | :--- |
| `file` | `FILE` | yes | The CSV file with prospect rows |
| `prospect_email_column` | `string` | yes | column name of emails in CSV file |
| `prospect_first_name_column` | `string` | no | column name for first names in CSV file |
| `prospect_last_name_column` | `string` | no | column name of last name in CSV file |
| `user_email` | `string` | yes | user account email to verify user uploading prospects |
| `user_password` | `string` | yes | user account password to verify user uploading prospects |


Example payload:
```JSON
{
	"file": "new_prospects.csv",
   "prospect_email_column": "email_address",
   "prospect_first_name_column": "given_name",
   "prospect_last_name_column": "surname",
   "user_email": "sample@samplemail.com",
   "user_password": "sample1234"
}
```


**Successful Response Status & Payload:**

<!-- Please fill in any parameters you would need in the response payload and the status code for a successful response -->

### Response upon successful file upload
The csv file is uploaded asynchronously. A success response is sent after contents are uploaded to the database.
| key | value | description |
| :---: | :---: | :--- |
| `uploaded_prospects` | int | Number of rows loaded |
| `overwrite_existing` | bool | option selected by user on upload client |
|`skipped_existing_prospects` | int | Any rows that were skipped as duplicates - this property will not appear if `overwrite_existing` is set to `true` |
| `ignore_header_row` | bool | option selected by user on upload client|


Response: `201 Created`
Body (example):
```JSON
{
   "uploaded_prospects": 65,
   "overwrite_existing": false,
   "skipped_existing_prospects": 3,
   "ignore_header_row": true
}
```


**Analysis of Error Cases:**

<!-- Please mention and discuss any possible errors that you could see happening in the upload/import process and how you would handle them and what payload/status codes you would return -->

**Error response body:**
All errant requests include a response body with attributes:
| Key | Value (`string`) |
| :---: | :--- |
| `ERROR_TYPE` | Type of error that occurred (bad field, unauthorized, unprocessable entity, etc.) |
| `DETAILS` | Why the error occurred, including any incorrectly entered or missing fields |


<!-- missing body parameters -->
<!-- invalid user credentials -->
<!-- missing authentication -->
| Error case | Response | Possible causes |
| :--- | :---: | :--- |
| Missing body parameters | `400 Bad Request` | Client did not call API correctly |
| Invalid User Credentials | `404 Not Found` | User does not exist |
| Missing authentication keys/tokens | `401 Unauthorized` | Client did not provide bearer token |
| Wrong file extension | `400 Bad Request` | user uploaded a file without .csv extension |
| Corrupt file | `400 Bad Request` | The file could not be read |
| Non existing csv column | `400 Bad Request` | Column names parsed by client do not match actual fields in CSV |
| File too large | `400 Bad Request` | The file being sent exceeds size limit |
| Data column not found | `500 Internal Server Error` | Schema was updated without handling requests accordingly |
| Upload failed |  `500 Internal Server Error` | The server failed to upload prospects to the database |

Example response:
```JSON
{
   "ERROR_TYPE": "UPLOAD_FAILED",
   "DETAILS": "32 rows in file 'prospects2022.csv' failed to load"
}
```

**Database Changes**

<!--
If we wanted to know for each imported prospect, which original file it was imported from, how would you store this information?
If we wanted to keep a history of past imports by a user, what database changes would be required?
-->

Update schema:
1. Create a table `Files`:
| Column name | Description |
| :---: | :--- |
| `id` | (primary key) auto incremented integer |
| `name`: | Name of file |
| `size` | Size of file in bytes |
| `content` | Contents of file |
| `user_id` | (foreign key) id of user that uploaded the file |
| `uploaded_at` | Timestamp |

2. Prospects should be given a `file_id` column that points to a row in `Files` table



**Discuss Alternatives**

<!-- You can leverage "background tasks" to implement this API call. Why or why not would you want to do that? -->
A large csv file could take a long time to upload. Sending a response upon receipt of the file and valid parameters would allow the client to proceed without stalling. Once the upload task has finished, the api would emit an event to send the user a notification about successful or failed upload. The user would be able to continue browsing after sand working However, this could fall out of sync with the client if users needed immediate access to all their prospects for another business task following the upload. Users would need to know that not all prospects are available until they are notified of successful upload, which could cause a misperception in utilizing prospects. <br />

<!-- What are some other alternative approaches we can take when implementing this feature? -->
- Use a load balancer for larger files and high upload traffic
- Using a front-end implementation to validate the file before sending to the server, and show the user projected upload time


<!-- Please analyze each alternatives and discuss pros/cons here. -->
Load balancer: <br />
Pros:
- Splitting files into multiple concurrent uploads across multiple databases could reduce upload time.<br />
Cons:
- Complicates the process of retrieving prospects, since they are now scattered
- Database queries would potentially take longer
<br />

Front end file validation:<br />
Pros:
- Increased transparency so users know what to expect for a given file
- Reduces the amount of errant requests <br />
Cons:
- Takes extra time
- The API would still need to validate the file in case something went wrong during the request. This copies the process and possibly complicates the codebase
