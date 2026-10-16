# CLO835 Assignment 1: Two-Tier Containerized Web App

A Flask web tier and a MySQL data tier, each in its own Docker container, connected over a custom Docker bridge network.

## Docker Hub images
- Web app: https://hub.docker.com/r/aneeta3021/webapp (tags `v1`, `v2`)
- MySQL: https://hub.docker.com/r/aneeta3021/mysql-db (tag `v1`)

## Repository contents
| File | Purpose |
|---|---|
| `Dockerfile` | Web tier image (Ubuntu 26.04, Python venv, Flask app on port 8080) |
| `Dockerfile_mysql` | MySQL 8.0 image that loads `mysql.sql` on first start |
| `compose.yaml` | The whole stack on one bridge network |
| `app.py`, `templates/` | The Flask application |
| `mysql.sql` | Creates the `employees` database and sample rows |

## Build and push
```bash
docker build -t aneeta3021/webapp:v1 -f Dockerfile .
docker build -t aneeta3021/mysql-db:v1 -f Dockerfile_mysql .
docker login -u aneeta3021        # use a Docker Hub access token as the password
docker push aneeta3021/webapp:v1
docker push aneeta3021/mysql-db:v1
```

## Pull and run with docker run
```bash
docker pull aneeta3021/webapp:v1
docker pull aneeta3021/mysql-db:v1

docker network create app-net
docker run -d --name mysql --network app-net -e MYSQL_ROOT_PASSWORD=pw aneeta3021/mysql-db:v1

docker run -d --name blue --network app-net -p 8081:8080 \
  -e DBHOST=mysql -e DBPORT=3306 -e DBUSER=root -e DBPWD=pw -e DATABASE=employees -e APP_COLOR=blue \
  aneeta3021/webapp:v1
docker run -d --name pink --network app-net -p 8082:8080 \
  -e DBHOST=mysql -e DBPORT=3306 -e DBUSER=root -e DBPWD=pw -e DATABASE=employees -e APP_COLOR=pink \
  aneeta3021/webapp:v1
docker run -d --name lime --network app-net -p 8083:8080 \
  -e DBHOST=mysql -e DBPORT=3306 -e DBUSER=root -e DBPWD=pw -e DATABASE=employees -e APP_COLOR=lime \
  aneeta3021/webapp:v1
```

Test the database connection with a second MySQL container on the same network:
```bash
docker run -it --rm --network app-net mysql:8.0 mysql -h mysql -uroot -ppw -e "use employees; select * from employee;"
```

Test the apps from the host, then ping between containers by name:
```bash
curl http://localhost:8081
curl http://localhost:8082
curl http://localhost:8083
docker exec blue ping -c 3 pink
docker exec blue ping -c 3 lime
```

## Why three apps can all listen on port 8080
Each container has its own network namespace and its own IP address on the bridge network, so port 8080 inside blue, pink and lime never conflict. Only the host's ports must be unique. The mappings `-p 8081:8080`, `-p 8082:8080` and `-p 8083:8080` forward three different host ports to port 8080 in three different containers.

## Docker Compose
```bash
docker compose up -d
docker compose ps
docker compose down -v
```
`compose.yaml` defines the `mysql-db`, `blue`, `pink` and `lime` services on a user-defined bridge network called `app-net`. The web services reach the database by the service name `mysql-db`, and they wait for the MySQL healthcheck before they start.

## Updating the app
1. Make the change on a branch and open a pull request into `master`.
2. After the merge, pull `master`, rebuild with a new tag (for example `v2`), and push it to Docker Hub.
3. Remove the old container and run it again from the new image.

## Notes on credentials
No Docker Hub password or access token is stored in this repository. The `pw` value used above is only the password of a throwaway local MySQL container for this assignment.
