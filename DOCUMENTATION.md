# BUD — technical documentation

This is the setup and implementation reference for BUD, a Windows desktop ticketing application built for the 2024 Bases de Dados course at the University of Aveiro. For the project story and screenshots, start with the [README](README.md).

The application connects directly to SQL Server through ADO.NET. There is no web server or API to start. These instructions describe the checked-in source; they are not a record of a new build or database deployment.

## Setup

### 1. Prepare the tools

The desktop application needs Windows, Visual Studio with the .NET desktop development workload, and the .NET Framework 4.7.2 targeting pack. The [project file](ui/BUD/BUD.csproj) targets that framework; the [solution](ui/BUD/BUD.sln) was saved with Visual Studio 17. Its dependencies are framework assemblies, with no third-party package references.

Use a SQL Server instance and SQL Server Management Studio (SSMS) to create and populate the database. The SQL scripts use T-SQL features such as `GO` batches, table-valued parameters, `IDENTITY`, and stored procedures.

### 2. Create a fresh database

Create an empty database in SSMS, for example `BUD`. The scripts create the `BUD` schema inside the selected database; they do not create a database or select one with `USE`. Check the database selected in each query window before executing a script.

Run the scripts below in order, using a database owner account with the default `dbo` schema. Tables, views, functions, and triggers are qualified with `BUD`; the procedures and `ticket_fieldtype` are created without an explicit schema.

| Order | Script | What it creates |
| --- | --- | --- |
| 1 | [01_ddl.sql](db/01_ddl.sql) | Schema, tables, keys, relationships, and constraints |
| 2 | [07_db_init.sql](db/07_db_init.sql) | Required lookup data, service catalogue, fields, rooms, and help articles |
| 3 | [02_sp.sql](db/02_sp.sql) | Stored procedures and the ticket-fields table type |
| 4 | [03_udf.sql](db/03_udf.sql) | Ticket-count functions |
| 5 | [04_triggers.sql](db/04_triggers.sql) | Ticket reopening and deletion behavior |
| 6 | [05_indexes.sql](db/05_indexes.sql) | Ticket and room indexes |
| 7 | [06_views.sql](db/06_views.sql) | User information and service/category/field views |
| 8 | [08_sample_data.sql](db/08_sample_data.sql) | Demo accounts, 20,000 tickets, and two messages |

The initial data only depends on the tables, so it can be loaded before the procedures. The sample script needs both the procedures and that initial data. It is optional if you intend to create your own users and tickets through SQL; the desktop application has no registration screen.

`01_ddl.sql` drops existing BUD tables and their contents. Treat it as a reset script. The seed scripts are inserts, not migrations: rerunning them can produce duplicate-key errors or additional tickets. The sample script also tries to add its first user to a department already assigned during account creation; `AddUserToDepartment` reports that existing association and returns without adding it again.

Run `05_indexes.sql` once on a fresh schema. Its existing-index checks use table-object lookups, so they do not make reruns safe. Keep `09_test_indexes.sql` out of normal setup; its effects are described below.

### 3. Configure the desktop connection

Open [ui/BUD/BUD.sln](ui/BUD/BUD.sln) in Visual Studio. In [Database.cs](ui/BUD/Entities/Database.cs), set these variables for your database:

| Variable | Value |
| --- | --- |
| `serverAddress` | Your SQL Server host or instance |
| `databaseName` | The database populated above |
| `databaseUsername` | Your SQL Server login |
| `databasePassword` | That login's password |

The active connection string uses SQL Server authentication. A commented alternative in the same file uses Windows integrated authentication; enable the matching declarations if that is how you connect. `App.config` only declares the framework runtime and does not supply these connection settings.

The client both calls stored procedures and executes direct queries, so its database account needs permission for both. Build the solution and start the `BUD` project.

### 4. Sign in with a demo account

These credentials come from `08_sample_data.sql`, independently of the SQL Server login above:

