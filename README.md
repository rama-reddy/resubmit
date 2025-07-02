# SLURM Resubmit Mode Documentation

## Overview

The SLURM resubmit mode provides automatic fault tolerance for job execution by detecting and retrying jobs that fail due to known transient infrastructure issues. This feature is primarily designed for unittest scenarios and operates by monitoring job output files, detecting specific error patterns, and automatically resubmitting failed jobs with appropriate configuration adjustments.

## Architecture

The resubmit functionality is implemented across two main components:
- **ssm_driver.py**: `MultiSlurmSubmit` class - orchestrates resubmission logic
- **await_slurm_run.py**: `AwaitSlurmRun` class - monitors and integrates resubmitted jobs

## Prerequisites for Resubmit Mode

### Eligibility Check: `resubmit_possible()` (ssm_driver.py:599-623)

Before enabling resubmit mode, the system validates several conditions:

```python
def resubmit_possible(self):
    """Function to check if resubmission is possible"""
    
    if self.trigger_source != 'unittests':
        return False
    
    if len(self.slurm_inis_info) > 1:
        return False
    
    # Check for incompatible SLURM configurations
    patterns = [
        r'export TSPLIT_SUBMIT=.*',
        r'export MODE=.*',
        r'export SLURM_MULTI_SIM=.*',
        r'export NUM_SPLITS=.*'
    ]
```

**Requirements:**
1. Must be triggered by `unittests` source
2. Only single SLURM configuration allowed
3. No advanced SLURM features (TSPLIT, MODE, MULTI_SIM, NUM_SPLITS)

## Known Error Detection

### Error Patterns: `known_errors` (ssm_driver.py:100-105)

The system monitors for specific error patterns that indicate transient failures:

```python
self.known_errors = [
    "No space left on device",
    "Killed",
    "Segmentation fault", 
    "fcntl.flock(_pid_file, fcntl.LOCK_EX | fcntl.LOCK_NB)",
]
```

### Error Detection Logic: `file_contains_known_error()` (ssm_driver.py:491-498)

```python
def file_contains_known_error(self, file):
    with open(file) as f:
        content = f.readlines()
    for error in self.known_errors:
        for line in content:
            if error in line:
                return True
    return False
```

## Resubmission Workflow

### 1. Job Monitoring Setup

#### Thread Initialization: `start_resubmit_failed_dates_task()` (ssm_driver.py:580-597)

```python
def start_resubmit_failed_dates_task(self, stop=False):
    """Function to start threads to check job_id.err file and re-submit the failed jobs of known errors"""
    
    self.print_and_log("Starting check jobid err futures")
    for date in self.run_on_dates:
        t = self.resubmit_failed_date.submit(date=date)
        self.jobid_err_futures.append(t)
```

**Activation Condition** (ssm_driver.py:1478-1481):
```python
self.possible_to_resubmit = self.resubmit_possible()
if self.possible_to_resubmit and not os.path.exists(f"{self.base_dir}/resubmit/resubmission_completed.ready"):
    self.start_resubmit_failed_dates_task()
```

### 2. Failed Job Detection

#### Job Output Analysis: `get_jobid_error_file()` (ssm_driver.py:517-537)

```python
def get_jobid_error_file(self, date):
    """Function to get error file for a given date"""
    
    output_folders = glob.glob(f"{slurm_logs_dir}/output*")
    
    # Search for execution failure patterns
    retcode, output, error = self.execute_command(f"grep -lR 'BIPS EXECUTION FAILED' {output_folder}")
    output_file = [os.path.join(slurm_logs_dir, file.strip()) for file in output.split("\n") 
                   if self.is_valid(file, date) and len(file) > 0]
    
    # Check for successful completion
    if len(output_file) == 0:
        retcode, output, error = self.execute_command(f"grep -lR 'BIPS EXECUTION FINISHED SUCCESSFULLY' {output_folder}")
```

#### Date Validation: `is_valid()` (ssm_driver.py:500-514)

```python
def is_valid(self, job_id_out, check_date):
    """Function to check if the job_id_out is valid for a given date"""
    try:
        date = '20' + str(job_id_out.strip().split(".")[-2].split('_')[-1])
        if len(date) != 8 or not date.isdigit():
            return False
        
        datetime.strptime(date, '%Y%m%d')
        if check_date == date:
            return True
```

