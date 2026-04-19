# Massachusetts_records_request_determinations_2016-2026
Searchable database of more than 28,000 determinations issued by the Massachusetts Supervisor of Public Records office since 2016 

This repo hosts a full scrape of https://www.sec.state.ma.us/appealsweb/appealsstatus.aspx including full text of all determinations issued between 2016 and 2026. 2026 appeals go through early April. I'm working on setting it up to update as new ones come in. 

Data is stored in an SQLite database. Tables `appeals` and `case_details` are scraped directly from the Secretary of Commonwealth's website. `determinations` contains the full text of the Determination pdfs, as well as my attempts to parse them. Still working on making sure I've done that right, so caveat emptor for right now until I finish validating I've done it right, but full text search will work just fine. 


See data_dictionary.md for more details 

Download database from releases [https://github.com/JonGerhardson/Massachusetts_records_request_determinations_2016-2026/releases](https://github.com/JonGerhardson/Massachusetts_records_request_determinations_2016-2026/releases)

CC0 but if you publish something based on this dataset and don't give me credit you'll have 7 years bad luck 
