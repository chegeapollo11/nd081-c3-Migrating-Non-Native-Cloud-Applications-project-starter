# TechConf Registration Website

## Project Overview
The TechConf website allows attendees to register for an upcoming conference. Administrators can also view the list of attendees and notify all attendees via a personalized email message.

The application is currently working but the following pain points have triggered the need for migration to Azure:
 - The web application is not scalable to handle user load at peak
 - When the admin sends out notifications, it's currently taking a long time because it's looping through all attendees, resulting in some HTTP timeout exceptions
 - The current architecture is not cost-effective 

In this project, you are tasked to do the following:
- Migrate and deploy the pre-existing web app to an Azure App Service
- Migrate a PostgreSQL database backup to an Azure Postgres database instance
- Refactor the notification logic to an Azure Function via a service bus queue message

## Dependencies

You will need to install the following locally:
- [Postgres](https://www.postgresql.org/download/)
- [Visual Studio Code](https://code.visualstudio.com/download)
- [Azure Function tools V3](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash#install-the-azure-functions-core-tools)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Azure Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack)

## Project Instructions

### Part 1: Create Azure Resources and Deploy Web App
1. Create a Resource group
2. Create an Azure Postgres Database single server
   - Add a new database `techconfdb`
   - Allow all IPs to connect to database server
   - Restore the database with the backup located in the data folder
3. Create a Service Bus resource with a `notificationqueue` that will be used to communicate between the web and the function
   - Open the web folder and update the following in the `config.py` file
      - `POSTGRES_URL`
      - `POSTGRES_USER`
      - `POSTGRES_PW`
      - `POSTGRES_DB`
      - `SERVICE_BUS_CONNECTION_STRING`
4. Create App Service plan
5. Create a storage account
6. Deploy the web app

### Part 2: Create and Publish Azure Function
1. Create an Azure Function in the `function` folder that is triggered by the service bus queue created in Part 1.

      **Note**: Skeleton code has been provided in the **README** file located in the `function` folder. You will need to copy/paste this code into the `__init.py__` file in the `function` folder.
      - The Azure Function should do the following:
         - Process the message which is the `notification_id`
         - Query the database using `psycopg2` library for the given notification to retrieve the subject and message
         - Query the database to retrieve a list of attendees (**email** and **first name**)
         - Loop through each attendee and send a personalized subject message
         - After the notification, update the notification status with the total number of attendees notified
2. Publish the Azure Function

### Part 3: Refactor `routes.py`
1. Refactor the post logic in `web/app/routes.py -> notification()` using servicebus `queue_client`:
   - The notification method on POST should save the notification object and queue the notification id for the function to pick it up
2. Re-deploy the web app to publish changes

## Monthly Cost Analysis
Complete a month cost analysis of each Azure resource to give an estimate total cost using the table below:

| Azure Resource | Service Tier | Monthly Cost |
| ------------ | ------------ | ------------ |
| *Azure Database for PostgreSQL server* | Basic | $25.80 |
| *Service Bus Namespace* | Basic | Variable ($0.05 per million operations) |
| *Storage account* | general purpose v2 (Standard/Hot) | Variable ($0.00081/GB) |
| *Function App* | Consumption (Serverless) | Variable ($0.20 per million executions + $0.000016/GB-s for execution time) |
| *App Service* | F1 | $0.00 (Free) |
| **Total Cost** |  | **~ $26.05** |

## Architecture Explanation
Using Azure App Service for the Tech Conf Web Application was my preferred choice due to the following reasons:
1. Azure App Service offers a platform as a service solution that is quick, easy and straight forward to configure and deploy for an application built using the Python Flask framework such as the Tech Conf web application.
2. Azure App Service has a free (F1) pricing tier which offers a free and cost effective solution for non production applications which is our current scenario.
3. Customization of the underlying host operating system and installation of additonal software was not a requirement hence Azure App Service offered an out of the box solution that could host and run the Tech Conf web application.

Using an Azure Function App together with an Azure Service Bus Queue for the notification sending logic was my preferred choice due to the following reasons:
1. Sending emails on average could easily take more than 30 seconds especially when sending notifcations to many recipients so having this logic in the front end application results into slower response rates and a poor user experience. Moving this functionality to an azure function ensures that this logic is processed asynchronously in a background task resulting into faster response times for the front end application and hence a better user experience.
2. Azure Function Apps configured with a Consumption Plan for the Hosting option are charged only when the function executes and based on the execution time of the function thus is a cost effective solution.
3. Moving the send notification logic to a background job using an Azure Function App also results in a scaleable solution that can handle millions of notifications as the number of application users grows. Send notification requests are now processed outside the front end application resulting into less utilization of server resources for the front end application and hence the capability to support more users. 
