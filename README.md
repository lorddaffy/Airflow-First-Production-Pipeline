# Airflow-Temaplates-Pipelines
 **Applying templates and pipelines with Airflow**
 
 __________________________________________________________
 
 > Create the EmailOperator task using the template string for the html_content; Create a Python string that represents the email content you wish to send. Use the substitutions for the current date string (with dashes) and a variable called username; provide a string of a universally unique identifier as the subject field; Assign the params dictionary as appropriate with the username of testemailuser.
```
# Import the DAG object
# Import the BashOperator
from airflow.models import DAG
from airflow.operators.email_operator import EmailOperator
from datetime import datetime

# Create the string representing the html email content
html_email_str = """
Date: {{ ds }}
Username: {{ params.username }}
"""

email_dag = DAG('template_email_test',
                default_args={'start_date': datetime(2020, 4, 15)},
                schedule_interval='@weekly')
                
email_task = EmailOperator(task_id='email_task',
                           to='testuser@datacamp.com',
                           subject="{{ macros.uuid.uuid4() }}",
                           html_content=html_email_str,
                           params={'username': 'testemailuser'},
                           dag=email_dag)

```
 __________________________________________________________
 
 > Modify the cleandata.sh script to take two arguments - the date in YYYYMMDD format, and a file name passed to the cleandata.sh script, The filename for the first Operator is salesdata.txt and for the seconds Operator will be supportdata.txt
```
# Import the DAG object
# Import the BashOperator
from airflow.models import DAG
from airflow.operators.bash_operator import BashOperator
from datetime import datetime

default_args = {
  'start_date': datetime(2020, 4, 15),
}

cleandata_dag = DAG('cleandata',
                    default_args=default_args,
                    schedule_interval='@daily')

# Modify the templated command to handle a
# second argument called filename.
templated_command = """
  bash cleandata.sh {{ ds_nodash }} {{ params.filename }}
"""

# Modify clean_task to pass the new argument
clean_task = BashOperator(task_id='cleandata_task',
                          bash_command=templated_command,
                          params={'filename': 'salesdata.txt'},
                          dag=cleandata_dag)

# Create a new BashOperator clean_task2
clean_task2 = BashOperator(task_id='cleandata_task2',
                          bash_command=templated_command,
                          params={'filename': 'supportdata.txt'},
                          dag=cleandata_dag)
                           
# Set the operator dependencies
clean_task  >> clean_task2

```
__________________________________________________________

