# Cost Calculation
## Python script: cost_calculation.py

This is a python script which will generate the excel or json output with the Summary and Cost data as well as breaking down the worker/container/controller data.

It is a command line tool which parses the generated log files to calculate the actual cost and estimate cost savings based on on-demand pricing.

The script parses the data and collects information on the controller, workers and containers, then creates an output file. There are three types of output formats it generates, depending on what modules the environment has loaded or is willing to load.  There is an excel worksheet format, an html format and a json format.

1.	Python set up:

	The following python modules are necessary to run this script.

	Mandatory:

	- python3:
		- yum install python3
	- dateutil : module needed for handling timezones properly
		- python3 –m pip install python-dateutil

	Optional depending on output types:

	- To generate excel worksheet output
		- Xlsxwriter: 	pip3 install xlsxwriter
		- pandas: 	pip3 install pandas
	- To generate html format output
		- Json2html: 	pip3 install json2html

	python 3 or python 3.8 may be used

2.	The usage is as follows:

	Run a script on log files to calculate cost and estimate savings

    Options:

	**-l, --logs          	Location of log files to parse: must be directory   ex: logs/  (mandatory)**

	-s, --start        	 Start day  (in date format: 2022-03-17), (default to 30 days ago)

	-e, --end           	End day  (in date format: 2022-03-17), (default end of current day)

	-n, --num_days      	Number of days: start with today or number of days since, and end with end of
				day today (overrides any -s or -e entry) (default is this parameter is not used)

	-d, --discount      	Percent discount of on-demand instances to use for analysis, (default = 0%)

	-o, --output        	Enter type of output format: json, html, xlsx  (default is all 3 if modules are 							installed)

	-c, --container     	Enter container ID - will display only information associated with this container 							ID, (default = '', full output generated)

	-f, --containerLife 	Enter container ID - will display only information associated with this container ID in a 							'Lifetime' format, (default = '', full output generated)

   	-j, --jobid	 	Enter job ID - will display only information associated with this job ID in a 									'Lifetime' format, (default = '', full output generated)


Examples:

The parameter –l (--logs) is mandatory:  the log file path

python3.8 \<path\>cost_calculation.py -l \<log file path\>
```
python3.8 /var/log/cost_calculation.py -l /var/log/xspot/latest
```

Optional: to specify start date:
```
python3.8 <path>cost_calculation.py -l <log file path> -s 2023-03-17
```

Optional: to specify on xlsx output:
```
python3 cost_calculation.py -l <log file path> -s 2023-03-31 –o xlsx
```


## Cost Calculation Script Error Reference

| Installation errors:  | *specific packages are not available* |
| ----------- | ----------- |
| Python version must be 3 or greater   |   |
| No *\<module\>* found  | Timezone from datetime  |
|   | Json2html |
|   | Xlsxwrite |
|   | Pandas |

| Usage errors:  |  |
| ----------- | ----------- |
| *option -x not recognized* <br>   *\<Usage text should follow\>* | An incorrect option flag was entered |
| *Entry error:  -l: location of log files parameter is mandatory* | The –l (--logs) option is missing <br> This is a mandatory option |
| *Incorrect entry: --logs must be a directory* | The logs path must be a directory |
| *Incorrect entry: Start time (in 2022-03-17 format)* | The incorrect date format was used <br> Ex: -s 2023-04-01 <br> Default: the last 30 days are used |
| *Incorrect entry: End time (in 2022-03-17 format)* | The incorrect date format was used <br> Ex: -e 2023-04-01 <br> Default: end time is end of current day |
| *Incorrect Entry: --number of days must be integer only* | The value for the day range must be an integer <br>  Ex:  -n 10 <br> Date range will be the last 10 days <br> Default : it is not used  |
| *Incorrect entry: output file must be json, or html, or xlsx* | The output file type is entered incorrectly <br> Ex: -o xlsx <br> Default: all files are generated |
| *Incorrect Entry: --discount must be integer only* | The value for the discount must be an integer and is the percent discount (30% => -d 30) <br> Default is 0% |

| Log file parsing (time) errors:  |  |
| ----------- | ----------- |
| *No valid log files specified:* | Time stamp not found in first 5 and last 5 lines of file |
|   | Corrupt log file  |
|   | No write access to location of xspot log file(s)  |


| General errors:  |  |
| ----------- | ----------- |
| *Invalid instance type:* | ‘FinalizeWorkerMetadata’ not found – resulting in no instance type found |
| *json not matched:* | Terminating Worker Instance :  <br> misformed line in log file – no json data |
| *Error in file: No EndTime found for Controller endtime (or lastcount)* | Error while searching for controller end time – which is the date stamp in a line at the end of the file |
| *exception2:  list index out of range* | Controller data incorrect : endtime |
| *no instance type for:  \<worker number\>* | The instance type to be used for cost was unable to be found from worker log data |
| *Pricing file not found:<br>/etc/xspot/aws-pricing/.json* | The DefaultRegion was not found :  <br> which is in the Scheduler configurations line |
| *Pricing file not found:  /etc/xspot/aws-pricing/\<region\>.json* | The pricing files are not accessible or were not created |
| *Invalid instance type: \<instance type\>* | The pricing file was not found, or does not have the matching instance type |


| Running for one container only data (-c option):  |  |
| ----------- | ----------- |
| *The Container ID entered is not unique.* | More characters in the container ID string are needed |
| *The Container ID entered was not found.* | The entered Container ID did not match any in the log file |



