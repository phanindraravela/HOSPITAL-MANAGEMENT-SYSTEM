# Hospital Management System

Java web application built with **HTML / CSS / JavaScript** (front end), **JSP + Servlets** (back end),
**Apache Tomcat 9** (server) and **Oracle Database** (database). Built with **Maven** and ready for **Eclipse**.

## Requirements
- JDK 8 – 17 (the project is compiled for Java 8)
- Apache Tomcat 9.0
- Oracle Database 11g / 12c / 18c / 19c / 21c (Express Edition is fine) and SQL Developer
- Eclipse IDE for Enterprise Java and Web Developers (with Maven / m2e)

## Setup
1. **Database** – open **SQL Developer**, connect as the same user that is in `db.properties` (SYSTEM), open
   `database/hospital_oracle.sql` and press **Run Script (F5)** once. It creates the 10 tables, auto-number
   sequences/triggers and the sample data. Check the *Script Output* panel for errors.
2. **Connection settings** – edit `src/main/resources/db.properties`: `db.user`, `db.password` and **`db.url`**.
   The URL must use the same service name / SID as your SQL Developer connection:
   `jdbc:oracle:thin:@//localhost:1521/XE` (default), `jdbc:oracle:thin:@//localhost:1521/XEPDB1` (XE 18c/21c)
   or `jdbc:oracle:thin:@localhost:1521:XE` (SID style).
3. **Import** – Eclipse: *File → Import → Maven → Existing Maven Projects* → select this folder → Finish.
   Wait for Maven to download dependencies, then right-click the project → *Maven → Update Project* (tick *Force Update*).
4. **Add Tomcat 9** – *Window → Preferences → Server → Runtime Environments → Add → Apache Tomcat v9.0*.
5. **Run** – right-click the project → *Run As → Run on Server* → choose Tomcat 9.
6. Open **http://localhost:8080/HospitalManagementSystem/**

### Run from the command line (no Eclipse)
```
mvn clean package
```
Copy `target/HospitalManagementSystem.war` into Tomcat's `webapps` folder and start Tomcat.

## Logins
| Role | Username | Password |
|------|----------|----------|
| Admin | admin | admin123 |
| Doctor | doctor1 | doctor123 |
| Receptionist | reception1 | recep123 |

(`doctor2 / doctor123` also exists.) Choose the matching role on the login page.
New staff can register on the **Sign Up** page (Doctor or Receptionist only; a doctor sign-up also creates the doctor record).
Admin accounts can only be added by an existing admin under *Staff Users*.

## Features
- Public pages: Home, About, Contact (messages saved in DB), Login, Forgot password
- Dashboards (admin/receptionist and doctor) and Reports (admin)
- Patients, Doctors, Appointments (no double booking for the same doctor/date/time)
- Medical records and prescriptions (several medicines per prescription, printable view)
- Billing (total auto-calculated, *Pay* button for pending bills)
- Pharmacy (low-stock and expired alerts) and Laboratory
- Admin: staff users and contact messages
- Role-based access on the server; doctors only see their own appointments, records, prescriptions and lab tests
- Passwords stored as SHA-256 hashes, prepared statements, CSRF token on every form

## Project structure
```
pom.xml
database/hospital_oracle.sql         optional manual script (the app no longer needs it)
src/main/resources/schema.sql        same schema in JDBC format (used automatically at start-up)
src/main/resources/db.properties.example
src/main/resources/db.properties
src/main/java/com/hms/
    filter/SecurityFilter.java      login, role and CSRF checks
    model/AppUser.java              logged-in user
    model/Entities.java             definition of every module/table and its fields
    servlet/ModuleServlet.java      generic list / add / edit / view / delete
    servlet/Login|Logout|Signup|Forgot|Contact|Public|Dashboard|ReportServlet.java
    util/DBUtil.java (connection, URL fallbacks, lock protection), DbErrors.java (readable ORA messages),
    util/DbInit.java (creates missing tables, fixes sequences, loads sample data when Tomcat starts), Security.java
src/main/webapp/
    index.jsp                       home page
    css/style.css, js/app.js
    WEB-INF/web.xml
    WEB-INF/jsp/layout/header.jsp, footer.jsp
    WEB-INF/jsp/public/about|contact|login|signup|forgot.jsp
    WEB-INF/jsp/module/list|form|view.jsp
    WEB-INF/jsp/dashboard.jsp, reports.jsp, error.jsp
```
To add a field to a module: add the column in the SQL table and one `.add(...)` line in `Entities.java`.

## Troubleshooting a 404
- Use the exact context path shown in Eclipse: *Servers* view → double-click Tomcat → *Modules* tab → *Path*.
  This project's pom sets `finalName` to `HospitalManagementSystem`, so the URL is
  `http://localhost:8080/HospitalManagementSystem/`.
- Right-click project → *Maven → Update Project*, then *Server → Add and Remove* and make sure the project is listed,
  then *Clean…* and restart the server.
- Read the Console view: if Tomcat shows a startup error, every URL returns 404.
- The login / sign-up pages now show the Oracle error code together with what to fix. Details below.
- **ORA-01017 invalid username/password** – the user/password in `db.properties` are not accepted by Oracle.
  Test them first in SQL Developer (same user, same service). If you do not know the SYSTEM password, reset it:
  open a Command Prompt as Administrator, run `sqlplus / as sysdba`, then
  `ALTER USER system IDENTIFIED BY "NewPassw0rd";` (keep the double quotes - required when the password starts with a digit
  or contains symbols). Put the same password in `db.properties`, save, restart Tomcat. On XE 18c/21c the password can differ
  between the root (`/XE`) and the pluggable DB (`/XEPDB1`): set `db.url` to the service your SQL Developer login uses.
  After a wrong password the app pauses 30 s before contacting Oracle again, so it does not lock the account.
- **ORA-28000 account locked** – `ALTER USER system ACCOUNT UNLOCK;` (as SYSDBA), then fix the password.
- **ORA-12514 / 12505 / 12541 / 17002** – wrong service/SID or listener stopped (start `OracleServiceXE` and `OracleXETNSListener`).
  The URLs in `db.url.fallbacks` are tried automatically for these errors.
- **ORA-00942 table does not exist** – restart Tomcat: the tables are now created automatically in the schema the app uses.
- **ORA-04098 trigger is invalid** – restart Tomcat: broken `HMS_TRG_*` triggers are dropped automatically (ids come from sequences now).
- Instead of editing the file you can set `HMS_DB_URL`, `HMS_DB_USER`, `HMS_DB_PASSWORD` (environment) or `-Dhms.db.*` (JVM options).
- Open `http://localhost:8080/HospitalManagementSystem/diag.jsp` (works only on the server itself) to see the driver, the connection,
  the service you are really connected to, and the start-up schema check. Remove `diag.jsp` before real deployment.
- Tomcat must run on JDK 8–17.

## Notes
- *Forgot password* is a demo: it resets the password when username and registered e-mail match. It never works for admin accounts.
- Tables are created in the schema of the user in `db.properties`. Using SYSTEM is fine for a college project;
  for real use create a separate user (`CREATE USER hms IDENTIFIED BY ...; GRANT CONNECT, RESOURCE TO hms;`)
  and put that user in `db.properties` and run the SQL script as that user.
- Usernames are case-insensitive at login. Appointment times are stored as `HH:MM` text.
