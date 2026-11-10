# GROUP-WORK
NAMES                                          



MULISA ISAEL JALIES                            20252SEN228


MUTAMBA ALICE                                    27688

IRUMVA AMANI                                      29765

TUMWINE KENELT                                   20252SEN344

ISHIMWE  DANIEL                                  20252SEN037

KACYEYE  IGIHOZO JANE                         20251SEN124


1. Create the Trigger TRG_RESTRICT_LOGIN_HOURS
You must create it as SYS or user with ADMINISTER DATABASE TRIGGER.sql-- Run as SYSDBA
CREATE OR REPLACE TRIGGER TRG_RESTRICT_LOGIN_HOURS
AFTER LOGON ON DATABASE
DECLARE
  v_user      VARCHAR2(30);
  v_day_num   NUMBER; -- 1=Sunday, 2=Monday ... 7=Saturday in NLS_TERRITORY=AMERICA
  v_day_name  VARCHAR2(10);
  v_time_num  NUMBER; -- HH24MI e.g. 0830 = 830, 1700 = 1700
BEGIN
  -- 2. Identify user
  v_user := SYS_CONTEXT('USERENV','SESSION_USER');

  -- 4. Exceptions: SYS, SYSTEM, DBA_ADMIN can log in any time
  IF UPPER(v_user) IN ('SYS','SYSTEM','DBA_ADMIN') THEN
    RETURN;
  END IF;

  -- Get day and time from DB server
  -- Adjust if your DB server time is not Africa/Kigali
  v_day_num  := TO_NUMBER(TO_CHAR(SYSDATE, 'D', 'NLS_TERRITORY=AMERICA'));
  v_day_name := TO_CHAR(SYSDATE, 'DY', 'NLS_DATE_LANGUAGE=AMERICAN');
  v_time_num := TO_NUMBER(TO_CHAR(SYSDATE, 'HH24MI'));

  -- Convert: 2=Mon, 3=Tue, 4=Wed, 5=Thu, 6=Fri, 7=Sat, 1=Sun
  IF v_day_num BETWEEN 2 AND 5 THEN -- Monday-Thursday
    IF v_time_num < 800 OR v_time_num > 1700 THEN
      RAISE_APPLICATION_ERROR(-20001, 
        'Login denied: Database access allowed Mon-Thu 08:00-17:00 only. User: '||v_user||' Day: '||v_day_name||' Time: '||TO_CHAR(SYSDATE,'HH24:MI'));
    END IF;

  ELSIF v_day_num = 6 THEN -- Friday
    IF v_time_num < 800 OR v_time_num > 1200 THEN
      RAISE_APPLICATION_ERROR(-20002, 
        'Login denied: Friday access allowed 08:00-12:00 only. User: '||v_user);
    END IF;

  ELSIF v_day_num IN (1,7) THEN -- Saturday and Sunday
    RAISE_APPLICATION_ERROR(-20003, 
      'Login denied: No database access on Saturday and Sunday. User: '||v_user);

  END IF;

END;
/





Here is the complete auditing system - tested on Oracle 19c/21c.1. Create Audit TableRun as SYS or DBA_ADMIN:sql-- Create in a central schema e.g. DBA_ADMIN or SYSTEM
-- If you create in SYSTEM, grant privileges to others to insert

CREATE TABLE USER_LOGIN_AUDIT (
    AUDIT_ID      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    USERNAME      VARCHAR2(30)   NOT NULL,
    SESSION_ID    NUMBER,
    LOGIN_TIME    TIMESTAMP DEFAULT SYSTIMESTAMP,
    LOGOUT_TIME   TIMESTAMP,
    ACTION_TYPE   VARCHAR2(10), -- LOGIN / LOGOUT
    IP_ADDRESS    VARCHAR2(45),
    HOSTNAME      VARCHAR2(64),
    OS_USER       VARCHAR2(30),
    PROGRAM       VARCHAR2(100)
);

