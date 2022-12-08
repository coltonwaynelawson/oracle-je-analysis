# Oracle Fusion Journal Entry (JE) Analysis

The first step of the process is to gain adminsitrator access to the Oracle Fusion Cloud application. We'll use the Oracle Business Intelligence (BI) Publisher to create an enterprise report to analyze all of the journal entries for a particular account on the chart of accounts. 

## Step #1: Write SQL query to pull data for Oracle's data model editor.
The data model editor is used to create data sets to be used in your report. A data set contains the logic to retrieve data from a single data source. A data set can retrieve data from a variety of data sources (for example, a database, an existing data file, a Web service call to another application, or a URL/URI to an external data provider), however this data model will be pulling from the accounting transaction database.

```MySQL
SELECT  SYS_GUID() as "Unique Identifier", #pull a unique identifier column
        "GL_CODE_COMBINATIONS"."SEGMENT<i>" as "Client Number", gl_flexfields_pkg.get_description_sql(chart_of_accounts_id,<i>,segment<i>) as "Client Name", 
        GL_JE_LINES"."DESCRIPTION" as "Journal Line Description",
        TO_CHAR("GL_JE_LINES"."EFFECTIVE_DATE", 'MM/dd/yyyy') as "Accounting Date",
        TO_CHAR("GL_JE_LINES"."CREATION_DATE", 'MM/dd/yyyy') as "Journal Creation Date",
        NVL("GL_JE_LINES"."ACCOUNTED_DR", 0.00) - NVL("GL_JE_LINES"."ACCOUNTED_CR",0.00) as "Amount"
FROM    "FUSION"."GL_CODE_COMBINATIONS" "GL_CODE_COMBINATIONS",
        "FUSION"."GL_JE_LINES" "GL_JE_LINES" 
WHERE   "GL_CODE_COMBINATIONS"."CODE_COMBINATION_ID"="GL_JE_LINES"."CODE_COMBINATION_ID"
AND     "GL_CODE_COMBINATIONS"."SEGMENT<i>" IN ('<Account>') and "GL_JE_LINES"."EFFECTIVE_DATE" >= SYSDATE - INTERVAL '365' DAY
```

