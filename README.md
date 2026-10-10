
# AWS RDS Read Replica + ElastiCache Redis

A hands-on AWS project demonstrating how to deploy a Python Flask REST API on Amazon EC2, use Amazon RDS MySQL with a Read Replica, and implement Redis caching with Amazon ElastiCache.

The project separates database read and write operations and uses Redis to cache frequently accessed data, reducing repeated database queries.

## Project Overview

The application runs on an Amazon Linux EC2 instance and connects to:

- **Amazon RDS MySQL** — Primary database for write operations.
- **Amazon RDS Read Replica** — Handles read operations.
- **Amazon ElastiCache for Redis** — Caches query results.
- **Amazon EC2** — Hosts the Python Flask application.

## Architecture

```text
                         Client
                            |
                            v
                     Flask REST API
                         (EC2)
                            |
                   +--------+--------+
                   |                 |
                   v                 v
              Redis Cache       RDS MySQL
             (ElastiCache)      Primary DB
                   |                 |
             Cache Hit/Miss          |
                   |                 v
                   |             Read Replica
                   |                 ^
                   +---- Cache Miss-+
```

### Request Flow

#### Read Operations (GET)

1. The client sends a GET request to the Flask API.
2. The application checks Redis for cached data.
3. If the data exists, the cached result is returned.
4. If there is a cache miss, the application queries the RDS Read Replica.
5. The result is stored in Redis with a TTL of 90 seconds.
6. The result is returned to the client.

#### Write Operations (POST, PUT, DELETE)

1. The client sends a write request to the Flask API.
2. The application connects to the primary RDS database.
3. The database operation is executed.
4. Relevant Redis cache keys are deleted to prevent stale results from being served.

## AWS Services Used

| AWS Service | Purpose |
|---|---|
| Amazon EC2 | Hosts the Python Flask application |
| Amazon RDS for MySQL | Stores application data |
| RDS Read Replica | Handles database read queries |
| Amazon ElastiCache for Redis | Caches frequently accessed results |
| Amazon VPC | Provides network isolation |
| Security Groups | Control traffic between AWS resources |

## Technologies Used

- Python 3
- Flask
- Flask-CORS
- MySQL
- `mysql-connector-python`
- Redis
- Amazon Linux
- Git and GitHub
- Linux command line

## Project Structure

```text
Elastic-cache/
├── app.py
├── cache.py
├── test.sql
├── requirements.txt
└── README.md
```

*Adjust the structure above to match your actual repository.*

## Database Setup

The `test.sql` file creates the `dev` database, creates the `users` table, and inserts four sample users.

### `test.sql`

```sql
CREATE DATABASE IF NOT EXISTS dev;
USE dev;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE
);

INSERT INTO users (name, email) VALUES
('John Doe', 'john@example.com'),
('Alice Smith', 'alice@example.com'),
('Bob Johnson', 'bob@example.com'),
('Eve Adams', 'eve@example.com');
```

### Database Schema

| Column | Data Type | Description |
|---|---|---|
| `id` | INT, AUTO_INCREMENT | Primary key |
| `name` | VARCHAR(100) | User's name |
| `email` | VARCHAR(100), UNIQUE | User's email address |

### Import the SQL File

After creating the RDS database and configuring connectivity from EC2, install the MySQL-compatible client tools:

```bash
sudo yum install mariadb105-server -y
```

Import the SQL file into RDS:

```bash
mysql -h <RDS_ENDPOINT> -u admin -p < test.sql
```

Enter the database password when prompted.

This executes `test.sql` and creates the `dev` database and its sample data on the RDS instance you connect to.

**Important:** If the script is run against the primary database, the changes should replicate to the RDS Read Replica after replication catches up. Do not initialize the same table independently on the replica.

## Project Setup

### 1. Launch an EC2 Instance

- Launch an Amazon Linux EC2 instance.
- Configure SSH access using a key pair.
- Connect to the instance.

### 2. Create an RDS MySQL Database

- Create the primary RDS MySQL instance.
- Configure database access from the EC2 security group on port `3306`.
- Use the database name `dev`.

### 3. Create an RDS Read Replica

- Create a read replica of the primary database.
- Copy the primary and reader endpoints.
- Configure the application to use the primary endpoint for writes and the reader endpoint for reads.

### 4. Create an ElastiCache Redis Cluster

