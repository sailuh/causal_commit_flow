# Replication Package for Causal Analysis

## 1. Foreword

The intent of this document is to provide instructions for the replication of the study described in “A Socio-technical Perspective on Software Vulnerabilities: A Causal Analysis” hereafter called the “Original Study.”

Given the non-determinism of search algorithms provided in our primary tool for Causal Discovery, Tetrad [see references in original study], we would not expect a replication to provide detailed results identical to those obtained for the original study; however, in the two years since the original study, the software stack underlying the original study results has itself changed, and thus even greater divergence might be expected. In particular, the Tetrad tool has gone through several versions from a pre-release of 6.8.0 (August 2020) used in the study to 7.1.2-2 (November 2022), the latest publicly-available release as we write this replication package. That two-year old pre-release is no longer publicly available and thus we ended up having to replicate our own study with Tetrad version 7.1.2-2 during which the main search algorithm used, Fast and Greedy Equivalent Search (FGES), has evolved somewhat as has the user interface to Tetrad and some of the features of the tool we employed (affecting screenshots created two years ago).

We thus had an opportunity to follow our own replication package but with a newer version of the main tool used. What we learned was this: the details will change (e.g., which specific social smells in the current time period affect which specific work-rate variables in the next time period) but the overall conclusions documented at the end of our study still hold, in particular, quoting from the study’s conclusions:

> “results: …social smells are indeed important factors that mediate significant project outcomes in terms of the incidence of and the effort associated with security vulnerabilities.”


## 2. Starting point

The starting point for causal discovery is this dataset (CSV file) produced by the issue-tracking and mailing-list scraping tools we were using; henceforth called the “scraping tool”: 

```diff
+ openssl_social_smells_timeline.csv
```

Here is a Data Dictionary explaining the variables in the dataset; both those variables (1) included in the initial dataset produced by the scraping tool; or (2) newly derived variables for input to causal discovery. While the second column, Definition, duplicates material already covered better elsewhere, note that for any time periods when there were zero commits, these definitions imply that many of the measures should be zero; and thus it is safe to impute zero for this particular source of missing values (what is called MD Reason 2 further below). (Don’t do any of these changes yet—we’ll address them below the table.)

