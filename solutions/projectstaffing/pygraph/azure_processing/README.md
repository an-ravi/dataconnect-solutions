# Azure Processing
The `pygraph` module contains pyspark jobs which are meant to be run on an Azure Databricks (ADB) cluster:
+ [mail_enrichment](mail_enrichment_processor.py) - processes mails and extracts relevant data (such as tokens) from it,
+ [profiles_enrichment](profiles_enrichment_processor.py) - processes M365 profiles and extracts specific fields, to be reused within the main portal later,
+ [mail_role_detection](mail_role_detection_taxo.py) - processes mails and leverages FSE (Fast Sentence Embedding) to classify the employee roles.
+ [create_azure_search_index](create_azure_search_index.py) - creates an AzureSearch index (e.g. employees, emails), using a schema file from AZBS provided as argument

In production, these jobs are orchestrated by Azure DataFactory (ADF) pipelines.  

While doing development, the pyspark jobs can be run either:
+ locally - run using sample data locally:
  + profiles_enrichment (`process_local_job()` instead of `run_spark_job()`),
  + mail_enrichment (`process_local()` uncommented in `__main__`),
  + mail_role_detection - use `process_local()`.
+ directly on the ADB cluster, or
+ remotely, from the local environment to a target ADB cluster using `databricks-connect`.

## Set up jobs configuration for Databricks
The environment specific configuration for each job can be provided via a configurations file.  

For local debugging and execution you should have a configuration file named `config.json`, with the following structure:
```
{
  "SERVICE_PRINCIPAL_SECRET": "[...]"
}
```

The following fields need to be filled:
- `SERVICE_PRINCIPAL_SECRET` - the secret for the `gdc-service` service principal, that is used by all spark jobs to connect to Azure services (AZBS, AzureSql, KeyVault etc) 

> **Make sure not to commit this file** since it will contain sensitive information! Check if the rules from the `.gitignore` file cover your file name.  
> Only store locally information for non-critical environments, which don't contain sensitive data and which can be easily decommissioned!


Jobs like [`mail_enrichment_processor.py`](./mail_enrichment_processor.py) or [`profiles_enrichment_processor.py`](./profiles_enrichment_processor.py) will run on the Databricks cluster,and can be monitored in the cluster's SparkUI.

Be mindful if cluster is active/inactive, in order to:
+ minimize load - if cluster is used for upgrade / demo
+ TCO - cluster doesn't have to be spun up needlessly, costs should be limited.


## Python Environment Setup
Set up your environment, and activate it both in your shell and in your IDE:
+ `conda create --name gdc --file requirements_conda.txt`

If any changes to the setup are needed, remember to document changes in requirements:
+ preferably with conda: `conda list --export > requirements_conda.txt` (rather than `pip freeze > requirements.txt`)

Any subsequent installs have to be made in the:
+ activated conda environment - `conda install package-name=2.3.4`
+ outside conda environment - `conda install package-name=2.3.4 -n gdc`

Note: Latest requirements file has the `smart_open` import bug from gensim patched by upgrading smart_open (`conda update smart_open`). You shouldn't encounter this error unless `gensim` reinstalls the faulty `smart_open` version.