The system will use your SQL code to fetch data from the database.
![image](https://user-images.githubusercontent.com/46463801/206571110-6334ab83-b241-426e-9ad7-8274da318a71.png)

Once you have built the query, click Save to return to the data model editor. The query appears in the SQL Query box. Click OK to save the data set. Since we don't want our users to be able to modify the results we will NOT add a bind variable to the query. You'd only do this if you want users to be able to pass a parameter to the query to limit the results. For example, if we were fetching an employee listing, you might want users to be able to choose a specific department.

## Step #2: Define the data output structure.
The Data Model diagram helps you to quickly and easily define data sets, break groups, and totals for a report based on multiple data sets.

Link data sets to define a master-detail (or parent-child) relationship between two data sets. Defining an element-level link enables you to establish the binding between the elements of the master and detail data sets.

In many cases, the data fetched for one part of the data set (or query) is determined by the data fetched for another part. This is often called a "master/detail," or "parent/child," relationship, and is defined with a data link between two data sets (or queries). When you run a master/detail data model, each row of the master (or parent) query causes the detail (or child) query to be executed, retrieving only matching rows.

![image](https://user-images.githubusercontent.com/46463801/206572260-79fa7f70-ee8b-4d32-86f8-9b8d4bc4bdf9.png)

At this time we'd also have the option to define the parameters to pass to the query, define lists of values for users to select parameter values, and define Event Triggers, however we will not. In addition we can define a flexfield is a flexible data field that your organization can customize to your business needs without programming. Oracle applications (Oracle E-Business Suite and Oracle Fusion Applications) use two types of flexfields: key flexfields and descriptive flexfields.

## Step #3: Create the Oracle Business Intelligence Publisher report
In this step you create the Oracle Business Intelligence Publisher report extract used as the data source for the integration with the Oracle Enterprise Performance Management Cloud. On the Create Report page, click Use Data Model to use an existing data model and then from Create a report using an existing Data Model, select the data model and then click Next. We'll need to pull this download as a csv as we are expecting a alarge amount of data since we are pulling all journal entries from the last year. While editing the report, ensure that CSV is included as the default output format for the report. , to do this To select CSV as the default output format, click View as List.

![image](https://user-images.githubusercontent.com/46463801/206574604-b68212b8-3779-40b7-9895-5ef13c374432.png)

## Step 4: Make an HTTP request to the Oracle API
Now we're ready to use Oracle's API to obtain our enterprise level report. Oracle utlizes a SOAP webservice, so we'll need to decipher the XML document containing the report when the response is received from the Oracle Could server. We'll use the following parameters for our request.

**HTTP Method**
```HTML
POST
```

**Request Headers**
```HTML
Content-Type: application/soap+xml
Authorization: Basic <Base-64 encoded username:password>
```

**Request Body**
```HTML
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope" xmlns:pub="http://xmlns.oracle.com/oxp/service/PublicReportService">
  <soap:Header/>
   <soap:Body>
      <pub:runReport>
         <pub:reportRequest>
            <pub:attributeFormat>csv</pub:attributeFormat>
            <pub:flattenXML>false</pub:flattenXML>
            <pub:reportAbsolutePath>/Custom/IT Reports/<Report Name>.xdo</pub:reportAbsolutePath>
            <pub:sizeOfDataChunkDownload>-1</pub:sizeOfDataChunkDownload>
         </pub:reportRequest>
      </pub:runReport>
   </soap:Body>
</soap:Envelope>
```

The report data is contained in the ‘reportBytes’ element and is a Base-64 encoded string which will need to be decoded.  We do this using Python's json library to make the request. API call using HTTP/POST to call the report we just created from Oracle. Because the dataset is very large, the client will sometimes recieve an HTTP status code of 500 with the message Internal Server Error as a response for API calls. The HTTP status code 500 is a generic error response. It means that the server encountered an unexpected condition that prevented it from fulfilling the request. Becasue of this we will try 5 more times to connect to Oracle, sleeping for 30 seconds in between attempts to allow the server to process the data model.

```Python
import json

get_response = requests.post(url, data=soapQueryRequest, headers=headers)
success = False
retries = 1
if get_response.status_code == 500:
  while (success == False) and (retries <= 5):  # try 5 more times to connect to Oracle, sleeping for 30 seconds in between attempts
    print(f'Connection attempt {retries} failed. Trying again...') 
    time.sleep(30)
    get_response = requests.post(url, data=soapQueryRequest, headers=headers)
    print(f'Response status code: {get_response.status_code}')
    if get_response.ok == True:
      success = True
if get_response.ok == False:
  raise Exception(f'There is an issue with the Oracle API response. Response status code: {get_response}')
```

When we've successully called the report, we will need to read and format the data. Applications store data in different formats, and Oracle . This is used to receive data formatted as XML sent using an HTTP/POST coded as utf-8.

```Python
import base64
from io import StringIO

content = get_response.content
with open('output.xml', 'w') as outputXML:
  decoded_content = content.decode('utf-8')
  outputXML.write(decoded_content)
with open('output.xml', 'r') as readXML:
  for line in readXML.readlines():
    if 'reportBytes' in line: # The report data is contained in the ‘reportBytes’ element and is a base-64 encoded string which will need to be decoded.  
      x = line.split('reportBytes')[1]
      x = x.replace('>','').replace('</','')
finalOutput = base64.b64decode(x + '==')
finalOutput = finalOutput.decode('utf-8', errors='ignore')
df = StringIO(finalOutput)
```

## Step 5: Data analysis using Python

We are attempting to find all of the journal entries that don't have a balancing entry. For every credit, there must be a debit. However, Oracle allows the creation of journal entires that don't balance. The difficulty in this analysis is that not all journal entry amounts are the same, so a 5.00 credit could have been balanced with five 1.00 debits for example. This is similar to the 'Two Sum' LeetCode problem where you're given an array of integers and an integer target, and you're tasked with returning indices of the numbers such that they add up to the target, and you may not use the same element twice.

```Python
#filter the data
df = gl[(gl['Client_Number'] == client_code) & (gl['Tax_Value'] == tax_code)]

# format date columns
if df.dtypes['Accounting_Date'] == '<M8[ns]':
  df['Accounting_Date'] = df['Accounting_Date'].apply(lambda x: x.strftime('%Y-%m-%d'))
  df['Journal_Creation_Date'] = df['Journal_Creation_Date'].apply(lambda x: x.strftime('%Y-%m-%d'))

# create a list of the unique amounts
amts = df['Amount'].tolist()
amts_abs = list(set([abs(ele) for ele in amts])) # sets the absolute value of each amount and de-duplicates.

# create a list of lists that shows the [[amount 1, count 1], [amount 2, count 2]]
cnts_of_amts = []
for x in set(amts_abs):
  if x != 0:
    cnts_of_amts.append([[x, amts.count(x)], [(x * -1), amts.count(x * -1)]]) # create a list of counts of the positive values and negative values

# for each item in the list find which items match and find which items don't match
for y in cnts_of_amts:
  # if the items match, print the matching dates in the research columns
  if y[0][1] == y[1][1]: # --- MATCH ---
    for z in y:
      date_list = list(set([df.loc[df['Amount'] == (z[0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
      if len(date_list) == 1:
        df.loc[df['Amount'] == z[0], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
      elif len(date_list) > 1:
        df.loc[df['Amount'] == z[0], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

  # if the items don't match, print the no match for the unmatched amounts and print the matching dates for the matched amounts in the research field
  elif  y[0][1] != y[1][1]: # --- NO MATCH! ---
    if y[0][1] > y[1][1]:
      remove_idx = df.loc[df['Amount'] == y[0][0]].index.values
      for i in range(abs(y[0][1] - y[1][1])):
        # print no match for the amounts that don't have a match ie the difference
        df.loc[remove_idx[i], 'Research_Notes'] = "No match"

      if y[0][1] > 1:
        for i in range(abs(y[0][1] - y[1][1]), y[0][1]):
          # print dates for the first list item if the first list item is greater than the second
          date_list = list(set([df.loc[df['Amount'] == (y[0][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
          include_idx = df.loc[df['Amount'] == y[0][0]].index.values
          if len(date_list) == 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
          elif len(date_list) > 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

        for i in range(y[1][1]):
          # print dates for the second list item
          date_list = list(set([df.loc[df['Amount'] == (y[1][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
          include_idx = df.loc[df['Amount'] == y[1][0]].index.values
          if len(date_list) == 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
          elif len(date_list) > 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

    elif y[1][1] > y[0][1]:
      remove_idx = df.loc[df['Amount'] == y[1][0]].index.values
      for i in range(abs(y[1][1] - y[0][1])):
        # print no match for the amounts that don't have a match ie the difference
        df.loc[remove_idx[i], 'Research_Notes'] = "No match"

      if y[1][1] > 1:
        for i in range(abs(y[1][1] - y[0][1]), y[1][1]):
          # print dates for the first list item if the second list item is greater than the first
          date_list = list(set([df.loc[df['Amount'] == (y[1][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
          include_idx = df.loc[df['Amount'] == y[1][0]].index.values
          if len(date_list) == 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
          elif len(date_list) > 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

        for i in range(y[0][1]):
          # print dates for the second list item
          date_list = list(set([df.loc[df['Amount'] == (y[0][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n')))
          include_idx = df.loc[df['Amount'] == y[0][0]].index.values
          if len(date_list) == 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
          elif len(date_list) > 1:
            df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

# iterate through all of the 'no match' line items using itertools to see if a combination of two or more line items makes up the balancing entry
no_match = df[df['Research_Notes'] == "No match"]['Amount'].to_list()
no_match_neg = [num for num in no_match if num < 0]
no_match_pos = [num for num in no_match if num > 0]

if len(no_match_neg) < 20: # this portion of the script would take way too long if there were more than 20 values to iterate through
  for i in range(len(no_match_pos)):
    target = no_match_pos[i] # the number we are looking for

    # use itertools to iterate through the remaining list items to see if there is a combination that adds to the target
    result = [seq for i in range(len(no_match_neg), 0, -1) for seq in itertools.combinations(no_match_neg, i) if sum(seq) == (target * -1)]

    if len(result) == 1:
      # print the dates for the target line item
      result = list(result[0]) # turn the result into a list
      date_list = [(df.loc[df['Amount'] == x, 'Accounting_Date'].to_string(index=False)) for x in result]
      target_idx = df.loc[df['Amount'] == target].index # get the index of the target
      df.loc[target_idx, 'Research_Notes'] = f"Balancing entries occur on all of the following dates: {*sorted(date_list),}"

      # print the dates for the result line items
      target_date = df.loc[target_idx, 'Accounting_Date'].to_string(index=False) # get the date of the target
      for amt in result:
        result_idx = df.loc[df['Amount'] == amt].index.values
        df.loc[result_idx, 'Research_Notes'] = f"Balancing entry occurs on date: ('{target_date}')"

if len(no_match_pos) < 20: # this portion of the script would take way too long if there were more than 20 values to iterate through
  for i in range(len(no_match_neg)):
    target = no_match_neg[i] # the number we are looking for

    # use itertools to iterate through the remaining list items to see if there is a combination that adds to the target
    result = [seq for i in range(len(no_match_pos), 0, -1) for seq in itertools.combinations(no_match_pos, i) if sum(seq) == (target * -1)]

    if len(result) == 1:
      # print the dates for the target line item
      result = list(result[0]) # turn the result into a list
      date_list = [(df.loc[df['Amount'] == x, 'Accounting_Date'].to_string(index=False)) for x in result]
      target_idx = df.loc[df['Amount'] == target].index # get the index of the target
      df.loc[target_idx, 'Research_Notes'] = f"Balancing entries occur on all of the following dates: {*sorted(date_list),}"

      # print the dates for the result line items
      target_date = df.loc[target_idx, 'Accounting_Date'].to_string(index=False) # get the date of the target
      for amt in result:
        result_idx = df.loc[df['Amount'] == amt].index.values
        df.loc[result_idx, 'Research_Notes'] = f"Balancing entry occurs on date: ('{target_date}')"

# if the amount is zero, print no match in the research column
df.loc[df['Amount'] == 0, 'Research_Notes'] = "Amount is zero - ignore"

  ```
