# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Plants By WebSphere is a Jakarta EE 10 eCommerce application modernized for cloud deployment. This is a modified version of the IBM WebSphere sample app, originally built on Java EE 6 and upgraded to Jakarta EE 10. The modernization was based on IBM Cloud Transformation Advisor recommendations for migration to WebSphere Liberty and containerized environments (OpenShift, IBM Cloud Private, IBM Cloud Kubernetes Service).

The application is a fictional online plant store that supports user registration and product purchases, backed by a relational database (MySQL/MariaDB).

## Architecture

### Maven Structure

The project uses a simplified single-module Maven structure:

- **pbw-web**: Web application WAR containing JPA entities, servlets, JSF pages, and utilities

All code has been consolidated into the WAR module for simpler deployment and maintenance. The previous multi-module structure (pbw-lib JAR and pbw-ear EAR) has been eliminated as part of the Jakarta EE 10 modernization.

### Runtime Architecture

- **Application Server**: WebSphere Liberty (based on Open Liberty) running Jakarta EE 10 features
- **Java Version**: Java 21 (OpenJ9)
- **Jakarta EE Features**: Faces 4.0, Servlet 6.0, JPA 3.1, CDI 4.0, Enterprise Beans 4.0, Bean Validation 3.0
- **Database**: MariaDB or MySQL accessed via JDBC with JPA 3.1 for ORM
- **Configuration**: Liberty server.xml in `wlp/config/` defines features, datasources, and JNDI resources
- **Database Connection**: Uses environment variables (DB_HOST, DB_PORT, DB_USER, DB_PASSWORD) injected via Kubernetes secrets

### Key Java Packages

- `com.ibm.websphere.samples.pbw.jpa.*`: JPA entities (Customer, Order, Inventory, etc.)
- `com.ibm.websphere.samples.pbw.war.*`: Servlet beans (ProductBean, AccountBean, etc.)
- `com.ibm.websphere.samples.pbw.ejb.*`: Enterprise beans (CatalogMgr, CustomerMgr, ShoppingCartBean, etc.)
- `com.ibm.websphere.samples.pbw.utils.*`: Utility classes (Util, ListProperties)
- `pagecode.*`: JSF page backing beans

### Container & Deployment

- **Docker**: Base image `icr.io/appcafe/websphere-liberty:full-java21-openj9-ubi-minimal` with MySQL JDBC driver
- **Liberty Config**: Defines Jakarta EE 10 features, JTA/non-JTA datasources
- **Final Artifact**: `plants-by-websphere-jakarta-mysql.war` deployed to Liberty's apps directory

## Common Development Commands

### Running Locally with Docker Compose (Recommended for Development)

```bash
# Build the application
mvn clean package

# Start all services (MariaDB + Liberty)
docker-compose up -d

# View logs
docker-compose logs -f liberty

# Stop all services
docker-compose down

# Stop and remove volumes (clean database)
docker-compose down -v
```

**Application will be available at:**
- http://localhost:9080/ (main application)
- http://localhost:9443/ (HTTPS)

**Database credentials (configured in docker-compose.yml):**
- Host: localhost:3306
- Database: plantsdb
- User: dbuser
- Password: dbpass

### Database Initialization

On first startup, the database must be populated with sample data:

```bash
# After starting the application, populate the database
curl "http://localhost:9080/servlet/AdminServlet?admintype=populate"

# Or navigate to this URL in a browser
# http://localhost:9080/servlet/AdminServlet?admintype=populate
```

**Important**: The populate function requires `pbw.properties` to be present in `pbw-web/src/main/resources/`. This file contains all the inventory, customer, and supplier data used to initialize the database.

You can verify the database has been populated:

```bash
docker exec plantsdb mysql -udbuser -pdbpass plantsdb -e "SELECT COUNT(*) FROM INVENTORY;"
# Should return 43 items
```

### Building the Application

```bash
# Build the WAR from root directory
mvn clean package

# The final WAR artifact: pbw-web/target/plants-by-websphere-jakarta-mysql.war
```

### Building the Docker Image

```bash
# Build requires the WAR in pbw-web/target/ directory
docker build -t pbw-mariadb-web:1.0.0 .
```

### Running Tests

```bash
# Run tests
mvn test
```

### Deployment Scripts (For Kubernetes/OpenShift)

```bash
# Create Kubernetes secrets for database credentials
./scripts/create-secrets.sh

# Generate Kubernetes deployment manifests (interactive, prompts for namespace)
./scripts/create-deployment.sh

# Test database connection using secrets file
./scripts/test-db-connection.sh <db-validation-service-url>
```

## Deployment Targets

This app supports multiple deployment scenarios:

- **OpenShift**: Templates in `openshift/templates/` for S2I, Docker, and CI/CD workflows
- **IBM Cloud Private/Kubernetes**: Manifests in `k8s/` and Jenkinsfiles for automated pipelines
- **Helm Charts**: Chart in `chart/pbw-liberty-mariadb/` for Kubernetes deployment

## Critical Configuration Files

- `wlp/config/server.xml`: Liberty feature manager, datasource definitions (MySQL/MariaDB), JNDI names
- `pbw-web/src/main/resources/META-INF/persistence.xml`: JPA persistence unit configuration
- `pbw-web/src/main/webapp/WEB-INF/web.xml`: Web app deployment descriptor
- `Dockerfile`: Multi-stage build copying MySQL driver and server.xml into Liberty image

## Environment Variables

The application expects these environment variables (typically from Kubernetes secrets):

- `DB_HOST`: Database server hostname
- `DB_PORT`: Database port (typically 3306)
- `DB_USER`: Database username
- `DB_PASSWORD`: Database password

Database name is hardcoded to `plantsdb` in server.xml.

## CI/CD Pipelines

Multiple Jenkinsfiles support different environments:

- `Jenkinsfile`: Base Kubernetes pipeline
- `Jenkinsfile.ocp`: OpenShift-specific pipeline
- `Jenkinsfile.iks`: IBM Kubernetes Service pipeline
- `Jenkinsfile.ext-icp`: IBM Cloud Private external registry

Pipelines use Maven, Docker, and kubectl containers in Jenkins pod templates.

## Liberty HTTP Endpoints

- Application Port: 9080 (HTTP)
- HTTPS Port: 9443
- Context Root: `/` (root context)

## Modernization Notes

This application has undergone significant modernization from its original Java EE 6 implementation:

### Migration Path

- **Original**: Java EE 6 on Java 6/8
- **Current**: Jakarta EE 10 on Java 21

### Key Upgrades

- **Java**: Upgraded from Java 8 to Java 21 with OpenJ9 JVM
- **Application Server**: Upgraded to WebSphere Liberty 25.x with Jakarta EE 10 support
- **Jakarta EE Features**: Migrated from Java EE 6 to Jakarta EE 10
  - JSF 2.x → Faces 4.0
  - JPA 2.1 → JPA 3.1
  - Servlet 3.x → Servlet 6.0
  - CDI 1.x → CDI 4.0
  - EJB 3.x → Enterprise Beans 4.0
- **Namespace Migration**: All `javax.*` packages migrated to `jakarta.*`
- **Module Structure**: Simplified from multi-module (lib/web/ear) to single WAR deployment
- **Maven Plugins**: Updated to modern versions (maven-war-plugin 3.4.0, maven-compiler-plugin 3.13.0)
- **Liberty Base Image**: Uses `icr.io/appcafe/websphere-liberty:full-java21-openj9-ubi-minimal`

### Database Support

Can switch between MySQL and MariaDB by uncommenting respective datasource blocks in server.xml.
