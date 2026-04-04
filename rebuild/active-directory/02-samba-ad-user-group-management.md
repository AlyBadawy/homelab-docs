# Samba AD — User & Group Management Cheatsheet

All commands are run on the DC (`dc.id.in.alybadawy.com`) with `sudo`.

---

## Users

### List all users
```bash
sudo samba-tool user list
```

### Create a user
```bash
sudo samba-tool user create john.doe \
  --given-name="John" \
  --surname="Doe" \
  --mail-address="john@alybadawy.com" \
  --login-shell=/bin/false
```
> You will be prompted for a password. It must meet complexity requirements.

### Delete a user
```bash
sudo samba-tool user delete john.doe
```

### Disable a user (without deleting)
```bash
sudo samba-tool user disable john.doe
```

### Enable a user
```bash
sudo samba-tool user enable john.doe
```

### Reset a user's password
```bash
sudo samba-tool user setpassword john.doe
```

### Show details of a specific user
```bash
sudo samba-tool user show john.doe
```

### List all groups a user belongs to
```bash
sudo samba-tool user getgroups john.doe
```

---

## Groups

### List all groups
```bash
sudo samba-tool group list
```

### Create a group
```bash
sudo samba-tool group add immich-users
```

### Delete a group
```bash
sudo samba-tool group delete immich-users
```

### List all members of a group
```bash
sudo samba-tool group listmembers immich-users
```

### Add a user to a group
```bash
sudo samba-tool group addmembers immich-users john.doe
```

> You can add multiple users at once (comma-separated):
> ```bash
> sudo samba-tool group addmembers immich-users john.doe,jane.doe
> ```

### Remove a user from a group
```bash
sudo samba-tool group removemembers immich-users john.doe
```

---

## Reference — Groups in This Domain

| Group | Purpose |
|---|---|
| `immich-users` | Regular Immich users |
| `immich-admins` | Immich administrators |
| `nextcloud-users` | Regular Nextcloud users |
| `nextcloud-admins` | Nextcloud administrators |

---

## Reference — Key DNs

| Object | DN |
|---|---|
| Base DN | `DC=id,DC=in,DC=alybadawy,DC=com` |
| Users container | `CN=Users,DC=id,DC=in,DC=alybadawy,DC=com` |
| Administrator | `CN=Administrator,CN=Users,DC=id,DC=in,DC=alybadawy,DC=com` |
| A user (example) | `CN=john.doe,CN=Users,DC=id,DC=in,DC=alybadawy,DC=com` |
| A group (example) | `CN=immich-users,CN=Users,DC=id,DC=in,DC=alybadawy,DC=com` |