| Variable (column)	| Definition |	How treated in preparing for and conducting causal discovery |
|-------------------|------------|----------------------------------------------------------------|
|cve_id (A)| The identifier of the CVE being remediated within the OpenSSL project, which maintains the Secure Socket Layer code critical to the Chromium Project; as described in our article. Figure 1 depicts overall timeline for a single CVE’s remediation. | In the original dataset, cve_id was represented as a string. Converted to an integer due to a limitation in what datatypes are supported by the software packages we were using (including Tetrad [we’ll say more about “Tetrad” later in this document]). <br /> <br /> Detail: the earliest cve is from year 2006, and so the “20” is redundant, so concatenate the last two digits of the year with the last four digits of the cve_id and convert into an integer. The specific Excel formula used was `=NUMBERVALUE(CONCAT(LEFT(RIGHT(cve_id,7),2),RIGHT(cve_id,4)))` without the quotes. (Further detail: create a new column B, select B2, paste the above formula into it, and replace both instances of “cve_id” with the cell address (here, A2) and then that cell formula is copied down to the bottom of the spreadsheet, the 2nd column selected, copied, and paste values before deleting the original cve_id column.)  The resulting conversion is an injection: records with the same unique cve_id are mapped to the same unique integer. <br /> <br />   How used: later, cve_id is converted into a set of dichotomous indicator variables, one per cve_id; and the variable itself was deleted. This conversion allows searching the entire dataset as a single CSV file (rather than one per cve_id). While making the application of causal discovery much simpler, there is a downside to this approach, in that it presumes that there is a single underlying causal model relating sociotechnical measures to outcomes within consecutive time periods for CVE remediation. However, the extent to which such a common-across-CVEs structural causal model exists is also concurrently tested by this inclusion of the dichotomous CVE indicators in the dataset being searched; and seeing whether and how many edges form between a CVE dichotomous indicator. So, in what follows, we pay particular attention to how many such edges form between particular CVEs and any sociotechnical or outcome variables. |
| commit_interval (activity_0: B) (activity_2: C)	| Concatenation of the hashes of the first and last instances of CVE-related activity (especially commits) during the time period. If there is only one commit within the 3-month period associated with a given CVE remediation, then the hashes will be identical. If there are none, then the field will just be blank. | Replaced with indicator variables: <br /> <br /> •	activity_0 (equals 1 if and only if zero hashes appear) and <br /> •	activity_2 (equals 1 if and only if two hashes appear) <br /> Later, both indicator variables were removed following their use in preparation of the dataset for causal discovery, given their redundancy. <br /> <br /> Detail: variable commit_interval is the concatenation of two 40-character hashes separated by a hyphen; or blank if there are no commits to form hashes from. Here are the Excel formulas used to derive the two activity indicator variables: <br /> <br /> •	activity_0 - used `=IF(commit_interval="",1,0)` <br /> •	activity_2 - used `=IF(AND(NOT(activity_0),LEFT(commit_interval,40)<> RIGHT(commit_interval,40)),1,0)` (Less detail this time: create 2 new columns, edit 2nd row cells with the above formula but replacing “activity_0” and “commit_interval” with the correct cell addresses, etc.)
| start_day (D) | 	First day of the time period |	Initially, renamed “Start” for brevity. <br /> <br /> Replaced with date ordinal value because our causal discovery tool doesn’t support the datetime datatype. The conversion was achieved by using an Excel formula: `=datevalue(left(start_day,10))`
| end_day | 	90 days after the start_day	| Deleted due to deterministic relationship with start_day |
|(3 Communication smells-related variables)	| | The next three variables (org_silo, missing_links, and radio_silence) represent “communication smells” and are derived in a similar way: the scraping tool we used builds two graphs: Graph A for who posts and who responds, Graph B for relationships between code units, and then maps Graph A into Graph B (developer D worked on code C). All three communication smells are counts of particular features of the connectivity among these graphs. (The main text of the article provides a more precise explanation for radio_silence [silence].) | 
| org_silo (E) | 	Count of “the number of single collaborations between different developers in which at least one of them does not participate in the [appropriate] communication channel” for the current time period	| No change to variable name | 
| missing_links (F)	| Count of instances where two people were working on same file within the time period, but there’s no evidence of communication from looking at issue tracker and mailing list. | Renamed “mis_link” for brevity | 
| radio_silence (G)	| Count of delays due to lack of direct communication between two sub-communities engaged in a CVE’s remediation. | Renamed “silence” for brevity | 
| primma_donna |	(Not yet fully operationalized at the time of our analysis and thus dropped from further consideration.)	| Deleted | 
| st_congruence (H) |	Short for “Socio-technical congruence.” A single continuous measure of overall similarity between collaboration graph and the file graph. Scale from 0..1 (1 means perfect congruence). |	Renamed “congruence” for brevity | 
| Communicability (I)	A continuous measure of the diffusion of information. How likely is it that an architectural decision is known by the set of developers that need to know about it. |  Scale from 0..1 (1 means perfect communicability). Is the inverse of incommunicability. |	Renamed “communicate” for brevity | 
| code_only_devs (J) |	Count of coders who made commits to the implicated CVE-related files during the time period. (For the CVE identified in column A.) |	Renamed “code_dev” for brevity | 
| code_files (K)	| Count of files involved in CVE’s remediation. |	Renamed “file” for brevity | 
| ml_only_devs (L)	| Count of people who sent at least one item to mailing list during time period. Not necessarily on topic. |	Renamed “mail_dev” for brevity | 
| ml_threads (M) | Count of email threads in ML during that time window on any topic. | Renamed “thread” for brevity | 
| n_commits (N) |	Count of commits (there can be multiple commits for multiple files) during time period | 	Renamed “commit” for brevity |
| sum_churn (O)	| Count of Lines of Code (LOC) committed to the code file counted in code_files; summed over the commits in n_commits.	| Renamed “churn” for brevity | 
| (next time period-related variables) | | DON’T DO THIS STEP YET. This step is best done after addressing missing data. Wait until the subsection “Appending Variables Representing the Next Time Period” to accomplish this step. | <br /> <br /> The final few variables in this table are a repeat of the 11 variables assigned columns E-O above but instead pertain to the next time period (90 days) in a CVE’s remediation; and their names match the 11 variables above but have a suffix “2” appended to their names. <br /> <br /> These 11 new variables represent what happened in the next time period that could (in principle) be partly attributed to what happened in the current time period for the CVE identified in column A, or to idiosyncrasies of the CVE itself (e.g., whose resolution requires a skillset not present in coders involved in its remediation). <br /> <br /> In terms of manipulating the dataset, each of these next-time period-related variables is derived in this way: make a copy of the corresponding original variable and shift all of its values up by one row. Of course, the top value of each original-variable column is simply unused; and the bottom value of the new column is blank.  <br /> <br /> Finally, for each CVE id, the record representing the last time period for a CVE’s remediation is then deleted. Of course, it had gotten either the values of the first time period of the next CVE, or blanks, if it was the last row of the entire dataset. For more detail on how to accomplish this step, see the subsection below titled “Appending Variables Representing the Next Time Period.” <br /> <br /> The variables appear in the same order as the corresponding original 11 variables above: org_silo2, mis_link2, silence2, congruence2, communicate2, code_dev2, file2, mail_dev2, thread2, commit2, churn2. | 
| org_silo2 (P) |	Same count as “org_silo” but applied to the next time period |
| mis_link2 (Q)	| Same count as “mis_link” but applied to the next time period |
| silence2 (R)	| Same count as “silence” but applied to the next time period |
| congruence2 (S)	| Same 0..1 measure as “congruence” but applied to the next time period |
| communicate2 (T) | Same 0..1 measure as “communicate” but applied to the next time period |
| code_dev2 (U)	| Same count as “code_dev” but applied to the next time period |
| file2 (V)	| Same count as “file” but applied to the next time period |
| mail_dev2 (W)	| Same count as “mail_dev” but applied to the next time period |
| thread2 (X)	| Same count as “thread” but applied to the next time period |
| commit2 (Y)	| Same count as “commit” but applied to the next time period |
| churn2 (Z)	| Same count as “churn” but applied to the next time period |

