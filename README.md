<h1>Home Lab Running Active Directory</h1>

<h2>Description</h2>
Built a virtualized enterprise network environment to simulate real-world IT administration tasks. Using virtualization software, a Windows Server domain controller was deployed and configured with Active Directory to manage users, groups, and organizational units. A client machine was connected to the domain to replicate a typical corporate workstation environment. The lab demonstrates core system administration skills including user provisioning, domain management, and basic automation using PowerShell.
<br />


<h2>Languages and Utilities Used</h2>

- <b>PowerShell</b> 
- <b>Remote Desktop Protocol</b>
- <b>Active Directory Domain Services</b>

<h2>Environments Used </h2>

- <b>Windows 10</b>
- <b>Windows Server 2019</b>
- <b>Oracle VirtualBox</b>

<h2>Project Walk-Through:</h2>

<p align="center">
Created the virtual lab environment<br/>
<img src="https://i.imgur.com/oiNwqSx.png" height="80%" width="80%" alt="virtual lab environment"/>
<br />
<br />
Installed Windows Server and configured the domain controller<br/>
<img src="https://i.imgur.com/SNB2VUs.png" height="80%" width="80%" alt="domain controller"/>
<br />
<br />
Configured network settings for the server<br/>
<img src="https://i.imgur.com/2W3OwB7.png" height="80%" width="80%" alt="network settings"/>
<br />
<br />
Installed AD DS and Promoted the server to a domain controller<br/>
<img src="https://i.imgur.com/Ajjho3C.png" height="80%" width="80%" alt="Active Directory Domain Services"/>
<br />
<br />
Created organizational units and administrative structure<br/>
<img src="https://i.imgur.com/PFA0dnr.png" height="80%" width="80%" alt="organizational units and administrative structure"/>
<br />
<br />
Joined the client machine to the domain<br/>
<img src="https://i.imgur.com/KQiQJsx.png" height="80%" width="80%" alt="Joined the client machine to the domain"/>
<br />
<br />
Verified domain user login from the client machine<br/>
<img src="https://i.imgur.com/pUyJV6e.gif" height="80%" width="80%" alt="domain user login from the client machine"/>
</p>

<h2>Results</h2>
By following these steps, I successfully set up a home lab environment running Active Directory on a Windows Server VM in Oracle VirtualBox. I was able to install and configure AD DS, promote the server to a domain controller, and use PowerShell scripts to populate user accounts.
