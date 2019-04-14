## Introduction
This is contain a small server, which act as our application for deployment given in this [repo](https://github.com/KTH-DD2480/smallest-java-ci). 
## Dependencies to install
* Maven 4.0.0
* Java jre 8

## How to run
To run this repo in Ubuntu -
* mvn package
* java -jar target/CI-jar-with-dependencies.jar

Then in your browser like Firefox type in http://localhost:8000/ to check that the server is reacting

For those who do not have Maven can still run this with java -jar target/CI-jar-with-dependencies.jar , the jar file is already precompiled in "Target" folder



