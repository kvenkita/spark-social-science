
# WARNING - PYSPARK IN DEVELOPMENT

Recent changes may have significantly changed/affected functionality. The SparkR functionality is now considered stable, and further development will happen outside of the master branch.

# spark-social-science
Automated Spark Cluster Builds with RStudio or PySpark for Policy Research.<br>
[Comments welcome on Github or on Twitter @alexcengler]

The goal of this project is to deliver powerful and elastic Spark clusters to researchers and data analysts with as little setup time and effort possible. To do that, at the Urban Institute, we use two critical components: (1) a Amazon Web Services (AWS) CloudFormation script to launch AWS Elastic MapReduce (EMR) clusters (2) a bootstrap script that runs on the Master node of the new cluster to install statistical programs and development environments (RStudio and Jupyter Notebooks). Further down, you can also look at an AWS Command Line approach to using the bootstrap scripts.

This guide illustrates the (relatively) straight forward process to provide clusters for your analysts and researchers. For more information about using Spark via the R and Python APIs, see these repositories:

<ul>
	<li>https://urbaninstitute.github.io/spark-social-science-manual/</li>
	<li>https://github.com/UrbanInstitute/sparkr-tutorials</li>
	<li>https://github.com/UrbanInstitute/pyspark-tutorials</li>
</ul>

### Security Disclaimer:

I am not a security expert and this guide does not specify security specifics necessary for secure data analysis. There are many important security considerations when doing data analysis in the cloud that this guide does not cover. Please make sure to consult with security experts whenever working in cloud compute.


## Setup 

Just a few steps will get you to working Spark Clusters. We're going to setup the bootstrap scripts first, then the CloudFormation script. Note that all the required changes that you need to make involve replacing text that contains the phrase 'goes-here'. Search for that phrase within the CloudFormation script and bootstrap files to make sure you have replaced all those text fields.

### 1. Setup Your Bootstrap Scripts

Bootstrap scripts run at the start of the cluster launch. We're going to host them in AWS S3, then reference them in our CloudFormation Template. So, everytime we launch a cluster, the bootstrap scripts will run, setting up our statistical environment: either R and RStudio pointed at SparkR, or Python and Jupyter Notebooks pointed at PySpark.

First, create an AWS S3 bucket for your EMR scripts and logs. You'll want to put the pertinent shell script (`rstudio_sparkr_emr5lyr-proc.sh` or `jupyter_pyspark_emr5-proc.sh`) into that bucket. Then make a subfolder for you logs.

### 2. Setup Your CloudFormation Template

The CloudFormation Script needs a few changes to work as well.

<ul>
	<li>Replace the phrase 'your-bucket-name-goes-here' with the name of the bucket you created a minute ago for your bootstrap scripts.</li>
	<li>Create a new S3 bucket for the logs from your clusters, and replace the phrase "logs-bucket-name-goes-here" with the name of your new bucket.</li>
	<li>Change the CIDR IP to your organzation's or your personal computer's IP. This will only allow your organization or your computer to access the ports you are opening for RStudio / Jupyter / Ganglia. This is optional, but know that if you do not do this, anyone can access your cluster at these ports.</li>
</ul>


### 3.a. Launch a Cluster with CloudFormation

