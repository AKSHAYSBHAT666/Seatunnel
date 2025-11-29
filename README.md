STEP 1 — Start all containers

Inside your project folder (where your docker-compose.yml exists):

docker compose up -d


Check running containers:

docker ps


You SHOULD see:

oracle1 → port 1521

oracle2 → port 1522

seatunnel

STEP 2 — Wait for Oracle to fully initialize (~30–60 sec)

Check logs:

docker logs oracle1 --tail 50
docker logs oracle2 --tail 50


When you see DATABASE IS READY TO USE!, continue.

STEP 3 — Connect to oracle1 and create a table + insert sample data

Open SQL*Plus inside oracle1:

docker exec -it oracle1 sqlplus system/OraclePwd1@FREEPDB1


Run:

CREATE TABLE payments (
    id NUMBER PRIMARY KEY,
    amount NUMBER,
    status VARCHAR2(20)
);

INSERT INTO payments VALUES (1, 1000, 'PAID');
INSERT INTO payments VALUES (2, 2500, 'FAILED');
INSERT INTO payments VALUES (3, 900,  'PROCESSING');

COMMIT;
EXIT;

STEP 4 — Create SeaTunnel job file (inside host machine)

Path on host:

./seatunnel/jobs/oracle_payments.conf


Content:

env {
  parallelism = 1
  job.mode = "BATCH"
}

source {
  Jdbc {
    url = "jdbc:oracle:thin:@oracle1:1521/FREEPDB1"
    driver = "oracle.jdbc.OracleDriver"
    user = "SYSTEM"
    password = "OraclePwd1"
    query = "SELECT id, amount, status FROM payments"
  }
}

sink {
  Jdbc {
    url = "jdbc:oracle:thin:@oracle2:1521/FREEPDB1"
    driver = "oracle.jdbc.OracleDriver"
    user = "SYSTEM"
    password = "OraclePwd2"
    table = "payments"
    generate_sink_sql = true
    database = "FREEPDB1"
    primary_keys = ["id"]
  }
}


Make sure the JDBC driver exists at:

./seatunnel/libs/ojdbc8.jar

STEP 5 — Open a shell inside the SeaTunnel container

Since your image already contains the full SeaTunnel installation:

docker exec -it seatunnel bash


You will now be inside:

/opt/seatunnel

STEP 6 — Run SeaTunnel job in LOCAL MODE (IMPORTANT)
bin/seatunnel.sh --config jobs/oracle_payments.conf -m local


Expected output:

Job submitted...
Job completed successfully!

STEP 7 — Verify data reached Oracle2

Connect to oracle2:

docker exec -it oracle2 sqlplus system/OraclePwd2@FREEPDB1


Check table:

SELECT * FROM payments;