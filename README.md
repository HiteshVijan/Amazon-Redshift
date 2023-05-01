# Analyze Data with Amazon Redshift

Data analysts often work with very large datasets. For example, some data warehouses can contain multiple petabytes of data, and data lakes can be even larger. Amazon Redshift is designed to handle extremely large datasets. Unlike traditional relational database management systems, Amazon Redshift uses columnar data storage. Columnar storage improves query performance and saves on storage costs. To read more about columnar storage, see [Columnar storage](https://docs.aws.amazon.com/redshift/latest/dg/c_columnar_storage_disk_mem_mgmnt.html) in the Amazon Redshift documentation.


# Objectives

After completing this lab, you will be able to:

- Access Amazon Redshift in the AWS Management Console
- Create an Amazon Redshift cluster.
- Load data from Amazon Simple Storage Service (Amazon S3) into an Amazon Redshift table
- Query data in Amazon Redshift

**Prerequisites**

This lab requires:

- Access to a notebook computer with Wi-Fi and Microsoft Windows, macOs, or Linux (Ubuntu, SuSE, or Red Hat)
- For Microsoft Windows users: Administrator access to the computer
- An internet browser such as Chrome, Firefox, or IE9 (previous versions of Internet Explorer are not supported)

**Duration**

This lab requires **60** minutes to complete. The lab will remain active for **120** minutes to allow extra time to complete the challenge questions.

## Accessing the AWS Management Console

1. At the top of these instructions, choose <span id="ssb_voc_grey">Start Lab</span> to launch your lab.

  A **Start Lab** panel opens, which displays the lab status.

2. Wait until you see the message *Lab status: ready*, then close the **Start Lab** panel by choosing the **X**.

3. At the top of these instructions, choose <span id="ssb_voc_grey">AWS</span>

  This will open the AWS Management Console in a new browser tab. The system will automatically log you in.

  **Tip**: If a new browser tab does not open, there will typically be a banner or icon at the top of your browser that indicates that your browser is preventing the website from opening pop-up windows. Choose the banner or icon and then choose **Allow pop ups**.

4. Arrange the **AWS Management Console** browser tab so that it displays next to these instructions. Ideally, you should be able to see both browser tabs at the same time, which can make it easier to follow the lab steps.

## Scenario summary ##

In this lab, you are a data analyst who works for an enterprise that manages ticket sales for music events. The company has a web application that aggregates ticket sales between buyers and sellers. When customers buy tickets to these events, the web application stores data about both the buyer and the seller.

Your manager asked you to generate several reports from the company's data files. These data files are already stored in Amazon S3, so you decide to use Amazon Redshift to load the data and build the reports.

The following diagram illustrates the environment you will build in this lab:

<img src="images/lab-4-overview.png" alt="AWS Glue crawler output" width="400">

## Task 1: Review the security group for accessing the Amazon Redshift console

When you access the Amazon Redshift console, you must be authenticated and authorized for Amazon Redshift access. This access is controlled by an AWS Identity and Access Management (IAM) role. Roles are used by IAM to limit which actions console users can take. A role has already been created for this exercise.

  **Note**: In a real-world scenario, an administrator would create this role for any user who must access Amazon Redshift. To learn more about how IAM and Amazon Redshift work together, see <a href="https://docs.aws.amazon.com/redshift/latest/mgmt/authorizing-redshift-service.html" target="_blank">Authorizing Amazon Redshift to Access Other AWS Services on Your Behalf</a>.

To review the IAM role:

5. From the list of services, choose **IAM**.

6. From the navigation , choose **Roles**.

7. In the **Search** window, enter `myRedshiftRole`

8. Choose **myRedshiftRole**.

9. Choose **AmazonS3ReadOnlyAccess**.

The policy document opens. This policy gives Amazon Redshift permission to get a list of items from Amazon S3 and to retrieve those items.


## Task 2: Create and configure an Amazon Redshift cluster

Clusters are the main infrastructure component of an Amazon Redshift data warehouse. A cluster is made up of one or more compute nodes. If there is more than one node, then one of the nodes will be the *leader node* and the other nodes will be *compute nodes*. Client applications interact with the leader node. The following diagram illustrates the overall Amazon Redshift infrastructure:

<img src="images/redshift.png" alt="Redshift infrastructure" width="400">

To read more about Amazon Redshift infrastructure, see <a href="https://docs.aws.amazon.com/redshift/latest/dg/c_redshift_system_overview.html" target="_blank">Amazon Redshift System Overview</a>.


To create a cluster for your data warehouse:

10. On the AWS Management Console, on the **Services** , choose **Amazon Redshift**.

    **Note**: You might see an error in the Amazon Redshift console that states that you are not authorized to describe reserved node offerings. This is because of security restrictions in the lab environment. You can ignore the error.

11. Choose **Create cluster**.

12. Under **Cluster configuration**, enter the following values. Some of these configurations might already be prefilled with the correct values:

  - **Cluster identifier:**  `redshift-cluster-1`
- **Node type:** *dc2.large*
  - **Nodes:** `2`
  - **Database port:**  `5439`
  - **Master user name:**  `awsuser`
  
13. Enter the following value for the **Master user password**:
  
  - **Master user password:** `Passw0rd1`
  
14. Expand **Cluster permissions** and select *myRedshiftRole* from the drop down list.

15. Choose **Associate IAM role**.

16. Choose **Create cluster**.

   It will take a few minutes for the Amazon Redshift to create the cluster.

17. From the navigation pane, choose **Clusters**.

18. Choose **View all clusters**.

Note that *redshift-cluster-1* is in the list of clusters, and the status shows *creating*.

### Task 2.1: Create a security group for your cluster

You must configure the virtual private cloud (VPC) that hosts your Amazon Redshift cluster so it allows traffic through port *5439*. To do this, you must create and configure a security group for your VPC.

19. On the AWS Management Console, go to the **Services**  and choose **EC2**.
20. In the **NETWORK & SECURITY** section, choose **Security Groups**.
21. Choose **Create Security Group**.
22. Enter the name `Redshift Security Group` and the description `Security group for my Amazon Redshift cluster`.
23. In the **Security group rules** section, choose **Add Rule**.
24. In the **Security group rules** section, enter the following values:

   - **Type:**  *Redshift*
   - **Protocol:**  *TCP*
   - **Port Range:** *5439*
   - **Source:** *Anywhere*
   - **IP:**  *0.0.0.0/0,::/0*
   - **Description:** `Redshift inbound rule`
25. Choose **Create**.

### Task 2.2: Configure your Amazon Redshift cluster

26. On the AWS Management Console, on the **Services** menu, choose **Amazon Redshift**.

27. From the navigation pane, choose **Clusters**.

  On the **Clusters** page, you will see a list with the name of the cluster that you created. The **Status** column will display *pending* until your cluster is ready for the next step. You may need to use the refresh button on the top-right corner to refresh the window until you see the status change.

28. When the **Cluster Status** changes to *available* and the **DB health** displays *healthy*, go to the list of clusters and choose your cluster. 

29. Navigate to the **Properties** tab.

30. In the **Network and security settings** section, choose **Edit*.* **

31. Select the **Redshift Security Group** from the dropdown list and remove the **default** group.

32. Choose **Save changes**.

## Task 3: Load data into your Amazon Redshift cluster

Now that you created your cluster, you can load data into the cluster from Amazon S3. Amazon Redshift can store data from a variety of formats, including:

- Text
- Apache Avro
- JavaScript Object Notation (JSON)
- Apache Parquet
- Optimized Row Columnar (ORC)

The data you use in this lab is stored as text files with the pipe (|) delimiter. To load the data, you will first create the tables with the `create table` command, and then copy the data with the `copy` command.

### Task 3.1: Create the tables in the dev database

33. From the navigation, choose **Clusters**.

34. From the list of clusters, choose **redshift-cluster-1**.

35. From the navigation pane, choose **Query editor**.

36. Choose **Connect to database** and enter the following values:

    - **Database:** `dev`
    - **Database user:** `awsuser`

37. Choose **Connect**.

   **Note**: If you receive this error message—*Query Editor is not available for the cluster you selected. Upgrade the cluster to the latest release to enable it.*—then refresh the Amazon Redshift page and try connecting again. The connection will eventually be established.

38. From the **Schema** dropdown list, choose the **public** schema.

39. Copy the following code block, and paste it into the query window. This code creates the *users* table.

   ```
   create table users(
   userid integer not null distkey sortkey,
   username char(8),
   firstname varchar(30),
   lastname varchar(30),
   city varchar(30),
   state char(2),
   email varchar(100),
   phone char(14),
   likesports boolean,
   liketheatre boolean,
   likeconcerts boolean,
   likejazz boolean,
   likeclassical boolean,
   likeopera boolean,
   likerock boolean,
   likevegas boolean,
   likebroadway boolean,
   likemusicals boolean);
   ```

40. Choose **Run query**.

   After the query completes, you should see *Statement completed successfully*.

41. Copy the following code block, and paste it into the query window. This code creates the *data* table.

   ```
   create table date(
   dateid smallint not null distkey sortkey,
   caldate date not null,
   day character(3) not null,
   week smallint not null,
   month character(5) not null,
   qtr character(5) not null,
   year smallint not null,
   holiday boolean default('N'));
   ```

42. Choose **Run query**.

43. Copy the following code block, and paste it into the query window. This code creates the *sales* table.

   ```
   create table sales(
   salesid integer not null,
   listid integer not null distkey,
   sellerid integer not null,
   buyerid integer not null,
   eventid integer not null,
   dateid smallint not null sortkey,
   qtysold smallint not null,
   pricepaid decimal(8,2),
   commission decimal(8,2),
   saletime timestamp);
   ```

44. Choose **Run query**.

### Task 3.2: Load data from Amazon S3

To load the data from Amazon S3:

45. Copy and paste the following code into the query window. Replace *&lt;iam-role-arn&gt;* with the ARN that you stored when you reviewed the *MyRedshiftRole* in Task 1.

  ```
  copy users from 's3://awssampledbuswest2/tickit/allusers_pipe.txt'
  credentials 'aws_iam_role=<iam-role-arn>'
  delimiter '|' region 'us-west-2';
  ```

  After the query runs, you should see *Statement completed successfully*.

46. Copy and paste the following code into the query window. Replace *&lt;iam-role-arn&gt;* with the ARN that you stored when you reviewed the *MyRedshiftRole* in Task 1.

  ```
  copy date from 's3://awssampledbuswest2/tickit/date2008_pipe.txt'
  credentials 'aws_iam_role=<iam-role-arn>'
  delimiter '|' region 'us-west-2';
  ```

  The sales data is stored with a \t delimiter. Because the data includes time data, you must also specify the time format.

47. Copy the following code and run it in the query window. Replace *&lt;iam-role-arn&gt;* with the ARN that you stored when you reviewed the *MyRedshiftRole* in Task 1.

  ```
  copy sales from 's3://awssampledbuswest2/tickit/sales_tab.txt'
  credentials 'aws_iam_role=<iam-role-arn>'
  delimiter '\t' timeformat 'MM/DD/YYYY HH:MI:SS' region 'us-west-2';
  ```


## Task 4: Query the data ##

Now that you loaded the data into your Amazon Redshift cluster, you can write queries to generate the reports that your manager asked for. Here are two queries to get you started:

To see the overall structure of the *sales* table:

48. View the first 10 records by going to the *sales* table and choosing the **Preview** icon next to it.

  You should see query results like the following example:

<img src="images/lab4queryresults.png" alt="Query Editor results" width="400">

49. To retrieve the number of items sold for a particular date, copy the following query, paste it into the **Query window**, and choose **Run query**.

  ```sql
  SELECT sum(qtysold)
  FROM sales, date
  WHERE sales.dateid = date.dateid
  AND caldate = '2008-01-05';
  ```

  The query should return the value of *210*.

50. To find the top 10 buyers by quantity, copy the following query, paste it into the **Query window**, and choose **Run query**.

  ```sql
  SELECT firstname, lastname, total_quantity
  FROM
  (SELECT buyerid, sum(qtysold) total_quantity
  FROM  sales
  GROUP BY buyerid
  ORDER BY total_quantity desc limit 10) Q, users
  WHERE Q.buyerid = userid
  ORDER BY Q.total_quantity desc;
  ```

The query should return a list of customers and the quantity that was sold.

## Challenge questions

Now that you can run queries, try the following challenges to get more experience with the Amazon Redshift Query Editor.

### Challenge one

Write a query to find events in the highest 99.9 percentile for all-time gross sales.

  **Hint**: The structured query language (SQL) `ntile` function will return a dataset into a group of buckets as specified by the function's argument. For example, `ntile(1000) orders` will return the highest 99.9 percentile for a dataset of orders.

### Challenge two

Write a query to list the event IDs and the total sales for each event in descending order.

## Lab complete

<i class="icon-flag-checkered"></i> Congratulations! You have completed the lab.

51. At the top of this page, choose <span id="ssb_voc_grey">End Lab</span> and then confirm that you want to end the lab by choosing <span id="ssb_blue">Yes</span>

   A panel will appear, with this message: *DELETE has been initiated... You may close this message box now.*

52. To close the panel, go to the top-right corner and choose the **X**.