- Create an ElastiCache Redis-compatible cache.
- Copy its endpoint.
- Configure connectivity from EC2 to the cache on port `6379`.
- Enable TLS in the client when required by your cache configuration.

### 5. Install Required Packages

Update the Amazon Linux instance and install the required tools:

```bash
sudo yum update -y
sudo yum install git -y
sudo yum install python3 python3-pip -y
sudo yum install mariadb105-server -y
```

### 6. Clone the Repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd <YOUR_PROJECT_DIRECTORY>
```

### 7. Install Python Dependencies

The `requirements.txt` file should contain:

```text
Flask
flask-cors
mysql-connector-python
redis
```

Install the dependencies:

```bash
python3 -m pip install -r requirements.txt
```

### 8. Configure Database and Redis Endpoints

Configure the following values in the application:

| Setting | Description |
|---|---|
| RDS writer endpoint | Used for POST, PUT, and DELETE |
| RDS reader endpoint | Used for database reads on cache misses |
| Database name | `dev` |
| Database username | Your configured RDS username |
| Database password | Store securely; do not hardcode |
| Redis endpoint | Your ElastiCache endpoint |
| Redis port | `6379`, if configured for this endpoint |
| Cache TTL | `90` seconds in the supplied application code |

For improved security, use environment variables or AWS Secrets Manager for credentials.

### 9. Run the Application

If the API implementation is in `app.py`:

```bash
python3 app.py
```

If running the separate caching demonstration script:

```bash
python3 cache.py
```

Run the command corresponding to the application you want to start. The supplied Flask API code listens on port `5000`.

## API Endpoints

The Flask API implements the following routes:

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/users` | Retrieve all users |
| GET | `/users/<user_id>` | Retrieve one user |
| POST | `/users/add` | Add a user |
| PUT | `/users/update/<user_id>` | Update a user |
| DELETE | `/users/delete/<user_id>` | Delete a user |

## API Testing with cURL

Run these commands from the EC2 instance or another machine that can access the Flask API.

### GET — Retrieve All Users

```bash
curl http://localhost:5000/users
```

### GET — Retrieve One User

```bash
curl http://localhost:5000/users/1
```

### POST — Add a User

```bash
curl -X POST http://localhost:5000/users/add \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie Brown","email":"charlie@example.com"}'
```

### PUT — Update a User

```bash
curl -X PUT http://localhost:5000/users/update/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"John Updated","email":"john.updated@example.com"}'
```

### DELETE — Delete a User

```bash
curl -X DELETE http://localhost:5000/users/delete/1
```

## Redis Caching Strategy

This project demonstrates the **cache-aside pattern**.

- **Cache hit:** Return data from Redis without querying RDS.
- **Cache miss:** Query the RDS Read Replica and cache the result.
- **TTL:** Cache entries expire after 90 seconds by default.
- **Cache invalidation:** Write operations remove the corresponding cached entries.

This approach can reduce repeated database reads and improve response times for frequently requested data.

## Security Considerations

- Restrict SSH access to trusted IP addresses.
- Allow MySQL traffic only from authorized security groups.
- Allow Redis traffic only from authorized application resources.
- Never expose database or Redis ports directly to the public internet.
- Do not commit passwords, access keys, or other secrets to GitHub.
- Use least-privilege IAM policies and secure secret storage.
- Use TLS where supported and required by the ElastiCache configuration.

## What I Learned

Through this project, I practised:

- Deploying a Python Flask API on Amazon EC2.
- Connecting an application to RDS MySQL.
- Separating read and write database operations.
- Configuring and using an RDS Read Replica.
- Integrating Redis caching with Amazon ElastiCache.
- Implementing cache expiration and invalidation.
- Testing REST APIs using cURL.
- Configuring AWS networking and security groups.
- Troubleshooting database and cache connectivity.

## Future Improvements

- Deploy the application behind an Application Load Balancer.
- Configure Auto Scaling for the application tier.
- Add CloudWatch metrics, logs, and alarms.
- Store credentials in AWS Secrets Manager.
- Add automated tests and CI/CD using GitHub Actions.
- Improve API validation, exception handling, and cache consistency.

## Author

**Aman Bhivgade**

Building hands-on projects while learning AWS, Cloud Computing, and DevOps.

---

**Topics:** AWS, EC2, RDS, MySQL, Read Replicas, ElastiCache, Redis, Python, Flask, REST API, Linux, Cloud Computing, DevOps.