Note the details of how the conversions mentioned in the third column above are achieved won’t matter as long as certain properties of the variables are retained: in particular, the cve_id conversion need only be an injection (one-to-one); the conversion of start-day to an integer can be any standard datetime conversion to integer (as long as it is linear and monotonic; as many such conversions are). 
Now we implement the data conversions and renaming of variables implied in the table above. The results of this initial conversion appear here: 

```diff
+ openssl_social_smells_timeline..renameVariables.csv
```

Note that we keep the original filename but append a brief phrase (“...renameVariables”) to indicate what action we just took on the dataset. The original dataset and this name-converted dataset both have 6697 rows.

Also, we assume that all the rows dealing with the same CVE are contiguous to each other and also that for each CVE, rows appear in time period order (for the same CVE). It’s straightforward to check these conditions in Microsoft Excel, for example by adding a formula(s) in a new column(s) to check this out. (We’ll give a couple examples later of similar manipulations in Excel which have additional explanation.)

## 3. Handling missing data (MD) 

When we examine the CSV file identified above, we find a spreadsheet of 6697 rows (not counting the header row) and 16 columns; however, many of the cells in the spreadsheet are blank, that is, they have no value assigned to them—this is referred to as “missing data” (MD). Further examination reveals there is a pattern to the missingness. We can specify that pattern through a partially-manual examination of the spreadsheet using Microsoft® Excel®, which takes time, but it can be done, or we can use a tool, such as Mplus® [1], to automate that analysis more systematically and completely. Because the Mplus tool is a commercial tool that not everyone will have relatively easy access to, we briefly describe both approaches to characterizing the MD in the dataset.

