# Oracle Fusion Journal Entry (JE) Analysis

## Executive Summary
The primary goal of the analysis was to extract journal entries from the Oracle Fusion ERP system and identify unbalanced entries for each customer. The data structure extracted from Oracle contained **1,309,144 rows of data**. The analysis would need to be performed in **1,376 iterations**, one for each unique customer in Oracle Fusion. For the analysis I used SQL, Python, and PowerBI.

### Step #1: Write SQL query to pull data into Oracle Fusion's data model editor.
I began the analysis by writing a SQL query to fetch the appropriate data from the database. The query pulls a unique identifier column for each journal entry, and includes client number, client name, journal description, date, and amount for all of the journal entries made in the last year.

```MySQL
SELECT  SYS_GUID() as "Unique Identifier", # pull a unique identifier column
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
I leveraged the Oracle Business Intelligence (BI) Publisher to create a report to analyze all of the journal entries on the chart of accounts. Oracle Fusion’s data model editor was used to write the logic to retrieve data from Oracle Fusion’s tables and create the data sets to be used in the report. 

![image](https://user-images.githubusercontent.com/46463801/206571110-6334ab83-b241-426e-9ad7-8274da318a71.png)

After defining the parameters to pass to the query, defining lists of values for users to select parameter values (none in this case), and defining event triggers, I leveraged Oracle Fusion’s data model diagram to help link and define a master-detail (or parent-child) relationship between data sets. Defining an element-level link enables you to establish the binding between the elements of the master and detail data sets. In many cases, the data fetched for one part of the data set (or query) is determined by the data fetched for another part.

![image](https://user-images.githubusercontent.com/46463801/206572260-79fa7f70-ee8b-4d32-86f8-9b8d4bc4bdf9.png)

## Step #2: Create the Oracle BI Publisher report
I created an Oracle BI Publisher report which utilizes Oracle Fusion data models to create end user reports in Excel, csv, pdf, etc. The report was created in csv format due to the large amount of data (remember the analysis contains 1,309,144 rows of data) and utlizes the data model created in step #1 above.

![image](https://user-images.githubusercontent.com/46463801/206574604-b68212b8-3779-40b7-9895-5ef13c374432.png)

## Step 3: Make an HTTP request to the Oracle API
Now I was ready to extract the data from Oracle Fusion using the Oracle API. Oracle utilizes a SOAP webservice, so I needed to write a Python script that could make a request to the Oracle Fusion Cloud server and then decipher the XML document containing the report. To do this use the following parameters for the request.

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

I leveraged the requests library in Python, as requests is an easy to use and simple HTTP library for Python. I passed the parameters to make an API call to request the report that I created in Oracle Fusion in step #2 above.

Because the dataset was very large, the client was sometimes receiving an HTTP status code of 500 with the message Internal Server Error as a response. The HTTP status code 500 is a generic error response which means that the server encountered an unexpected condition that prevented it from fulfilling the request. Because of this I included a command to try 5 more times to connect to Oracle, sleeping for 30 seconds in between attempts to allow the server to process the data model.

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

The report data is contained in a Base-64 encoded string in the response, so once I successully called the report, I needed to read and format the data. I did this by decoding the xml file from utf-8 using the below Python code. 

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
output = base64.b64decode(x + '==')
output = StringIO(output.decode('utf-8', errors='ignore'))
```

## Step 4: Data analysis using Python

Oracle Fusion allows the creation of journal entires that don't balance (debits that don't have a credit and vice versa), so I need to identify all of the journal entries that don't have a balancing entry. I approached this using a technique similar to hashmap where I created a list of all of the unique journal entry amounts and then counted the number of occurances of each amount. If the number of debits for each amount does not match the number of credits, then I knew that there was an unbalanced entry and printed 'No match' in the Research Notes column. If the number of debits matched the number of credits, then I printed the dates that the balancing entry (or entries) occured in the Research Notes column.

Since I used a brute force approach I calculated the time complexity to be $O(n^2)$.

