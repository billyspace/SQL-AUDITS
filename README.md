# SQL-AUDITS
Audit Server and Database level
This will cover:

Server-level auditing (security/logins/roles).

Database-level auditing (HR, Finance, PII tables).

Both GUI and T-SQL steps (side-by-side).

Testing the audits (to prove it works).

Archiving with AzCopy (SQL Agent job).

Best practices (capacity, rollover, compliance).

SQL Server Audit Runbook (Production-Grade)


1. Server Audit (Base Target)
GUI Steps

In SSMS, go to Security → Audits → New Audit.

Name: Audit_Prod.

Destination: File → C:\SQLAudit.

Set Maximum file size = 200 MB.

Set Maximum rollover files = 5.

Queue delay: 5000 ms.

On audit log failure: Fail operation (SOX/HIPAA standard).

Click OK, then Enable Audit.

T-SQL
CREATE SERVER AUDIT [Audit_Prod]
TO FILE (FILEPATH = 'C:\SQLAudit\', MAXSIZE = 200 MB, MAX_ROLLOVER_FILES = 5)
WITH (QUEUE_DELAY = 5000, ON_FAILURE = FAIL_OPERATION);
ALTER SERVER AUDIT [Audit_Prod] WITH (STATE = ON);

2. Server Audit Specification (Critical Actions)
GUI Steps

Under Security → Server Audit Specifications → New Server Audit Specification.

Name: SERVER_AUDIT.

Bind to audit: Audit_Prod.

Add these Audit Action Types:

SERVER_PRINCIPAL_CHANGE_GROUP (new logins, dropped logins).

SERVER_ROLE_MEMBER_CHANGE_GROUP (role membership changes).

FAILED_LOGIN_GROUP (failed logins).

DATABASE_CHANGE_GROUP (new DBs, drops, alters).

AUDIT_CHANGE_GROUP (audit tampering).

SERVER_OPERATION_GROUP (instance-wide ops).

SERVER_OBJECT_CHANGE_GROUP (object creation/deletion).

Click OK, then Enable.

T-SQL
CREATE SERVER AUDIT SPECIFICATION [SERVER_AUDIT]
FOR SERVER AUDIT [Audit_Prod]
ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (FAILED_LOGIN_GROUP),
ADD (DATABASE_CHANGE_GROUP),
ADD (AUDIT_CHANGE_GROUP),
ADD (SERVER_OPERATION_GROUP),
ADD (SERVER_OBJECT_CHANGE_GROUP);
ALTER SERVER AUDIT SPECIFICATION [SERVER_AUDIT] WITH (STATE = ON);

3. Database Audit Specification (Sensitive DBs)
GUI Steps

Expand DB → Security → Database Audit Specifications → New.

Name: Finance_Audit.

Bind to audit: Audit_Prod.

Add critical actions:

SCHEMA_OBJECT_CHANGE_GROUP.

DATABASE_PRINCIPAL_CHANGE_GROUP.

DATABASE_OBJECT_PERMISSION_CHANGE_GROUP.

SELECT / UPDATE / DELETE / INSERT → on HumanResources.Employee, Sales.CreditCard, Sales.PersonCreditCard.

Click OK, then Enable.

T-SQL
CREATE DATABASE AUDIT SPECIFICATION [Finance_Audit]
FOR SERVER AUDIT [Audit_Prod]
ADD (SCHEMA_OBJECT_CHANGE_GROUP),
ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
ADD (DATABASE_OBJECT_PERMISSION_CHANGE_GROUP),
ADD (SELECT ON OBJECT::HumanResources.Employee BY PUBLIC),
ADD (UPDATE ON OBJECT::HumanResources.Employee BY PUBLIC),
ADD (DELETE ON OBJECT::HumanResources.Employee BY PUBLIC),
ADD (INSERT ON OBJECT::HumanResources.Employee BY PUBLIC),
ADD (SELECT ON OBJECT::Sales.CreditCard BY PUBLIC),
ADD (UPDATE ON OBJECT::Sales.CreditCard BY PUBLIC),
ADD (DELETE ON OBJECT::Sales.CreditCard BY PUBLIC),
ADD (INSERT ON OBJECT::Sales.CreditCard BY PUBLIC),
ADD (SELECT ON OBJECT::Sales.PersonCreditCard BY PUBLIC),
ADD (UPDATE ON OBJECT::Sales.PersonCreditCard BY PUBLIC),
ADD (DELETE ON OBJECT::Sales.PersonCreditCard BY PUBLIC),
ADD (INSERT ON OBJECT::Sales.PersonCreditCard BY PUBLIC);
ALTER DATABASE AUDIT SPECIFICATION [Finance_Audit] WITH (STATE = ON);

4. Testing Audit
Generate activity:
-- Trigger SELECT
SELECT TOP 1 * FROM HumanResources.Employee;

-- Trigger unauthorized login
EXEC xp_logininfo 'FakeUser'; -- expect failure

Read audit log:
SELECT event_time, server_principal_name, action_id, statement
FROM sys.fn_get_audit_file('C:\SQLAudit\*.sqlaudit', DEFAULT, DEFAULT)
ORDER BY event_time DESC;

5. Archiving Audit Files to Azure Blob (AzCopy)
Install AzCopy

Download: Microsoft AzCopy

Place in C:\Program Files (x86)\AzCopy\azcopy.exe.

Create Azure Blob container
az storage container create `
  --name sqlauditarchive `
  --account-name <storageaccount> `
  --auth-mode login

SQL Agent Job (Step)
"C:\Program Files (x86)\AzCopy\azcopy.exe" copy "C:\SQLAudit\*.sqlaudit" "https://<storageaccount>.blob.core.windows.net/sqlauditarchive?<SAS_TOKEN>" --move


--move ensures files are deleted locally after upload.

Schedule: Daily/Hourly depending on log volume.

Retention: Blob lifecycle policy can auto-delete after N days (GDPR/SOX policy).

6. Best Practices

Always use Fail Operation for audit failure in compliance environments.

Restrict file system ACLs on C:\SQLAudit to DBAs only.

Rotate to Azure Blob or SIEM daily — don’t rely on local 5-file rollover for long-term retention.

Regularly test audit by simulating unauthorized actions.

Document your audit matrix (who, what, why audited) signed off by Compliance team.

With this runbook, you now have:

Full server-level + database-level auditing.

T-SQL + GUI procedures.

Testing scripts.

Archiving/automation hardened with AzCopy.