**Characterizing the MD in a dataset using Excel.**  There are multiple approaches to analyzing the pattern of MD for a dataset opened with Microsoft Excel; we briefly describe one way here. Note that when a file has been opened using Excel, the bottom bar (called the “Status Bar”) displays summary information about the content of that column. (What information is displayed in that Status Bar can be set in various ways within the tool including simply bringing up the Context menu by right-clicking (on a Windows PC) on the Status Bar and selecting the information you desire be displayed.) By selecting a column in Excel, we can see a count of how many cells have some kind of content (not MD). For example, selecting the first column “cve_id”, we see from the Status Bar that there are 6698 cells with content (including the header), that is, there is no MD in the first column. Similarly, when we examine the second column “commit_interval” we see from the Status Bar that there are 4498 non-blank cells. Subtracting this from 6698 (6697 plus the header row) works out to 2200 cells that are blank. And we can likewise do this examination for the other 14 columns. There is another useful feature of Microsoft Excel and that is the column filter, which can be accessed from the ribbon on top, selecting “Data” and then “Filter.” In what follows, we assume the reader is either familiar with these Excel features, and Filter in particular, or is willing to come up to speed on these features by googling for a description of how this can be accomplished. For this dataset, we can pursue a “divide and conquer” approach to characterizing MD: systematically, one column at a time (until we’re done), filter on the column twice: once on the non-MD, and then on the MD, each time characterizing which columns (of the filtered spreadsheet) have cells with MD. In this way, we can discern if there is a systematic pattern to the MD. For example, if we select the second column “commit_interval” we can set up a filter on just the non-MD (by scrolling to the bottom of the filter pull-down menu and unclicking “(Blanks)”), we find that the first column with any MD is the fifth column “org_silo”, which has 4498-3591=907 blank cells. Proceeding carefully in this way, a systematic pattern of missingness then reveals itself, which matches the pattern uncovered in a more automated fashion using Mplus. (If handling MD manually with Excel, you don’t need to read the next subsection on using Mplus; instead, skip down to the subsection that begins with the words “After determining the MD patterns in the dataset, resolve the MD appropriately”.)

**Characterizing the MD in a dataset using Mplus.** The Mplus tool’s forte is general linear modeling; however, it also provides additional functionality such as characterizing where in the dataset there is MD. An analysis of the dataset using Mplus returned a summary that looked something like the following table. (Only the variable names have been changed: Mplus only recognizes the first eight characters of a variable name, and so, the variable names when specified in Mplus were shortened; however, to avoid confusing the reader the variable names appearing in the resulting table were then restored back to match those used in the table above.)

```
SUMMARY OF DATA

     Number of missing data patterns             4


SUMMARY OF MISSING DATA PATTERNS


     MISSING DATA PATTERNS (x = not missing)

           		1  	2 	3 	4
 cve_id   		x  	x  	x  	x
 activity_0		x  	x  	x  	x
 activity_2		x  	x  	x  	x
 Start     		x  	x  	x  	x
 End       		x  	x  	x  	x
 org_silo		x
 mis_link		x
 silence   		x
 congruence		x
 communicate		x
 code_dev  		x  	x
 file      		x  	x
 mail_dev 		x     		x
 thread    		x     		x
 commit    		x  	x
 churn     		x  	x

     MISSING DATA PATTERN FREQUENCIES

    Pattern   Frequency     Pattern   Frequency     Pattern   Frequency
    1        3590           	3        1973
    2         907           	4         227
```


Note that summing the number of cases across the four Missing Data (MD) patterns, we obtain 6697 cases, which matches the number of rows in the dataset. 

**After determining the MD patterns in the dataset, resolve the MD appropriately.** We begin the exposition with a brief account of what took place after determining the patterns of MD in the dataset and then follow this with the actions required to resolve the MD.

From discussions with someone knowledgeable about the opensource project as well as the developer of the website scraping tool, two reasons emerged as to why data should or could be missing from the dataset:

1. The mailing log in use until about 2000-2001 was missing due to failure to preserve the original mailing log during the transition to a new mailing list manager for the project in 2001. 

2. For some time periods, there are no commits. We can easily identify such cases because variable “commit” is blank (and also activity_0 is 1 precisely when commit is blank), which prevents the scraping tool from building the file graph used for computing several measures, resulting in those variables also being blank. 

A careful analysis of the dataset (using Mplus) concluded that the above two reasons explained all MD in the dataset. That is, there was no MD that we couldn’t ascribe to one of the above two reasons. 

The Mplus table helps clarify the nature of the missing data. (In this Replicability Package, we chose not to include the Mplus input script we developed for Mplus use, in order to not have to explain a second set of aliases for the variable names (there are constraints that need to be adhered to regarding what variable names are legal in an Mplus script), and because we don’t consider it critical to replication, only clarifying the general nature of the missing data returned by the scraping tool.)