| Email | Password | Seeded role(s) |
| --- | --- | --- |
| `jas@ua.pt` | `jas123` | Student and Teacher |
| `mjs@ua.pt` | `mjs123` | Teacher |
| `jm@ua.pt` | `jm123` | Administrator, stored as `Administator` |
| `cgp@ua.pt` | `cgp123` | Staff |

Use the first account to explore personal tickets and the last to access the shared ticket queue and statistics. Those screens check for the `Staff` role specifically; the Administrator account is not a substitute.

## Data model

The schema contains 18 tables. The original [entity–relationship diagram](DER.png) and [relational schema](ER.png) show their relationships.

![BUD relational schema](ER.png)

| Area | Tables and purpose |
| --- | --- |
| People | `user`, `picture`, `roles`, and `UserRoles`: identity, credentials, separately loaded pictures, and role assignments |
| Organisation | `department`, `userdepartment`, and `room`: department memberships and rooms identified by department, floor, and number |
| Request catalogue | `service`, `category`, `field`, and `category_field`: services, request types, minimum roles, and their input fields |
| Tickets | `ticket`, `ticket_field`, `status`, and `priority`: shared ticket metadata, category-specific answers, and lookup values |
| Conversations | `message` and `attachment`: ticket messages, sender/timestamp information, and file bytes |
| Help | `article`: service-related articles with title, author, content, and publication date |

`UserRoles` and `userdepartment` store start and end dates. Their composite primary keys allow one association per user/role or user/department pair, rather than repeated membership periods for the same pair.

The catalogue links each category to reusable fields. Ticket answers have their own rows in `ticket_field`, so changing a category's field associations does not rewrite existing answers. Deleting a field definition itself is different: its foreign key cascades to stored ticket answers.

The initial dataset contains four roles, three statuses, three priorities, 25 departments, 450 sample rooms, nine services, ten categories, 18 field definitions, and 29 help articles. Ticket categories cover Email, Web Hosting, Audiovisual, E-Learning, and Networks; the other service records do not have seeded ticket categories.

## From a form to a ticket

[NewTicketForm](ui/BUD/Forms/NewTicketForm.cs) reads `BUD.ServiceCategoriesFields`, filtering categories by the user's highest numeric role. It builds service cards, category cards, and input controls from those rows. Department and room fields have dropdown behavior selected by field IDs in [Category.cs](ui/BUD/Entities/Category.cs); other fields use text inputs. Adding ordinary fields is driven by database records, while adding new control types requires C# changes.

Submission builds a `DataTable` of field IDs and values and passes it to `CreateTicket` as `ticket_fieldtype`. The procedure inserts the ticket and its answers in one transaction, using `SCOPE_IDENTITY()` to connect them. New tickets begin as Open.

`SeeUserTickets` accepts optional requester, service, category, status, and priority filters. The desktop uses it for personal tickets and, without a requester filter, the staff queue. The shared queue requests pages of 20 rows through SQL `OFFSET`/`FETCH`.

Staff can change a ticket's priority and status. Saving those changes also sets the responsible user to the signed-in staff member; there is no separate assignee picker. `UpdateTicket` records the closing date when the status becomes Closed. `TicketReopened` clears the closing date and rating when a closed ticket returns to Open or In Progress.

The viewer combines shared metadata with the stored category-specific answers. Those answers remain read-only after submission. Uploading a file creates its own attachment message through `SendAttachmentMessage`, using the fixed text `Attachment: ` and storing the file as `VARBINARY(MAX)` with its filename. Files can be saved back to disk from the conversation. Messages reload when the viewer opens or sends content; there is no live polling.

An unrated, closed ticket in the requester's viewer offers a 0–5 rating. SQL checks the numeric range, while the desktop decides when to show the rating control.

## SQL and desktop reference

