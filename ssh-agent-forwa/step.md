This is a Windows permission issue. You are running PowerShell without Administrator privileges, so Windows is preventing you from configuring and starting the SSH Agent service.

Let's fix it.

### Step 1. Open PowerShell as Administrator

1. Press the Windows key.
2. Search for PowerShell.
3. Right-click Windows PowerShell.
4. Select Run as administrator.

### Step 2. Enable SSH Agent

Run:

```
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
```

Verify:

```
Get-Service ssh-agent
```

Expected status:

```
Status   Name       DisplayName
------   ----       -----------
Running  ssh-agent  OpenSSH Authentication Agent
```

### Step 3. Add your PEM key

```
ssh-add C:\Users\ASHIF\Downloads\test2.pem
```

Verify:

```
ssh-add -l
```

### Step 4. Connect to your source EC2

You can now use normal PowerShell (not necessarily Administrator):

```
ssh -A -i C:\Users\ASHIF\Downloads\test2.pem ec2-user@54.167.82.63
```

### Step 5. Connect to the private EC2

From your source EC2:

```
ssh-add -l
ssh ec2-user@10.0.6.60
```

You should now be able to authenticate using the forwarded key, provided the target has its matching public key.

One security note: Agent forwarding is convenient, but only enable it when connecting to a trusted bastion/source instance.