FROM python:3.9

RUN pip install mysql-connector-python
RUN pip install jupyterlab
RUN pip install psycopg2-binary
RUN pip install pymongo

# Install OpenJDK-8
RUN apt-get update 
RUN apt-get -y install default-jre

WORKDIR /user/app