```Python
import numpy as np
import pandas as pd

def je_analysis(client_code):
  # filter the entire Oracle Fusion dataset to a single customer - we'll perform the analysis one customer at a time
  df = output[output['Client_Number'] == client_code] 

  # find a unique list of journal entry amounts and input those into a list
  nums = df[df['Amount']].unique()
  nums = list(set([abs(ele) for ele in nums])) # sets the absolute value of each amount and de-duplicates.

  # count the number of occurances of each journal entry amount by creating a list of lists (e.g. [[amount 1, count 1], [amount 2, count 2]]...)
  cnts_of_amts = []
  for num in setnums:
    if num != 0: # $0 value journal entires can be ignored
      cnts_of_amts.append([[x, nums.count(x)], [(x * -1), nums.count(x * -1)]]) # create a list of counts of the positive values and negative values

  # for each item in the list find which items match and find which items don't match
  for count in cnts_of_amts:
    # if the items match, print all of the matching dates in the research notes column
    if count[0][1] == count[1][1]: # this is a debit that has a matching credit
      for c in count:
        date_list = list(set([df.loc[df['Amount'] == (c[0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
        if len(date_list) == 1:
          df.loc[df['Amount'] == c[0], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
        elif len(date_list) > 1:
          df.loc[df['Amount'] == c[0], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

    # if the items don't match, print the no match for the unmatched amounts and print the matching dates for all of the matching dates in the research notes column
    elif  count[0][1] != count[1][1]: # this is a debit that DOES NOT have a matching credit
      if count[0][1] > count[1][1]:
        remove_idx = df.loc[df['Amount'] == count[0][0]].index.values
        for i in range(abs(count[0][1] - count[1][1])):
          # print no match for the amounts that don't have a match ie the difference
          df.loc[remove_idx[i], 'Research_Notes'] = "No match"

        if count[0][1] > 1:
          for i in range(abs(count[0][1] - count[1][1]), count[0][1]):
            # print matching dates for the first list item if the first list item is greater than the second
            date_list = list(set([df.loc[df['Amount'] == (count[0][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
            include_idx = df.loc[df['Amount'] == count[0][0]].index.values
            if len(date_list) == 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
            elif len(date_list) > 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

          for i in range(count[1][1]):
            # print dates for the second list item
            date_list = list(set([df.loc[df['Amount'] == (count[1][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
            include_idx = df.loc[df['Amount'] == count[1][0]].index.values
            if len(date_list) == 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
            elif len(date_list) > 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

      elif count[1][1] > count[0][1]:
        remove_idx = df.loc[df['Amount'] == count[1][0]].index.values
        for i in range(abs(count[1][1] - count[0][1])):
          # print no match for the amounts that don't have a match ie the difference
          df.loc[remove_idx[i], 'Research_Notes'] = "No match"

        if count[1][1] > 1:
          for i in range(abs(y[1][1] - count[0][1]), count[1][1]):
            # print dates for the first list item if the second list item is greater than the first
            date_list = list(set([df.loc[df['Amount'] == (count[1][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n'))) # get all dates from the inverse amount, turn into list, and remove dupes
            include_idx = df.loc[df['Amount'] == count[1][0]].index.values
            if len(date_list) == 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
            elif len(date_list) > 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

          for i in range(count[0][1]):
            # print dates for the second list item
            date_list = list(set([df.loc[df['Amount'] == (count[0][0] * -1), 'Accounting_Date'].to_string(index=False)][0].split('\n')))
            include_idx = df.loc[df['Amount'] == count[0][0]].index.values
            if len(date_list) == 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on date: ('{date_list[0]}')"
            elif len(date_list) > 1:
              df.loc[include_idx[i], 'Research_Notes'] = f"Balancing entry occurs on one of the following dates: {*sorted(date_list),}"

  # if the amount is zero, print no match in the research column
  df.loc[df['Amount'] == 0, 'Research_Notes'] = "Amount is zero - ignore"
  return df

  ```

The resulting dataframe looks like this. As you can see the research notes column was added in the Python script and it identifies journal entries that have a balancing entry, and for any journal entry that doesn't balance the Research Notes column displays 'No match.'

| Unique_Identifier | Client_Number | Journal_Line_Description | Journal_Creation_Date | Amount | Research_Notes
| --- | --- | --- | --- | --- | --- |
| A0163 | B55 | MT Adjustment | 12/8/2022 | 85.12 | Balancing entry occurs on date: ('1/26/2022')
| A710F | B55 | Bi-Weekly Regular | 1/5/2022 | -101.42 | No match
| A0164 | B55 | 147556 Federal | 1/26/2022 | -85.12 | Balancing entry occurs on date: ('12/08/2022')
| A7880 | B55 | Bi-Weekly Regular | 1/26/2022 | 10.50 | No match

The dataset pulled from Oracle Fusion contained 1,376 customers, and since journal entries need to balance by customer, I needed to loop through the analysis for every customer and then concatenate the resulting dataframes together to create a master dataframe. 

```Python
df_list = []
client_codes = output[output['Client_Number']].unique()
for client_code in client_codes:
  temp_df = je_analysis(client_code)
  df_list.append(temp_df)
master_df = pd.concat(df_list, ignore_index=True)
```

The master dataframe can now be filtered down to just those journal entries without a balancing entry.

```Python
master_df[master_df['Research_Notes'] == 'No match']
```

There were 1,052 journal entries that did not have a balancing entry. Those entries would need to be audited to determine the root cause of the issue.

## Step 4: Visualize the results in PowerBI