| Source | Responsibilities |
| --- | --- |
| [Stored procedures](db/02_sp.sql) | Account creation, authentication, role/department association, ticket creation/query/update/delete/rating, messages, attachments, and profile pictures |
| [Functions](db/03_udf.sql) | Total tickets, optionally by requester, and counts by status, priority, category, or service |
| [Triggers](db/04_triggers.sql) | Reopening cleanup; deleting a ticket's attachments and messages; removing user/department associations during deletion |
| [Views](db/06_views.sql) | `UserInfo` joins profile, role, and department data; `ServiceCategoriesFields` supplies request-form metadata |
| [DashboardForm](ui/BUD/Forms/DashboardForm.cs) | Ticket lists, filters, staff statistics, deletion, and title/content search for articles |
| [ProfileForm](ui/BUD/Forms/ProfileForm.cs) | Read-only identity and membership details, plus profile-picture updates |

The database includes 15 stored procedures, five scalar functions, four triggers, two views, and four explicit secondary indexes.

## Implementation notes

Most write procedures use transactions with `TRY`/`CATCH`. They generally communicate failure through `PRINT` and return codes, so callers need to inspect those results rather than assuming every failure raises an exception. Some client handlers use `ExecuteNonQuery()` without reading the procedure's return value; successful command execution does not necessarily mean the requested write succeeded.

Some boundaries matter when extending the project. Role checks are made in the desktop; the write procedures do not independently enforce ownership or staff authorization. Role selection also does not check assignment end dates. Authentication uses a per-user salt and SHA-256 in SQL, and database connection settings live in source. These describe the coursework implementation, not a complete deployment security design.

The statistics screen reverses the High and Low labels: the seed assigns priority `1` to High and `3` to Low, while `LoadStatistics()` labels their counts Low and High respectively. The Medium count is mapped correctly. The login screen's “Remember me” checkbox and password-recovery link are also placeholders without implemented behavior.

Two query details also deserve care: ticket pagination sorts only by submission date, so rows sharing a date have no deterministic tie-breaker; and the reopening trigger branches on whether any updated row was reopened, then applies its cleanup to every row in that update. The desktop updates one ticket at a time, but bulk-update callers would need different handling.

## Index experiment

[05_indexes.sql](db/05_indexes.sql) indexes ticket requester, priority, and status, plus room department. The optional [09_test_indexes.sql](db/09_test_indexes.sql) compares three ticket queries before and after creating the ticket indexes.

The original [saved results](IndexesTesting.rpt), dated 5 June 2024, record:

| Filter | Rows returned | Without index | With index |
| --- | ---: | ---: | ---: |
| `requester_id = 1` | 6,763 | 110 ms | 34 ms |
| `priority_id = 1` | 6,705 | 47 ms | 13 ms |
| `status_id = 1` | 20,000 | 63 ms | 47 ms |

The sample generator creates four batches of 5,000 tickets, randomising requester and priority while leaving every ticket Open. These figures are one recorded run of `SELECT *` queries timed with `GETDATE()`, without repeated trials or documented cache controls. They are useful project measurements, not a general performance guarantee.

The test drops the three ticket indexes at both the start and the end. After running it, restore them explicitly:

```sql
CREATE INDEX IX_ticket_requester_id ON BUD.ticket(requester_id);
CREATE INDEX IX_ticket_priority_id ON BUD.ticket(priority_id);
CREATE INDEX IX_ticket_status_id ON BUD.ticket(status_id);
```

The room index is unaffected. Do not rerun the entire index script just to restore these three.

## Original project material

The original Portuguese [submission report](apft_submit.md), [PDF report](apft_submit.pdf), [presentation](presentation.pdf), [demonstration video](video_demo.mp4), and [screenshots](screenshots/) remain in the repository. The report documents the requirements, normalization discussion, UI queries, and changes made during the coursework. Some of its SQL links use the old `sql/` path; the working scripts are in `db/` as linked above.

BUD was developed by **Miguel Vila** and **Miguel Reis**, group **P10G7**, for **Bases de Dados, University of Aveiro, 2024**.
