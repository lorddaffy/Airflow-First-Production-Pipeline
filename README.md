# Airflow-Branching-Templates-Pipelines
**Applying Branching, Templates and Pipelines with Airflow**
____________________________

> After learning about the power of conditional logic within Airflow, you wish to test out the BranchPythonOperator. You'd like to run a different code path if the current execution date represents a new year (ie, 2020 vs 2019); Your current task is to implement the BranchPythonOperator.
```
# Import the DAG object
# Import the BashOperator
from airflow.models import DAG
from airflow.operators.python_operator import BranchPythonOperator
from airflow.operators.python_operator import PythonOperator
from datetime import datetime

# Create a function to determine if years are different
def year_check(**kwargs):
    current_year = int(kwargs['ds_nodash'][0:4])
    previous_year = int(kwargs['prev_ds_nodash'][0:4])
    if current_year == previous_year:
        return 'current_year_task'
    else:
        return 'new_year_task'

# Define the BranchPythonOperator
branch_task = BranchPythonOperator(task_id='branch_task', dag=branch_dag,python_callable=year_check, provide_context=True)
```
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

## First Production Pipeline

> From what you've learned about the process, you know that there is sales data that will be uploaded to the system. Once the data is uploaded, a new file should be created to kick off the full processing.

**Instructions:**
 - start_date= '2022,3,15'
 - The filepath='/home/repl/workspace/startprocess.txt', you should use this path fro sensor
 - Poke_interval=5
 - timeout=15
 - task: delete all .tmp files in '/home/repl/'
 - run sensor upstream bash_task
 - run python_task (last task) downstream bash_task
 - Use the process_data function

```
# Imports
from airflow.models import DAG
from airflow.contrib.sensors.file_sensor import FileSensor
from airflow.operators. bash_operator import BashOperator
from airflow.operators. python_operator import PythonOperator
from datetime import date, datetime

def process_data(**context):
  file = open('/home/repl/workspace/processed_data.tmp', 'w')
  file.write(f'Data processed on {date.today()}')
  file.close()

    
dag = DAG(dag_id='etl_update', default_args={'start_date': datetime(2022,3,15)})

sensor = FileSensor(task_id='sense_file', 
                    filepath='/home/repl/workspace/startprocess.txt',
                    poke_interval=5,
                    timeout=15,
                    dag=dag)

bash_task = BashOperator(task_id='cleanup_tempfiles', 
                         bash_command='rm -f /home/repl/*.tmp',
                         dag=dag)

python_task = PythonOperator(task_id='run_processing', 
                             python_callable=process_data,
                             dag=dag)

sensor >> bash_task >> python_task
```
__________________________________________________________
### Some Modifecations

> Continuing on your last workflow, you'd like to add some additional functionality, specifically adding some SLAs to the code and modifying the sensor components.

**Instructions:**
 - start_date= '2022,3,15'
 - The filepath='/home/repl/workspace/startprocess.txt', you should use this path fro sensor
 - task: delete all .tmp files in '/home/repl/'
 - run sensor upstream bash_task
 - run python_task (last task) downstream bash_task
 - Use the process_data function
 - Add an SLA of 90 minutes to the DAG.
 - Update the FileSensor object to check for files every 45 seconds.
 - Update the FileSensor object and remove the timeout
 - Modify the python_task to send Airflow variables to the callable.
```
# Imports
from airflow.models import DAG
from airflow.contrib.sensors.file_sensor import FileSensor
from airflow.operators.bash_operator import BashOperator
from airflow.operators.python_operator import PythonOperator
from datetime import timedelta, datetime, date


def process_data(**kwargs):

    file = open("/home/repl/workspace/processed_data-" + kwargs['ds'] + ".tmp", "w")
    
    file.write(f"Data processed on {date.today()}")
    
    file.close()

# Update the default arguments and apply them to the DAG
default_args = {
  'start_date': datetime(2022,3,15),
  'sla':timedelta(minutes=90)
}

dag = DAG(dag_id='etl_update', default_args=default_args)

sensor = FileSensor(task_id='sense_file', 
                    filepath='/home/repl/workspace/startprocess.txt',
                    poke_interval=45,
                    dag=dag)

bash_task = BashOperator(task_id='cleanup_tempfiles', 
                         bash_command='rm -f /home/repl/*.tmp',
                         dag=dag)

python_task = PythonOperator(task_id='run_processing', 
                             python_callable=process_data,
                             provide_context=True,
                             dag=dag)

sensor >> bash_task >> python_task

```
__________________________________________________________
### Final Modifecations

> To finish up your workflow, your manager asks that you add a conditional logic check to send a sales report via email, only if the day is a weekday. Otherwise, no email should be sent. In addition, the email task should be templated to include the date and a project name in the content.

**Instructions:**

 - start_date= '2022,3,15'
 - The filepath='/home/repl/workspace/startprocess.txt', you should use this path fro sensor 
 - task: delete all .tmp files in '/home/repl/'
 - run sensor upstream bash_task
 - run python_task (last task) downstream bash_task
 - Use the process_data function
 - Add an SLA of 90 minutes to the DAG.
 - Update the FileSensor object to check for files every 45 seconds.
 - Update the FileSensor object and remove the timeout
 - Modify the python_task to send Airflow variables to the callable.
 - Add email_operator with this email [to='sales@mycompany.com]
 - Add dummy_operator for no_mail_task
 - Use the written functions `process_data` , `check_weekend` and the templated command `email_subject` 
```
# Imports
from airflow.models import DAG
from airflow.contrib.sensors.file_sensor import FileSensor
from airflow.operators.bash_operator import BashOperator
from airflow.operators.python_operator import PythonOperator
from airflow.operators.python_operator import BranchPythonOperator
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators.email_operator import EmailOperator
from datetime import datetime, timedelta, date


def process_data(**context):
  file = open('/home/repl/workspace/processed_data.tmp', 'w')
  file.write(f'Data processed on {date.today()}')
  file.close()
  
# define the default arguments and apply them to the DAG.
default_args = {
  'start_date': datetime(2022,3,15),
  'sla': timedelta(minutes=90)
}
    
dag = DAG(dag_id='etl_update', default_args=default_args)

sensor = FileSensor(task_id='sense_file', 
                    filepath='/home/repl/workspace/startprocess.txt',
                    poke_interval=45,
                    dag=dag)

bash_task = BashOperator(task_id='cleanup_tempfiles', 
                         bash_command='rm -f /home/repl/*.tmp',
                         dag=dag)

python_task = PythonOperator(task_id='run_processing', 
                             python_callable=process_data,
                             provide_context=True,
                             dag=dag)


email_subject="""
  Email report for {{ params.department }} on {{ ds_nodash }}
"""


email_report_task = EmailOperator(task_id='email_report_task',
                                  to='sales@mycompany.com',
                                  subject=email_subject,
                                  html_content='',
                                  params={'department': 'Data subscription services'},
                                  dag=dag)


no_email_task = DummyOperator(task_id='no_email_task', dag=dag)


def check_weekend(**kwargs):
    dt = datetime.strptime(kwargs['execution_date'],"%Y-%m-%d")
    # If dt.weekday() is 0-4, it's Monday - Friday. If 5 or 6, it's Sat / Sun.
    if (dt.weekday() < 5):
        return 'email_report_task'
    else:
        return 'no_email_task'
    
    
branch_task = BranchPythonOperator(task_id='check_if_weekend',
                                   provide_context=True,
                                   python_callable=check_weekend,
                                   dag=dag)

    
sensor >> bash_task >> python_task

python_task >> branch_task >> [email_report_task, no_email_task]

```
