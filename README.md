<h1>Github commits collector</h1>

The purpose of this project is to create a simple ETL pipeline that extracts commits data from Github API and loads it in a local database<br />
IMPORTANT: Because we don't have an effective way to detect modified published commits in a repository only with the provided API
```/repos/{owner}/{repo}/commits``` (except a full load and delete old records). We can ignore that for now and assume Github commits for a particular repository is append-only.<br /><br />
<img width="789" alt="image" src="https://github.com/user-attachments/assets/839e154e-2d59-4dc7-8396-7d834e9ce0cc" />

The system for this pipeline includes:
- A backend API written in Golang using Gin framework, this will be used to extract data from the API and produce the messages to Kafka
- A Kafka broker with one topic: github_commits
- A backend API written in Java using Spring Boot framework, this will be used to receive messages from Kafka topic and create new records in the raw table of the destination database
- A destination database (Postgres) with three tables: collection_jobs (to manage collection history), commit_events (raw table for the commits data, this table may contain duplicate commits), commits (cleaned table that contains final data)

We can run the pipeline either manually (through the API endpoint provided by the first service) or by a cron job (for the scope of this project, we will not implement it. In a real-world context, we can schedule it using an orchestration tool like Jenkins)

Steps to start the services:
- Clone all three repositories in this project
- Start docker services: cd docker -> docker compose up -d
- Start the Postgres database with some scripts (we can use a library like go-migrate or flyway to do the migrations)
```
DROP TABLE IF EXISTS commit_events;
CREATE TABLE IF NOT EXISTS commit_events (
    id UUID PRIMARY KEY,
    sha TEXT NOT NULL,
    committer VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    committed_at TIMESTAMP NOT NULL,
    collected_at TIMESTAMP NOT NULL
);

DROP TABLE IF EXISTS commits;
CREATE TABLE IF NOT EXISTS commits (
     id UUID PRIMARY KEY,
     sha TEXT UNIQUE NOT NULL,
     committer VARCHAR(255) NOT NULL,
     message TEXT NOT NULL,
     committed_at TIMESTAMP NOT NULL
);

DROP TABLE IF EXISTS collection_jobs;
CREATE TABLE IF NOT EXISTS collection_jobs (
    id UUID PRIMARY KEY,
    collected_from TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE OR REPLACE FUNCTION public.upsert_to_commits_table()
    RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO public.commits (id, sha, committer, message, committed_at)
    VALUES (NEW.id, NEW.sha, NEW.committer, NEW.message, NEW.committed_at)
    ON CONFLICT (sha)
        DO UPDATE SET
                      committer = EXCLUDED.committer,
                      message = EXCLUDED.message,
                      committed_at = EXCLUDED.committed_at;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_upsert_to_commits_table
    AFTER INSERT OR UPDATE ON public.commit_events
    FOR EACH ROW
EXECUTE FUNCTION public.upsert_to_commits_table();
```
- Create a new Kafka topic: github_commits
- Start Gin server: cd producer -> go install -> go run cmd/api/main.go (assume that we have Go installed in the machine)
- Start Spring Boot server: cd consumer -> mvn clean install -> mvn spring-boot:run (assume that we have Java and Maven installed in the machine)
- Send a request to server: ```http://localhost:9191/api/v1/collect?from_time=2024-01-01T00:00:00Z```
- Enjoy ^^

<h2>Output</h2>
<img width="1009" alt="image" src="https://github.com/user-attachments/assets/321c6843-d8a2-40ce-b9ee-902d5c2ae28d" />
<img width="1424" alt="image" src="https://github.com/user-attachments/assets/87263717-c99f-4eba-9857-1e78027e60a8" />
<img width="1364" alt="image" src="https://github.com/user-attachments/assets/d76a5ef4-b79a-4529-919a-65b149ea0f40" />

<h2>Answering questions</h2>
1. For the ingested commits, determine the top 5 committers ranked by count of commits and their number of commits.

<img width="681" alt="image" src="https://github.com/user-attachments/assets/b11b5dc5-82c1-442c-ab35-fa963f14f712" />

```
SELECT committer, COUNT(*) AS commit_count
FROM commits
GROUP BY committer
ORDER BY commit_count DESC
LIMIT 5;
```

2. For the ingested commits, determine the committer with the longest commit streak by day.

<img width="1162" alt="image" src="https://github.com/user-attachments/assets/82dbec39-0cc4-4482-b3bc-1ae99c1a994a" />

```
WITH commit_dates AS (
    SELECT DISTINCT committer, committed_at::DATE AS commit_date
    FROM commits
),
     streaks AS (
         SELECT committer, commit_date, commit_date - INTERVAL '1 day' * RANK() OVER (PARTITION BY committer ORDER BY commit_date) AS streak_group
         FROM commit_dates
     ),
     streak_lengths AS (
         SELECT committer, streak_group, COUNT(*) AS streak_length
         FROM streaks
         GROUP BY committer, streak_group
     ),
     longest_streak AS (
         SELECT committer, MAX(streak_length) AS max_streak
         FROM streak_lengths
         GROUP BY committer
     )
SELECT committer, max_streak
FROM longest_streak
ORDER BY max_streak DESC
LIMIT 1;
```

3. For the ingested commits, generate a heatmap of number of commits count by all users by day of the week and by 3 hour blocks.

<img width="1305" alt="image" src="https://github.com/user-attachments/assets/addc6f25-9e40-4eda-b4d5-419596a5d046" />

```
WITH commit_counts AS (
    SELECT
        TO_CHAR(committed_at, 'FMDay') AS day_of_week,
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 1 THEN 1 END) AS "1-3",
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 4 THEN 1 END) AS "4-6",
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 7 THEN 1 END) AS "7-9",
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 10 THEN 1 END) AS "10-12",
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 13 THEN 1 END) AS "13-15",
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 16 THEN 1 END) AS "16-18",
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 19 THEN 1 END) AS "19-21",
        COUNT(CASE WHEN FLOOR(EXTRACT(HOUR FROM committed_at) / 3) * 3 + 1 = 22 THEN 1 END) AS "22-00"
    FROM commits
    GROUP BY day_of_week
)
SELECT *
FROM commit_counts
ORDER BY CASE
             WHEN day_of_week = 'Monday' THEN 1
             WHEN day_of_week = 'Tuesday' THEN 2
             WHEN day_of_week = 'Wednesday' THEN 3
             WHEN day_of_week = 'Thursday' THEN 4
             WHEN day_of_week = 'Friday' THEN 5
             WHEN day_of_week = 'Saturday' THEN 6
             WHEN day_of_week = 'Sunday' THEN 7
             END
    ;
```