### 3. Resubmission Execution

#### Main Resubmission Logic: `resubmit_failed_date()` (ssm_driver.py:540-577)

The core resubmission logic follows this workflow:

```python
@task(cache_policy=NONE)
def resubmit_failed_date(self, date):
    """Function to resubmit slurm job for a failed date with a known error"""
    
    while not self.stop_resubmit_failed_date_event[date]:
        output_file, date_failed = self.get_jobid_error_file(date)
        
        if output_file is not None:
            if date_failed and self.file_contains_known_error(output_file):
                self.print_and_log(f"Resubmitting slurm job for date: {date}")
                self.resubmission_happened = True
                self.remove_dated_dirs(date)
                slurm_ini_path = self.update_failed_slurm_config(date)
                
                if os.path.exists(slurm_ini_path):
                    # Create resubmission directory structure
                    rs_config_path = f"{self.base_dir}/resubmit/config/rs_input_{date}.ini"
                    await_results_path = f"{self.base_dir}/resubmit/await_results/results_{date}"
                    logging_path = f"{self.run_sims_logging_path}/resubmit/{date}_dag_logs"
                    
                    os.makedirs(await_results_path, exist_ok=True)
                    
                    # Configure resubmission parameters
                    rs_input_config = ConfigObj(rs_config_path)
                    rs_input_config['main'] = {'await_results_path': await_results_path}
                    rs_input_config.write()
                    
                    # Execute resubmission
                    cmd = f"{project_dir}/wrappers/run_sims_dag slurm {slurm_ini_path} -o {self.base_dir}/resubmit/{date}_outdir -i {rs_config_path} --logging_path {logging_path} --skip_notify --wait"
                    self.execute_command(cmd)
                    self.stop_resubmit_failed_date_event[date] = True
```

#### Configuration Updates: `update_failed_slurm_config()` (ssm_driver.py:459-489)

```python
def update_failed_slurm_config(self, date):
    """Function to resubmit slurm job for a failed date"""
    
    with open(slurm_ini, 'r+') as f:
        text = f.read()
    
    # Update configuration for resubmission
    text = re.sub(r"DATES=.*", f'DATES="{date}"', text)
    
    # Prevent directory clearing
    if re.search(r'export CLEAR_DIR=.*', text):
        text = re.sub(r'export CLEAR_DIR=.*', 'export CLEAR_DIR=false', text)
    else:
        text += "\nexport CLEAR_DIR=false"
    
    # Update job ID with resubmit suffix
    if re.search(r'export SLURM_OWN_JOB_ID=.*', text):
        text = re.sub(r'(export SLURM_OWN_JOB_ID=.*)', r'\1' + f'-resubmit-{date}', text)
    else:
        text += f"\nexport SLURM_OWN_JOB_ID=runsims-resubmit-{date}"
```

#### Directory Management: `remove_dated_dirs()` (ssm_driver.py:448-457)

```python
def remove_dated_dirs(self, date):
    """Function to remove dated directories of failed dates"""
    
    date_dir = f"{slurm_logs_dir}/{date}"
    if os.path.exists(date_dir):
        self.print_and_log(f"Removing dated directory: {date_dir}")
        self.execute_command(f"rm -rf {date_dir}")
```

### 4. Resubmission Completion Monitoring

#### Completion Check: `resubmit_jobs_completed()` (await_slurm_run.py:641-672)

```python
async def resubmit_jobs_completed(self):
    """ Checks if all jobs that are resubmitted are completed"""
    
    resubmit_dag_logs = f"{self.logging_path}/resubmit"
    if self.await_results_path == 'None' and not os.path.exists(resubmit_dag_logs):
        return True
    
    # Check for completion markers
    results_dirs = glob.glob(f"{await_results_dirs}/results*")
    resubmit_dag_logs_dirs = glob.glob(f"{resubmit_dag_logs}/*")
    slurm_summaries = glob.glob(f"{await_results_dirs}/results*/slurm_summary*")
    
    if len(results_dirs) == len(slurm_summaries) == len(resubmit_dag_logs_dirs) and len(results_dirs) > 0:
        return True
```