## Pyspark / Databricks setup
Oracle changed the licensing in 2019, so these will not work ([source](https://github.com/Homebrew/homebrew-cask-versions/issues/7253)):
+ `brew cask install java` or
+ `brew install --cask java8`

Installer links:
+ [JRE](https://www.java.com/en/download/)
+ [JDK8](https://www.oracle.com/ro/java/technologies/javase/javase-jdk8-downloads.html)
+ [JDK15](https://www.oracle.com/java/technologies/javase-jdk15-downloads.html)

Below steps have been used and tested for Mac OS X Catalina.


### Java Setup
In order to allow for multiple versions of java on the same system, we use `jenv`:
+ install - `brew install jenv`
+ append the following to config file (`~/.bashrc` or `~/.zshrc`) and run `source` or restart/open new shell:
```shell script
export PATH="$HOME/.jenv/bin:$PATH"
eval "$(jenv init -)"
```
+ check if jenv works - `jenv doctor`

After installing needed JDKs:
+ check installed Java VMs - `/usr/libexec/java_home -V`
+ add installed Java VMs to jenv:
    + `jenv add /Library/Java/JavaVirtualMachines/jdk-15.0.1.jdk/Contents/Home`
    + `jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home`
+ check - `jenv versions`

Optional, but recommended:
+ `jenv enable-plugin export`
+ `jenv enable-plugin maven`

Set JDK needed for Spark:
+ set java 1.8 as system-level java - `jenv global 1.8`
+ sanity check - `jenv doctor`
+ sanity check - `java -version`


### Python Setup
Similar to what we did for Java, in this case for Python:
+ install - `brew install pyenv pyenv-virtualenv`
+ append the following to config file (`~/.bashrc` or `~/.zshrc`) and run `source` or restart/open new shell:
```shell script
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```
+ check - `pyenv versions`
+ install - `pyenv install -v 3.7.8` => why `<3.8` ([source](https://stackoverflow.com/questions/58700384/how-to-fix-typeerror-an-integer-is-required-got-type-bytes-error-when-tryin))
+ set installed version globally - `pyenv global 3.7.8`
+ sanity check - `pyenv versions`

Your config file (`~/.bashrc` or `~/.zshrc`) should look at the end like this:
```shell script
# set only if not using databricks-connect
# export SPARK_HOME="/usr/local/Cellar/apache-spark/3.0.1/libexec"
# export PYSPARK_PYTHON="python3"
# export PATH=$PATH:$SPARK_HOME/bin
# export PATH=$PATH:$PYSPARK_PYTHON

export PATH="$HOME/.jenv/bin:$PATH"
eval "$(jenv init -)"

export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```


### Spark / Databricks Setup
+ install - `brew install apache-spark`
    + usually located in `/usr/local/Cellar/apache-spark/3.0.1/libexec/`

+ install - `pip3  install databricks-connect==6.6.0` - this version is needed for our current Databricks configuration (5.5.* at least, 6.4 should also work)
+ sanity check - `python3 -c 'import pyspark'`

+ sanity check - `spark-shell` (use `:quit`)
+ sanity check - `pyspark` (use `quit()`)
+ configure according to the below specs - `databricks-connect configure`
+ check - `databricks-connect test`
    + may error if pyspark wasn't uninstalled / left garbage folders (for some reason) -> `rm -rf /Library/Frameworks/Python.framework/Versions/3.8/lib/python3.8/site-packages/pyspark`


Optional objective:
+ TODO: can use local Spark while `databricks-connect` is installed and configured ?


### Finishing setup

Check your PATH:
+ `echo $PATH | tr ":" "\n"`
+ if there are duplicates, `echo $PATH` and clean in your IDE of preference
    + purge references to system-wide python (such as: `/Library/Frameworks/Python.framework/Versions/3.8/bin`) - these will confuse the system as to which `databricks-connect` to use, among others (eg: `site-packages` location used)
    + ( **VERY IMPORTANT** ) if Spark previously used in the system, `unset SPARK_HOME` - this along with other package confusion can lead to a faulty setup (hard to diagnose errors like "*TypeError: 'Java Package' object is not callable*")


Note: ( **VERY IMPORTANT** ) Perform sanity checks for faulty `PATH` or `SPARK_HOME` set in the following locations:
+ `~/.bash_rc`, `~/.bash_profile`
+ `~/.zsh_rc`, `~/.zprofile`

Note: Also try to perform sanity checks in a new tab and your IDE of choice:
+ for `jenv`, `pyenv`, `databricks-connect` (mentioned above).
+ run `python spark_test.py` and check the Spark UI


---

## More links:
+ https://docs.oracle.com/javase/8/docs/technotes/guides/install/mac_jdk.html
+ https://medium.com/@chamikakasun/how-to-manage-multiple-java-version-in-macos-e5421345f6d0
+ https://github.com/jenv/jenv/issues/212
+ https://stackoverflow.com/questions/41129504/pycharm-with-pyenv
+ https://docs.databricks.com/dev-tools/databricks-connect.html