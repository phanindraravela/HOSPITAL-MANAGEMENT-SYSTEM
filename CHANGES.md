# Changes in this version

## Why login/sign-up showed a database error
`diag.jsp` reported `ORA-01017: invalid username/password; logon denied`: Tomcat, JSTL and the Oracle driver were fine, but
Oracle rejected the user/password in `db.properties`. That value must be corrected on your side (see README, "ORA-01017").
Everything else below makes the next problem obvious or self-healing instead of repeating the same vague error.

## Fixes
- `DBUtil`: after ORA-01017/28000/28001 it pauses 30 s instead of retrying on every click (repeated failures lock SYSTEM);
  tries `db.url.fallbacks` (XEPDB1 / SID style) when the listener or service name is wrong; settings can come from
  environment variables or `-Dhms.db.*`; missing driver gives a clear instruction.
- `DbErrors` (new): login, sign-up and forgot-password show the ORA code plus what to change.
- `DbInit` + `schema.sql` (new): at start-up creates any missing tables in the schema the app really uses and loads the
  sample data whenever users/doctors/patients are all empty. It guarantees an `admin / admin123` account if the users
  table is empty. It never drops tables or data.
- **No database triggers any more (v2.1).** The first v2 run showed `ORA-04098: trigger 'SYSTEM.HMS_TRG_DOCTORS' is invalid`
  on startup: the auto-number triggers came out invalid on that Oracle, so every INSERT failed. Primary keys now come from
  the `hms_seq_<TABLE>` sequences, read by the Java code (`DBUtil.nextId`) in sign-up, contact form, all module forms and
  the sample data. Missing sequences are created automatically; broken `HMS_TRG_*` triggers are dropped at start-up and
  Oracle's reason is written to the Tomcat log.
- `LoginServlet`: `changeSessionId()` no longer fails if no session exists yet.
- `SignupServlet`: duplicate username detected by ORA-00001 as well as SQLState 23xxx.
- `ForgotServlet`: database problems show a message instead of a stack trace; admin accounts can no longer be reset this way
  (the admin e-mail is public in the sample data, so anyone could take over the admin account).
- `diag.jsp` / `error.jsp`: only usable from the server machine itself (no public stack traces or connection details).
- `.gitignore` ignores `db.properties` (keep secrets out of Git); `db.properties.example` added. README corrected.

## Not changed (suggested later)
- Passwords are unsalted SHA-256 (works, but use PBKDF2/bcrypt for real use).
- Using SYSTEM is fine for a college project; a dedicated `hms` schema user is better.
- No connection pool (a new connection per query).
