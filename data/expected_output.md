# output/consolidated_transactions.csv
transaction_id,date,branch,customer_id,product,quantity,unit_price,line_total,payment_method,payment_ref,source_file
<TXN-id>,<YYYY-MM-DD>,<text>,<CUST-id>,<text>,<int>,<int>,<int>,<text>,<text or blank>,<file name>
...

# output/rejected_transactions.csv
source_file,row_number,transaction_id,reasons,<original columns...>
<file name>,<int>,<TXN-id or blank>,<reason 1; reason 2>,...
...

# Console / output/summary.txt
=== Processing summary ===
Files discovered:        <n>
Files processed:         <n>
Files with column issues:<n>  (<file>: missing <columns>)
Rows read:               <n>
Rows accepted:           <n>
Rows rejected:           <n>
  - missing values:      <n>
  - invalid numbers:     <n>
  - duplicate IDs:       <n>
Total revenue (KES):     <amount>
Run finished at:         <ISO timestamp>
