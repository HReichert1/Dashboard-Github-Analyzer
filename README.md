# Dashboard-Github-Analyzer

This project is about analyzing activities on Github for the Webeet.io organization in the context of a 8-weeks internship.
I was given the task to create a fully interactive dashboard (allows users to filter, drill down, and explore data (using parameters, filters, and actions).

The dashboard should answer the following questions:

- Give insights about teams' activities: who is doing what? Are there different behavious in different teams?
- Provide different metrics for different teams
  The dashboard should focus on Focus on metrics that provide actionable insights, such as:
  - Comparison of closure speed across different reports
    Days to closure by different repositories, % closed within n days (n to be selected)
  - Performance comparisons between reviewers
  - Commits per pull request across different teams
  - Repo-specific metrics (e.g., which repo has more issues)
- Explore patterns in event behavior across weekdays, repos and team members
- Consider showing how event distribution changes over time for individuals or repos
- Allow selection of individual event types for detailed analysis
- Analysis by assignee and reviewer

As discussions are still ongoing which visualization tool has to be used I decided to utilize Tableau which then can be easily mapped to the chosen tool.

Working steps:
## 1. Read the provided files from Github and familiarize with the structure and content

4 PostgreSQL datasets were allocated to me:

- **github_repositories:**
  contains information about the different repositories available at Webeet.io
- **github_pull_requests:**
  contains information about pull requests on Github for the Webeet.io organization
- **github_issues**
  contains information about issues created on Github for the Webeet.io organization
- **github_events:**
  contains information about events (activities) on Github for the Webeet.io organization

  **ERD Diagram:**
  

<img width="823" height="1294" alt="image" src="https://github.com/user-attachments/assets/7d48be5a-0fb4-4be0-b38c-6b614d456777" />


   In order to be able to access the data in the database it is necessary to open a tunnel to the database on AWS by applying a detailed procedure. Once the tunnel is open the data can be accessed.

   In a Jupiter notebook using Python the following code was created:
```
# Connect to postgres DB with credentials
user_name='testing'
password='password'
 
# Conection
host = 'localh'
port = '1234'
database = 'task_analyzer'
schema='github_source_data'

#connection to db after having opened tunnel
engine = create_engine(f'postgresql+psycopg2://{user_name}:{password}@{host}:{port}/{database}')

# SQL query to select table, eg. github_events
query = f"""
SELECT * from github_source_data.github_events 
"""

# Execute the query, e.g. for events
with engine.connect() as conn:
    df_events= pd.read_sql(text(query), conn)
    #conn.commit()  # commit the transaction
df_events.info()
```
### 1.1 Create CSV files for loading into Tableau Desktop
```
# store as CSV:
df_events.to_csv("/Users/name/Documents/Jupiter Notebooks/events.csv", index=False)  
```
This steps have to be repeated for all 4 tables

in Tableau these tables can be connected via the repo_id which is available in all tables

## 2. Create additional tables for issues and pull requests with split authors and assignees (called "flat" tables)

In the 2 tables for issues and pull requests the authors and assignees are stored in one field where the names are concatenated.
In order to be able to perform the requested analysis it is required to split the names and store them in individual raws.

### 2.1 Issues
In issues the assignees are stored in a field as follows:

[NamedUser(login="name1"), NamedUser(login="name2"), NamedUser(login="name3"), NamedUser(login="name4")] 

The following Python code is used to split and write the names in individual rows:
```
# split assignees in issues


# extract all login names from assignees
import re
df_issues_copy['login_names'] = df_issues_copy['assignees'].apply(
    lambda lst: [re.search(r'login="([^"]+)"', x).group(1) for x in lst if re.search(r'login="([^"]+)"', x)]
)
df_issues_copy['num_logins'] = df_issues_copy['login_names'].apply(len)      # count number of logins per issue
df_flat= df_issues_copy.explode('login_names')                               # explode the list into separate rows
```

### 2.2 Pull requests
In pull requests the different actors are stored in a field as follows:

[{'actor': 'name1', 'created_at': '2025-07-07 09:26:14+00:00', 'event_type': 'review_requested'}, {'actor': 'name1', 'created_at': '2025-07-07 09:56:46+00:00', 'event_type': 'head_ref_force_pushed'}, {'actor': 'name2', 'created_at': '2025-07-07 10:21:47+00:00', 'event_type': 'merged'}, {'actor': 'name2', 'created_at': '2025-07-07 10:21:47+00:00', 'event_type': 'closed'}]


```
# extract all login names from pr_actions
import ast
def extract_actors_safe(entry):
    """Safely extract actor names from a string or list of PR actions."""
    try:
        # Convert string to Python list if needed
        if isinstance(entry, str):
            data = ast.literal_eval(entry)
        elif isinstance(entry, list):
            data = entry
        else:
            return []

        if not isinstance(data, list):
            return []

        # Extract all 'actor' values from valid dicts
        return [d.get('actor') for d in data if isinstance(d, dict) and 'actor' in d]
    except Exception as e:
        print("⚠️ Problem parsing entry:", entry, "| Error:", e)
        return []

# ✅ Apply the function to pr_actions
df_pull_copy['actor_names'] = df_pull_copy['pr_actions'].apply(extract_actors_safe)
df_pull_copy['num_actors'] = df_pull_copy['actor_names'].apply(len)

# Optional: expand into individual rows
df_flat_pull = df_pull_copy.explode('actor_names', ignore_index=True)
```
Both created files are transformed to CSV and stored on local drive as described above.

### 2.3 Combined file with all types of Github files

One file was created with the 3 Github types Events, Pull Requests and issues in order to be able to compare contributors across teams and types.

In order to be able to do this the 3 files had to be prepared. Only 15 columns are kept in this file with an additional columns used as identifier for the types:

Example for Pull Requests:
```
df_flat_pull_copy['type']= 'Pull Request'
df_flat_events_copy['type']= 'Activity'
df_flat_copy['type']= 'Issue'
 

Data columns (total 15 columns):
 #   Column          Non-Null Count  Dtype              
---  ------          --------------  -----              
 0   type            4887 non-null   object             
 1   names           4616 non-null   object             
 2   event_type      1996 non-null   object             
 3   created_at      4887 non-null   datetime64[ns, UTC]
 4   repo_id         4887 non-null   object             
 5   name_num        2891 non-null   float64            
 6   state           2891 non-null   object             
 7   created_by      637 non-null    object             
 8   closed_at       2316 non-null   datetime64[ns, UTC]
 9   closed_by       271 non-null    object             
 10  comments_count  637 non-null    float64            
 11  author_login    2254 non-null   object             
 12  is_merged       2254 non-null   object             
 13  commits_count   2254 non-null   float64            
 14  repo_full_name  2254 non-null   object            
```
Column "names" contains all contributors in separate rows.

This combined table was transformed to CSV and stored on local drive.

## 3. Load the data into Tableau Desktop

Schema of tables in Tableau:

<img width="971" height="378" alt="image" src="https://github.com/user-attachments/assets/9291e44b-e954-4eed-88d4-75e0058351dc" />



 
All Tables are linked via Repo_id, the "flat" tables in addition via assignees columns or author_names 

### 3.1 Dashboard story
<img width="1318" height="1022" alt="image" src="https://github.com/user-attachments/assets/0ad7acd9-f98f-406d-9a45-7961fd57b3b8" />

### 3.2 Pull Request by Status Analysis
<details>
<summary><strong>Merged Pull Requests (PR) </strong></summary>
<img width="1444" height="1111" alt="image" src="https://github.com/user-attachments/assets/6251714f-7e5b-4caa-8613-fe5ea6d81d9f" />


Interactivity 
- on Merged or Closed Pull Requests on Top visualizations
- <img width="214" height="89" alt="image" src="https://github.com/user-attachments/assets/1d6417a6-e52b-4af3-87b3-538eb27d986c" />

- on bottom visualization
  - show Repository yes/no
    - <img width="139" height="87" alt="image" src="https://github.com/user-attachments/assets/1f08b64e-d1dc-4d98-86a7-348e34f2b153" />

  - decide on x-axis time granularity (date, weekday, month or year)
    - <img width="143" height="137" alt="image" src="https://github.com/user-attachments/assets/9a33fd1d-0351-4e8e-bf75-90f7e6acbc02" />

  - decide which metric to visualize
    - <img width="243" height="171" alt="image" src="https://github.com/user-attachments/assets/82d29750-005c-468a-bcdf-e4f78e4db29b" />
 

</details>

---

### 3.3 Pull Requests by Days to Closure Analysis

<details>
<summary><strong>Pull Requests by Days to Closure Analysis</strong></summary>
<img width="1347" height="1123" alt="image" src="https://github.com/user-attachments/assets/0e5c6744-21f7-4ed4-994f-dd41d2adc3c7" />

Interactivity 
- on Merged or Closed Pull Requests for left side visualizations
- show Repository yes/no for left side visualizations
</details>

---

### 3.4 Pull Requests by Contribution Analysis  
<details>
<summary><strong>Pull Requests by Contribution Analysis</strong></summary>

<img width="1343" height="1112" alt="image" src="https://github.com/user-attachments/assets/8bbacf95-5f3a-4199-9be4-27ab2aa935e0" />

Interactivity
- left visualization
  - Filter to select Repositories
    - <img width="279" height="631" alt="Bildschirmfoto 2025-11-05 um 10 19 53" src="https://github.com/user-attachments/assets/74cf8ad8-62bd-4cad-a265-432ae4d690fa" />
  - show Repository yes/no
    - <img width="602" height="929" alt="image" src="https://github.com/user-attachments/assets/536ebb41-f732-422a-8aa6-74703129ecc6" />
- right visualization
  - Select contributors
    - <img width="172" height="660" alt="image" src="https://github.com/user-attachments/assets/19c118e9-862a-4ed5-ad90-79a1467c66fe" />
  - Select to show repository or contributors
    - <img width="201" height="94" alt="image" src="https://github.com/user-attachments/assets/12bbb5ac-a6a9-48f2-90b2-9bf18e67d861" />
  - Select repository in case repository was chosen in show repository or contributors
    - <img width="283" height="634" alt="image" src="https://github.com/user-attachments/assets/401815b1-5d7e-4f5c-bdca-bd378afb9ea3" />
    - <img width="695" height="464" alt="image" src="https://github.com/user-attachments/assets/32387079-743a-475c-97b3-e40800c7148e" />
  - Select contributor in case contributors where chosen
    - <img width="136" height="645" alt="image" src="https://github.com/user-attachments/assets/e951a405-79ca-49f1-8862-e5202b25bca9" />
    -<img width="664" height="850" alt="image" src="https://github.com/user-attachments/assets/f050b0b0-2f51-4117-8662-a11d8268f2da" />

</details>

---
### 3.5 Activities
<details>
<summary><strong>Activities</strong></summary>

<img width="1313" height="1095" alt="image" src="https://github.com/user-attachments/assets/8cbfb997-f924-41fb-a9d1-086ef1e3e5cd" />

Interactivity
- Filter to select reposistories
- Filter contributors
- Show contributors
- Show repository yes/no

  </details>

---

### 3.6 Issues by Status Analysis

<details>
<summary><strong>Issues by Status Analysis</strong></summary>

<img width="1313" height="1095" alt="image" src="https://github.com/user-attachments/assets/13d5a9bb-3ba0-4896-a0a4-c68320637275" />

  </details>

---

### 3.7 Issues by Days to Closure Analysis

<details>
<summary><strong>Issues by Days to Closure Analysis</strong></summary>

<img width="1313" height="1095" alt="image" src="https://github.com/user-attachments/assets/df44de7c-486f-48aa-98c9-b7f5fefa97ff" />
 </details>

---

### 3.8 Analysis across Github Types
<details>
<summary><strong>Analysis across Github Types</strong></summary>

<img width="1313" height="1095" alt="image" src="https://github.com/user-attachments/assets/83fddcc9-f0d4-43bd-b5fa-4328e6c04016" />

</details>
