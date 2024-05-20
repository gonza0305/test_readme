# Data Engineer Coding Exercise

Hi there!

If you're reading this, it means you're now at the coding exercise step of the data-engineering hiring process. 
We're really happy that you made it here and super appreciative of your time!

This repo is intentionally left empty except for the file you are currently reading. Please add your work with as much 
detail as you see fit.

## Expectations

* It should be executable, production ready code
* Take whatever time you need - we won’t look at start/end dates, you have a life besides this, and we respect that! 
Moreover, if there is something you had to leave incomplete or there is a better solution you would implement but 
couldn't due to personal time constraints, please try to walk us through your thought process or any missing parts, 
using the “Implementation Details” section below.

## About the Challenge

The goal of this exercise is for you to implement a fully functional end-to-end ETL pipeline via the open-source, 
full-stack data integration platform [Meltano](https://meltano.com/).

You will be working with the data provided by the [SpaceX API](https://github.com/r-spacex/SpaceX-API/tree/master), and
as it should be with any good data challenge, your work will be guided by one central question that we are aiming to
help find an answer for:

> When will there be 42,000 Starlink satellites in orbit, and how many launches will it take to get there?

You can assume that you will have a team of Data Analysts working in collaboration with you so you do not have to
provide the full solution. However, in your role as a Data Engineer, we would expect you to:

* create a Meltano custom extractor for [SpaceX-API](https://github.com/r-spacex/SpaceX-API/blob/master/docs/README.md) 
(following [this official tutorial](https://docs.meltano.com/tutorials/custom-extractor))
* configure Meltano`s [target-postgres](https://hub.meltano.com/loaders/target-postgres/) loader to send the data
extracted from the source by your custom extractor in the previous step into [Postgres](https://www.postgresql.org/)
* add a data transformation step via [Meltano`s implementation of dbt](https://docs.meltano.com/guide/transformation). 
Specifically:
  * Add a data transformation step via Meltano’s implementation of dbt. It must include a `transformation_updated_at` timestamp. This should result in a model (or models) that can be handed off to a team of data analysts.
* configure a [job](https://docs.meltano.com/reference/command-line-interface#job) in your Meltano project that runs the 
previous 3 steps in sequence

### When you are done

* Complete the "Implementation Details" section at the bottom of this README.
* Open a Pull Request in this repo and send the link to the recruiter with whom you have been in touch.
* You can also send some feedback about this exercise. Was it too short/big? Boring? Let us know!

## Useful Resources

* [Meltano Official Website](https://meltano.com/)
* [Meltano Docs](https://docs.meltano.com/)
* [Meltano Tutorial: Create a Custom Extractor](https://docs.meltano.com/tutorials/custom-extractor)
* [Meltano target-postgres Docs](https://hub.meltano.com/loaders/target-postgres/)
* [SpaceX-API Docs](https://github.com/r-spacex/SpaceX-API/blob/master/docs/README.md)
* [dbt Docs](https://docs.getdbt.com/)

  
## Implementation Details

### Table of Contents

1. [Description](#description)
2. [Architecture](#architecture)
4. [Components](#components)
5. [Implementation with Meltano](#implementation-with-meltano)
6. [How to run in locally](#how-to-run-it-locally)
7. [Considerations](#considerations)
8. [Challenges Encountered](#challenges-encountered)
9. [Future Implementations and Improvements](#future-implementations-and-improvements)
10. [Feedback](#feedback)

## Description

This project implements an end-to-end ETL pipeline using Meltano, targeting the goal of determining when there will be 42,000 Starlink satellites in orbit and how many launches it will take to get there. The pipeline extracts data from the SpaceX API, loads it into a PostgreSQL database, and transforms the data using dbt. Furthermore, the project is containerised, both the code and the postgre database. Furthermore, it has implemented CI/CD pipelines which, every time the project is updated, create new images of the project which are stored in ECR and run on ECS services.

## Architecture
The architecture of the project is illustrated in the following image:



![meltano_diagram drawio](https://github.com/gonza0305/test_readme/assets/15968266/5b605ce0-6561-4fed-9d70-aecb49478fc2)<br>




The project follows an ELT model where data is extracted using a custom extractor from the SpaceX API REST service, loaded into a PostgreSQL database, and then transformed using dbt-postgres, creating several models which we will explain below.

I have integrated Meltano with Docker and AWS services to automate the project. Each time an image is pushed to GitHub, a CodeBuild is triggered, which uses the project's buildspec file to create a Docker image and push it to ECR. ECR (Elastic Container Registry) is a fully managed Docker container registry that makes it easy to store, manage, and deploy Docker container images. This ECR image is then used by ECS, where we create a service that runs a task executing the Docker image and running the entire pipeline. This service also includes autoscaling. The Auto Scaling service automatically adjusts the number of EC2 instances in the Auto Scaling group according to the specified conditions, ensuring that the application has the right resources at the right time.

The project has different environments: development, staging, and production. We have different .env files with the credentials for each environment. These credentials are stored in the project, and each time we push to the corresponding branch of each environment, the relevant CodeBuild is triggered. It takes the .env file of the environment and passes it as arguments in the Docker build, constructing the image with the environment-specific credentials and then pushing that image to ECR. This is why we have 3 different CodeBuilds, 3 different ECR repositories, and 3 different ECS clusters, one for each environment. However, we only have one buildspec which receives the environment as a parameter from CodeBuild and performs the build with the corresponding .env file. Separating different environments is a best practice because it allows for better management and isolation of development, testing, and production resources, ensuring that changes can be tested thoroughly before being deployed to production.

## Components
- **CodeBuild**: Builds the project image and pushes it to the corresponding ECR based on the environment.
- **ECR (Elastic Container Registry)**: Stores the Docker images for the Meltano project.
- **GitHub**: Hosts the source code for the project.
- **SpaceX-API**: The data source providing information about SpaceX launches.
- **ECS Cluster (Elastic Container Service)**: Manages the deployment of the Meltano project container.
- **Task Definition**: Defines the task that pulls images from ECR and uses them in the service deployment.
- **Service**: Manages the execution of tasks.
- **Auto Scaling Group**: Manages the EC2 instances used by the ECS cluster.

## Implementation with Meltano


![ELT_meltano drawio](https://github.com/gonza0305/test_readme/assets/15968266/84e8ee88-1274-4a34-a083-d70af6ef091c)<br>


### Custom Extractor for SpaceX-API

Created a custom extractor named tap-spacexapi, which has two streams: launch and launches. These streams read from the endpoint "https://api.spacexdata.com/v5" and parse the data using the schema defined in the streams.py file. Currently, we are only fetching the latest launch and the set of launches, but it is possible to fetch much more, such as Starlink satellites, payloads, or ships. This will be described in the future implementations section.

### Configure target-postgres loader

Configured Meltano to load the extracted data into a PostgreSQL database. The loader creates the tables defined in the streams, in this case, creating two tables: launch and launches in the corresponding PostgreSQL environment.

### Data Transformation with dbt

As a transformer, we use dbt-postgres, which works by defining models. These models are transformations applied to the PostgreSQL database, in the tables and schemas defined by the transform/models/source.yml file. We define several models:

1. **Accumulated Satellites**
   - This model accumulates the number of satellites in orbit over time.
   - It calculates the cumulative sum of satellites launched by ordering them by date.
   - This helps to determine the total number of satellites in orbit at any given date.

2. **Crew**
   - This model extracts and transforms crew member data from launches.
   - It identifies crew members and their roles for each launch.
   - While not directly related to the satellite count, it provides useful information for crewed missions.

3. **Launches with Timestamps**
   - This model adds a timestamp to each launch record.
   - It helps to track when each transformation was updated, providing a time-based perspective on launches.
   - This is useful for auditing and tracking the freshness of the data.

4. **Payloads Data**
   - This model unpacks payload data from the launches and assumes each payload represents a satellite.
   - It counts the number of satellites per launch.
   - This data is critical for calculating the cumulative number of satellites in orbit.

5. **Rockets**
   - This model extracts unique rocket identifiers from the launches.
   - It helps to identify the different rockets used in the launches.
   - While not directly related to the satellite count, it provides information on the vehicles used for launching satellites.

6. **Satellites Goal**
   - This model checks if the goal of 42,000 satellites in orbit has been achieved.
   - It uses the accumulated satellite data to determine the date when the goal is met.
   - This directly answers the question of when there will be 42,000 Starlink satellites in orbit.

These models create different tables in the database, one for each model with the data corresponding to executing those queries on the launches table.

We have three databases, one for each environment (dev, staging, and prod). Having different databases for each environment is convenient because it allows for isolated development, testing, and production environments, reducing the risk of changes in one environment affecting the others.

### Docker

To containerize my project, I have used Docker. The project is currently being executed with a docker run on the built image, which connects to another Docker image with the PostgreSQL database. To allow communication between the two Docker containers, they are placed in the same network called meltano-network. Creating a network for the Docker containers allows them to communicate with each other seamlessly.

### How to run in locally

1. Create a Docker network:
   ```bash
   docker network create meltano-network
   
2. Run PostgreSQL container:
   ```bash
   docker run --name meltano_postgres --network meltano-network -p 5433:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=libra4281 -v $(pwd)/initdb:/docker-entrypoint-initdb.d -d postgres
   ```
   initdb:
   The initdb directory contains the initial SQL scripts to set up the databases for each environment
   The script create a database for each environment in the PostgreSQL container.

3. Build Docker images for each environment:
   ```bash
   docker build --build-arg ENV_FILE=.env.dev -t meltano-demo-project:dev .
   docker build --build-arg ENV_FILE=.env.staging -t meltano-demo-project:staging .
   docker build --build-arg ENV_FILE=.env.prod -t meltano-demo-project:prod .
   
4. Run the Meltano pipeline in each environment:
   ```bash
   docker run --network meltano-network meltano-demo-project:dev run tap-spacexapi target-postgres dbt-postgres:run
   docker run --network meltano-network meltano-demo-project:staging run tap-spacexapi target-postgres dbt-postgres:run
   docker run --network meltano-network meltano-demo-project:prod run tap-spacexapi target-postgres dbt-postgres:run

   
## Considerations
- Concerning the data, I have considered that each payload_id represents a satellite.
- As I am not the owner of the repository I can't implement webhooks to trigger the build images in CodeBuild, so I am currently triggering them manually, as you can see below:
![Screenshot 2024-05-19 at 22 17 35](https://github.com/gonza0305/test_readme/assets/15968266/e45a4539-255c-4719-81a8-508ea889dd71)
![Screenshot 2024-05-19 at 22 17 06](https://github.com/gonza0305/test_readme/assets/15968266/7b917c6d-faac-4b9a-94cf-776016604e19)


## Challenges Encountered

- Incompatibility between Python 3.12 and libraries like Mashumaro, which are part of dbt: The solution was to downgrade the Python version.
- Learning Curve to Learn Meltano: The learning curve to understand Meltano, its architecture, and its components.
- Version and Library Errors When Building the Docker Image: The solution was to manually install gcc, python3-dev, libpq-dev, and git through the Dockerfile.

## Future Implementations and Improvements

- **Include PostgreSQL RDS in AWS**: Currently, the database is running in a Docker image, but it would be better to use RDS with PostgreSQL on AWS. This would provide a highly available, scalable, and managed database service, reducing maintenance overhead and improving reliability.
- **Include More Streams**: Adding more streams for different endpoints of the SpaceX API to gather more data and create various models.
- **Use Meltano Superset Utility**: To analyze the data and create charts and dashboards, providing better insights and visualizations.


## Feedback

I found the project very enjoyable. Learning about a tool that facilitates and automates ETL processes is always helpful. I also liked the freedom provided by the assignment to implement and decide on the necessary models that can assist the Data Analyst team in a real environment. Having all the time to complete it also allowed me to work without pressure. I am very grateful for the opportunity to work on this and to be part of the selection process. If you have any questions, I would be happy to answer them.