-- Allow all users to write to it via triggers (triggers run with definer's rights)
GRANT INSERT, UPDATE, SELECT ON USER_LOGIN_AUDIT TO PUBLIC;

-- For performance
CREATE INDEX IDX_USER_LOGIN_AUDIT_SESSION ON USER_LOGIN_AUDIT(SESSION_ID, LOGOUT_TIME);2. Create AFTER LOGON Trigger - TRG_USER_LOGINsqlCREATE OR REPLACE TRIGGER TRG_USER_LOGIN
AFTER LOGON ON DATABASE
DECLARE
  v_sid NUMBER;
BEGIN
  -- Capture info using SYS_CONTEXT
  v_sid := SYS_CONTEXT('USERENV','SESSIONID');

  INSERT INTO DBA_ADMIN.USER_LOGIN_AUDIT (
    USERNAME,
    SESSION_ID,
    LOGIN_TIME,
    ACTION_TYPE,
    IP_ADDRESS,
    HOSTNAME,
    OS_USER,
    PROGRAM
  ) VALUES (
    SYS_CONTEXT('USERENV','SESSION_USER'),      -- DB username
    v_sid,                                      -- Session ID
    SYSTIMESTAMP,                               -- Login time
    'LOGIN',
    SYS_CONTEXT('USERENV','IP_ADDRESS'),         -- Client IP
    SYS_CONTEXT('USERENV','HOST'),              -- Client hostname
    SYS_CONTEXT('USERENV','OS_USER'),           -- OS user
    SYS_CONTEXT('USERENV','MODULE')
  );
EXCEPTION
  WHEN OTHERS THEN
    NULL; -- Never block login if audit fails
END;
/3. Create BEFORE LOGOFF Trigger - TRG_USER_LOGOUTThis updates the last login record with logout time.sqlCREATE OR REPLACE TRIGGER TRG_USER_LOGOUT
BEFORE LOGOFF ON DATABASE
BEGIN
  UPDATE DBA_ADMIN.USER_LOGIN_AUDIT
  SET 
    LOGOUT_TIME = SYSTIMESTAMP,
    ACTION_TYPE = 'LOGOUT'
  WHERE 
    SESSION_ID = SYS_CONTEXT('USERENV','SESSIONID')
    AND USERNAME = SYS_CONTEXT('USERENV','SESSION_USER')
    AND LOGOUT_TIME IS NULL;
    
  -- If no record found (e.g. DB restarted), insert a logout-only record
  IF SQL%ROWCOUNT = 0 THEN
    INSERT INTO DBA_ADMIN.USER_LOGIN_AUDIT (
      USERNAME, SESSION_ID, LOGOUT_TIME, ACTION_TYPE, IP_ADDRESS, HOSTNAME
    ) VALUES (
      SYS_CONTEXT('USERENV','SESSION_USER'),
      SYS_CONTEXT('USERENV','SESSIONID'),
      SYSTIMESTAMP,
      'LOGOUT',
      SYS_CONTEXT('USERENV','IP_ADDRESS'),
      SYS_CONTEXT('USERENV','HOST')
    );
  END IF;
EXCEPTION
  WHEN OTHERS THEN
    NULL; -- Never block logoff
END;
/If you created the table in SYSTEM, replace DBA_ADMIN.USER_LOGIN_AUDIT with SYSTEM.USER_LOGIN_AUDIT.4. Query to Display All Audit Recordssql-- 5. Display all login/logout activities
SELECT 
  AUDIT_ID,
  USERNAME,
  SESSION_ID,
  TO_CHAR(LOGIN_TIME, 'YYYY-MM-DD HH24:MI:SS') AS LOGIN_TIME,
  TO_CHAR(LOGOUT_TIME, 'YYYY-MM-DD HH24:MI:SS') AS LOGOUT_TIME,
  ROUND((LOGOUT_TIME - LOGIN_TIME)*24*60, 2) AS DURATION_MIN,
  ACTION_TYPE,
  IP_ADDRESS,
  HOSTNAME,
  OS_USER
FROM DBA_ADMIN.USER_LOGIN_AUDIT
ORDER BY LOGIN_TIME DESC;

-- Current connected users (not yet logged out)
SELECT USERNAME, SESSION_ID, LOGIN_TIME, IP_ADDRESS, HOSTNAME 
FROM DBA_ADMIN.USER_LOGIN_AUDIT 
WHERE LOGOUT_TIME IS NULL;5. Query to Check Triggers are Created and Enabledsql-- 6. Check both triggers
SELECT 
  TRIGGER_NAME,
  TRIGGER_TYPE,
  TRIGGERING_EVENT,
  STATUS,
  OWNER
FROM DBA_TRIGGERS
WHERE TRIGGER_NAME IN ('TRG_USER_LOGIN','TRG_USER_LOGOUT');

-- Should return 2 rows with STATUS = 'ENABLED'
-- If DISABLED:
-- ALTER TRIGGER TRG_USER_LOGIN ENABLE;
-- ALTER TRIGGER TRG_USER_LOGOUT ENABLE;Testssql-- 1. Create test user
CREATE USER test_user IDENTIFIED BY test123;
GRANT CONNECT, CREATE SESSION TO test_user;
GRANT SELECT ON DBA_ADMIN.USER_LOGIN_AUDIT TO test_user;

-- 2. Open a NEW session:
-- sqlplus test_user/test123
-- Then disconnect: EXIT

-- 3. As DBA, verify:
SELECT * FROM DBA_ADMIN.USER_LOGIN_AUDIT 
WHERE USERNAME = 'TEST_USER' 
ORDER BY AUDIT_ID DESC;

-- Expected: One row with LOGIN_TIME filled, and after EXIT, LOGOUT_TIME filled.Note: BEFORE LOGOFF will NOT fire if the session crashes, SHUTDOWN ABORT, or network disconnect. For full auditing, also enable Oracle unified auditing: AUDIT SESSION;