With respect to MD Reason 1 (specified above), about 17% of the rows (MD Patterns 2 and 4 in the Mplus table above) suffered missing values due to the loss of the mailing log (907 + 227 cases). Developers were presumably communicating during the time period but there is no data on such communication.


 * Specifically, this loss of the mailing log prevents computing variables “mail_dev” and “thread” directly; and variables “org_silo” through “communicate” indirectly (if mail activity is lost, we can’t compute one of the graphs used in computing these variables). In the rows affected by the loss of the mailing log, the scraping tool should not impute a value for all these missing values; and it does not—they are indeed blank. 

 * The authors felt that the conditions leading to the loss of the mailing log should have no noticeable causal effect on any of the other variables in the dataset (other than time-related variables such as “Start” and “End”). In other words, the authors deemed that data missing due to MD Reason 1 can be considered Missing Completely at Random (MCAR) [7]. Therefore, Listwise Deletion [7], that is, deleting all rows that had missing data that was MCAR, other than increasing the estimate of standard error, induces no bias. While deleting 17% of the rows is a pretty significant deletion, the authors felt that the retained 83% of the rows was still sufficient to identifying major direct causal relationships. Also, from an explainability and understandability perspective, deleting 17% of the rows seemed preferable to employing pseudo-random number generation to impute values. Finally, choosing deletion rather than imputation also helps ensure the results can be more easily replicated. (The authors did pursue Full Information Maximum Likelihood-based imputation [7] initially, but after such considerations, chose not to use the results but to instead pursue Causal Discovery on the dataset resulting from Listwise Deletion.)

With respect to MD Reason 2, when there is no commit activity within a time period, any measures of features (counts) related to commits should all be 0. While the scraping tool left blanks for their values, the value 0 should be imputed for such missing values.


 * Again, the authors considered each variable one by one to verify that this was sensible. Specifically, having absolutely no commits in a time period prevents building the graphs needed for computing variables “org_silo” through “communicate,” and when there are no commits, the scraping tool also leaves “commit” and “churn” blank. All of these commit-related variables should be zero.
 * The activity_0 variable is equal to 1 specifically in these 2200 (1973 + 227) cases.
 * Note that even though there might have been no commit activity in a time period, there might still be mailing list activity and thus measures associated with any activity in the mailing list (mail_dev and thread) could still be computed, and thus non-blank, that is, after we’ve deleted all rows associated with a lost mailing log (MD Reason 1).

Summarizing, and referring to the table generated by Mplus, we can say this about the dataset:

 * Number of cases with no MD at all (corresponds to MD Pattern 1 in the Mplus table): 3590
 * Number of cases with MD due to MD Reason 1 only (corresponds to MD Pattern 2 in the Mplus table): 907
 * Number of cases with MD due to MD Reason 2 only (corresponds to MD Pattern 3 in the Mplus table): 1973
 * Number of cases with MD due to MD Reasons 1-2 (corresponds to MD Pattern 4 in the Mplus table): 227

Thus, with the above changes to the dataset, of deleting about 17% (907 + 227) of the rows affected by the loss of an old mailing log, and imputing zeros for all remaining missing values, we thus have a dataset of 5563 cases) with no MD and can thus continue onward to the next steps in preparing the dataset for applying causal discovery.

**Detail for performing Listwise Deletion:** In Excel, place a Data > Filter on column “mail_dev” and in the filter select only “(Blanks)” from the dropdown menu. The number of such rows (after the header row) should be 1134 rows. Note that this seems right because that’s the total number of cases covered by Mplus MD Patterns 2 and 4. Then, selecting all those rows (but not the header row), delete them. The result should 5563 rows.

Detail for imputing zeros: within Excel, we performed a global selection of blank cells within the entire dataset (following Listwise Deletion, and thus the only blank cells left now are due to there being no commits for that time quarter), replacing the (blank) content of all such cells by 0. In further detail: we simply invoked the Replace command configured with these options: “Find what:” empty string, “Replace with:” 0, click on Options >> to check “Match entire cell contents”, etc. 17757 replacements were thus made. Note that this is 9*1973, the latter number is the number of cases covered by Mplus MD Pattern 3, which is the MD pattern corresponding to blank cells arising only due to there being zero commits for the time period. The 9 blank cells for each of these rows correspond to the variables indicated in the Mplus table above showing the missingness for MD Pattern 3.

Here’s the file we obtained by resolving the MD as described above:

```diff
+ openssl_social_smells_timeline..renameVariables..resolveMD.csv
```