<ul>
	<li>Go to CloudFormation on the AWS Dashboard - Hit Create Stack</li> 
	<li>Upload your CloudFormation Script - Hit Next</li>
	<li>On the 'Specify Details' page, you need to make a few changes, though once you have these figured out you can add them to the CloudFormation script.</li>
	<ul>
		<li>Create a Stack Name</li>
		<li>Set a Key Pair setup for your account (can set as a default within CloudFormation);</li>
		<li>Set a VPC - the default VPC that came with your AWS account will work;</li>
		<li>Set a Subnet (<a href="https://aws.amazon.com/about-aws/whats-new/2015/12/launch-amazon-emr-clusters-in-amazon-vpc-private-subnets/">can be private or public</a>);</li>
		<li>Set other options as needed, including changing the number of cores, the type of EC2 instances, and the tags (you can, but I don't recommend changing the ports).</li>
	</ul>
	<li>Hit next at the bottom of this screen, add tags if you want on the next screen, and hit Create Cluster.</li>
	<li>Go to your EMR dashboard and grab the DNS for the Master node. The whole string (seen below as 'ec2-54-89-114-32.compute-1.amazonaws.com') is your public DNS. <img src="./cluster-dns.png">
		<br>You should then be able to go to these URLs:
		<ul> 
			<li>RStudio at DNS:8787 - note that RStudio by default needs a username and password. These are set to 'hadoop' for both, and does affect how you are logged into the master node. We have run into errors changing this username and would be happy to hear about an alternative / fix.</li>
			<li>Jupyter Notebooks at DNS:8192</li>
			<li>Ganglia Cluster Monitoring at DNS/ganglia </li>
		</ul>
	</li>
</ul>

### 3.b. Launch a Cluster with the AWS Command Line

If you have installed and configured the AWS Command Line Interfance (CLI), you can run a single line to create a cluster. Note the CloudFormation scripts creates a security group during bootstrap and assigns it to the cluster. Below, you have to assign an existing security group `addtl-master-security-group` that opens up the correct port (8787 for RStudio and 8194 for Jupyter).

```shell
aws emr create-cluster --release-label emr-5.3.1 ^
  --name 'rstudio-sparkr-3-1' ^
  --applications Name=Spark Name=Ganglia ^
  --ec2-attributes KeyName=your-key-pair,InstanceProfile=EMR_EC2_DefaultRole,AdditionalMasterSecurityGroups="addtl-master-security-group",SubnetId="your-subnet" ^
  --service-role EMR_DefaultRole ^
  --instance-groups ^
    InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m4.xlarge ^
    InstanceGroupType=CORE,InstanceCount=2,InstanceType=m4.xlarge ^
  --region us-east-1 ^
  --log-uri s3://logs-bucket-name-goes-here ^
  --bootstrap-actions Path="s3://your-bucket-name-goes-here/rstudio_sparkr_emr5lyr.sh"
```


## Choosing Between SparkR & PySpark for Social Science

If you have a strong preference for either R or Python, you should let that preference guide your decision. Although R is much more convenient for statistical modeling, don't feel obligated to choose one over the other based on the impression they are dramatically different in terms of speed or available functionality:

<ul>
<li><b>Speed:</b> All of the language implementations (R, Python, and yes, Scala) are communicating with the same API - specifically the Spark DataFrames API. This means that the execution speeds written in any of the three languages are basically the same.</li>

<li><b>Available Functionality:</b> In terms of the Spark functionality available, there are some differences between SparkR and PySpark at present, with PySpark being more fleshed out. However, with the impending release of Spark 2.0 (DataBricks has suggested it will happen in the next few weeks), we expect many of these distinctions to disappear.</li>
</ul>

That being said, there are some notable differences that might affect your decision:

<ul>
<li><b>Statistical Modeling:</b> We have found SparkR to natively support statistical modeling with far more ease the PySpark. R's normal formula syntax works well, whereas PySpark's implementation has added complexity. If easy and familiar statistical modeling is important, you might want to stick with SparkR.</li>

<li><b>Code Syntax:</b> SparkR feels a bit more like normal R than PySpark feels like Python, although both have peculiarities. This is perhaps the case since R is natively built around dataframes, which are analagous to Spark’s DataFrame API. For this reason, you might prefer SparkR if you’re equally competent at R and Python. By the way, we oriented all of our tutorials (<a href = "https://github.com/UrbanInstitute/pyspark-tutorials">PySpark</a> / <a href="https://github.com/UrbanInstitute/sparkr-tutorials">SparkR</a>) around the DataFrame API, and strongly recommend that everyone works through this API (DataBricks makes the same suggestion).</li>

<li><b>Open Source Development:</b> It seems like Python has the advantage here, in that more of the <a href="https://spark-packages.org/">packages extending Spark functionality</a> come written in or accessible through PySpark than SparkR. This is likely a product of the current Spark community – more data scientists and engineers than more traditional statisticians and social scientists.</li>
</ul>