#### Integration with Job Monitoring: `check_if_jobs_completed()` (await_slurm_run.py:372-388)

```python
async def check_if_jobs_completed(self, monitor_job_id):
    """Check if all jobs are completed and Check for bad states"""
    
    if len(self.final_df[~self.final_df['STATE'].isin(self.SLURM_JOB_END_STATES)]) == 0 and len(self.final_df) > 0:
        if len(self.final_df[self.final_df['STATE'].isin(self.SLURM_BAD_STATES)]) == 0:
            if await self.resubmit_jobs_completed():
                self.log_info(f"Monitoring Complete for {monitor_job_id}")
                self._set_job_complete_result()
            else:
                self.log_info("Resubmitted jobs are not completed yet. Waiting for completion")
```

## Result Consolidation

### 1. Results Merging: `combine_await_slurm_results()` (ssm_driver.py:625-677)

#### CSV Results Consolidation (ssm_driver.py:628-644):
```python
def combine_await_slurm_results(self):
    """Function to concatenate slurm results of different resubmitted jobs"""
    
    # Combine slurm_results.csv files
    files = glob.glob(f"{self.base_dir}/resubmit/await_results/results*/slurm_results.csv")
    df = pd.read_csv(f"{slurm_logs_dir}/slurm_results.csv")
    
    for file in files:
        try:
            tmp_df = pd.read_csv(file)
            df = pd.concat([df, tmp_df], ignore_index=True)
        except Exception as e:
            self.print_and_log(f"Error in concatenating slurm results for file {file}. Error : {e}")
    
    df = df.drop_duplicates()
    df.to_csv(f"{slurm_logs_dir}/slurm_results.csv", index=False)
```

#### Summary Files Consolidation (ssm_driver.py:646-664):
```python
# Combine slurm_summary.ini files
files = glob.glob(f"{self.base_dir}/resubmit/await_results/results*/slurm_summary.ini")
config = ConfigObj(f"{slurm_logs_dir}/slurm_summary.ini")
c_dates = set(config.get('Completed Dates', []))
f_dates = set(config.get('Failed Dates', []))

for file in files:
    tmp_config = ConfigObj(file)
    tmp_c_dates = set(tmp_config.get('Completed Dates', []))
    c_dates = c_dates.union(tmp_c_dates)
    f_dates -= tmp_c_dates  # Remove completed dates from failed dates

config['Completed Dates'] = list(c_dates)
config['Failed Dates'] = list(f_dates)
```

### 2. Return Code Management (ssm_driver.py:666-672):

```python
# Update return code if any dates completed on resubmission
if len(c_dates) > 0:
    self.print_and_log("On resubmission some dates are Completed. Updating retcode of mss wrapper to 0")
    self.retcode = 0
    
    with open(self.retcode_file, 'w') as f:
        f.write("0")
```

### 3. Completion Marker: `postprocess_resubmitted_jobs()` (ssm_driver.py:679-686)

```python
def postprocess_resubmitted_jobs(self):
    """Function to postprocess the resubmitted jobs"""
    
    # Combine results and update return code
    self.combine_await_slurm_results()
    
    # Create completion marker
    with open(f"{self.base_dir}/resubmit/resubmission_completed.ready", 'w') as f:
        pass
```

## Integration Points

### Main Execution Flow (ssm_driver.py:1520-1527):

```python
# Wait for resubmission completion
if self.possible_to_resubmit:
    self.start_resubmit_failed_dates_task(stop=True)
    
    # Postprocess resubmitted jobs
    if not os.path.exists(f"{self.base_dir}/resubmit/resubmission_completed.ready") and self.resubmission_happened:
        self.print_and_log("Postprocessing resubmitted jobs")
        self.postprocess_resubmitted_jobs()
```

### Normal vs Resubmitted Job Handling (await_slurm_run.py:750-753):

```python
def postprocess_normal(self, monitoring_id):
    """Applies Post Processing to normal sims"""
    
    for id, status in self.final_df[['JOBID', 'STATE']].values:
        if self.await_results_path == 'None':
            if 'resubmit' in id:
                continue  # Skip resubmitted jobs in normal processing
```

## Directory Structure

The resubmit mode creates the following directory structure:

