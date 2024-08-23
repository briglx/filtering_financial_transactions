# Filtering Financial Transactions

This is an example project to filter out any transactions that match the filter criteria for memo terms or specific transaction numbers for an exported financial data file.

Example Data:
[sample_data.csv](./sample_data.csv)

The goal is to filter out any transactions that meet certain criteria. For example:

* Filter out any transactions that are transfers
* Filter out specific transactions by transaction number
* Filter out transactions that have a specific term in the memo


## Quick Start

1. Clone the repository
2. Open the PowerBI Template `filter_financial_transactions.pbit`
3. Provide the data file path to the `/path/to/project/sample_data.csv`
4. Enter filter terms for the memo field `Transfer,XPMPX`
5. Enter filter transaction numbers `1074432,911993`

## Details

I tend to use parameters for my PowerBI Reports. This allows me to upload a powerbi template and point to the data source.

Parameters:
* DataFilePath: The path to the data file
* FilterMemoTerms: A comma separated list of terms to filter out
* FilterTransactionNumbers: A comma separated list of transaction numbers to filter out

```m
let

    // Load the filter parameter values
    TermList = List.Transform(Text.Split(FilterMemoTerms, ","), each Text.Lower(Text.Trim(_))), 
    ExcludedTransactionNumberList = List.Transform(Text.Split(FilterTransactionNumbers, ","), each Number.FromText((Text.Trim(_)))), 

    // Custom function to filter out excluded transactions
    filter_excluded = (record) =>
        let

            // Check if Memo has any matching terms
            is_term_match = List.AnyTrue(
                List.Transform(TermList, each 
                    Text.Contains(Text.Lower(record[Memo]), _)
                )
            ),

            // Check for matching Transaction Numbers
            is_transaction_match = List.AnyTrue(
                List.Transform(ExcludedTransactionNumberList, each 
                    record[Transaction Number] =  _
                )
            ),
            result = if is_term_match or is_transaction_match
                then 0 
                else -record[Amount Debit]
        in result,
        
    // Standard Import steps
    Source = Csv.Document(File.Contents(DataFilePath),[Delimiter=",", Columns=11, Encoding=1252, QuoteStyle=QuoteStyle.None]),
    #"Removed Top Rows" = Table.Skip(Source,3),
    #"Promoted Headers" = Table.PromoteHeaders(#"Removed Top Rows", [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {
        {"Transaction Number", Int64.Type}, 
        {"Date", type date}, 
        {"Description", type text}, 
        {"Memo", type text}, 
        {"Amount Debit", type number}, 
        {"Amount Credit", type number}, 
        {"Balance", type number}, 
        {"Check Number", Int64.Type}, 
        {"Fees ", Int64.Type}, 
        {"Principal ", type number}, 
        {"Interest", Int64.Type}
    }),

    // Add custom column to exclude transactions
    #"Added Custom" = Table.AddColumn(#"Changed Type", "Amount Debit Filtered", filter_excluded, Currency.Type)

in
    #"Added Custom"
```

## Thoughts

I initially tried to have nested `each` when filtering out the terms. This didn't work because the magic underscore `_` was not available in the nested `each`.This lead me to create a custom function to handle the filtering.

Also, Instead of adding a new column, I could have filtered out the rows. I took the column route for no particular reason.

# References
- https://blog.crossjoin.co.uk/2015/05/11/nested-calculations-in-power-query/
- https://learn.microsoft.com/en-us/powerquery-m/understanding-power-query-m-functions
- https://excelguru.ca/each-keyword-power-query/