```
{base_dir}/resubmit/
├── config/
│   ├── slurm_{date}.ini          # Modified SLURM config for each failed date
│   └── rs_input_{date}.ini       # RunSims input config
├── await_results/
│   └── results_{date}/
│       ├── slurm_results.csv     # Job execution results
│       └── slurm_summary.ini     # Completion/failure summary
├── {date}_outdir/                # Output directory for resubmitted job
└── resubmission_completed.ready  # Completion marker file
```

## Detailed Resubmission Flow

### Step-by-Step Process:

1. **Detection Phase** (ssm_driver.py:544-547):
   - Monitor job output files continuously
   - Check for failure patterns using `get_jobid_error_file()`
   - Validate date format and error types

2. **Preparation Phase** (ssm_driver.py:548-552):
   - Mark resubmission as happening (`self.resubmission_happened = True`)
   - Clean up failed job directories (`remove_dated_dirs()`)
   - Update SLURM configuration for the specific failed date

3. **Execution Phase** (ssm_driver.py:554-567):
   - Create resubmission directory structure
   - Generate new input configuration with updated paths
   - Execute resubmission with `run_sims_dag` wrapper
   - Use `--skip_notify` and `--wait` flags for synchronous execution

4. **Monitoring Phase** (await_slurm_run.py:641-672):
   - Track resubmitted jobs alongside original jobs
   - Wait for completion of all resubmission tasks
   - Verify presence of result files and summary data

5. **Consolidation Phase** (ssm_driver.py:625-677):
   - Merge CSV results from original and resubmitted jobs
   - Combine summary files, prioritizing successful completions
   - Update overall return code based on final results

## Error Handling and Logging

### Error States and Recovery:
- **Transient Errors**: Automatically detected and resubmitted
- **Persistent Errors**: Jobs that fail again after resubmission are marked as permanently failed
- **Timeout Handling**: Resubmitted jobs are subject to the same timeout constraints as original jobs

### Logging Integration:
- All resubmission activities are logged through the standard logging mechanism (`self.print_and_log()`)
- Separate error files are created for debugging failed resubmissions
- Status updates are provided throughout the resubmission process
- Timestamps are included for all major resubmission events

### Thread Safety:
- Uses event-based coordination (`self.stop_resubmit_failed_date_event`)
- Employs locks for shared resource access
- Manages concurrent monitoring of multiple dates

## Key Benefits

1. **Automatic Recovery**: Reduces manual intervention for transient infrastructure failures
2. **Result Integrity**: Maintains comprehensive tracking of all job attempts
3. **Selective Resubmission**: Only retries jobs with known recoverable errors
4. **Consolidated Results**: Seamlessly merges original and resubmitted job results
5. **Non-Intrusive**: Does not affect normal job execution flow when not needed

## Limitations

1. **Scope**: Only available for unittest scenarios with single SLURM configurations
2. **Error Types**: Limited to predefined set of known recoverable errors
3. **Retry Logic**: Single retry attempt per failed date
4. **Resource Overhead**: Additional compute resources required for resubmission
5. **Configuration Constraints**: Cannot be used with advanced SLURM features (TSPLIT, MODE, etc.)

## Configuration Parameters

### Required Environment Variables:
- `trigger_source`: Must be set to 'unittests'
- `base_dir`: Base directory for resubmission structure
- `run_sims_logging_path`: Path for resubmission logs

### Key Configuration Files:
- Original SLURM ini file (modified for resubmission)
- `rs_input_{date}.ini`: RunSims input configuration for each resubmitted date
- `slurm_summary.ini`: Job completion/failure summary
- `resubmission_completed.ready`: Completion marker file

## Troubleshooting

### Common Issues:
1. **Resubmission Not Triggered**: Check if `trigger_source` is 'unittests' and only one SLURM config is present
2. **Jobs Not Found**: Verify date format validation in `is_valid()` function
3. **Directory Permissions**: Ensure write access to base directory for resubmission structure
4. **Lock Contention**: Monitor for file locking issues during concurrent operations

### Debug Information:
- Check resubmission logs in `{run_sims_logging_path}/resubmit/`
- Verify presence of error patterns in job output files
- Monitor `resubmission_completed.ready` file creation
- Review consolidated results in final CSV and summary files
