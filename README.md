# Brute
Brute is a project for monitoring authentication attempts on servers using OpenSSH. It tracks and records each attempt and provides detailed information about the person who made the attempt.
<br><br>
Currently, this project must use a specific version of OpenSSH. Unfortunately, the changes made to this may compromise the security of your server, so use with <b>caution</b>.

* <b>Straightforward</b> — Simply call the endpoint ```/brute/attack/add```, and Brute will log, analyze, and store the credentials for you.
* <b>Extendable Metrics</b> — Brute allows developers to easily add or remove metrics as needed. You can easily integrate additional metrics or connect an API with minimal effort to Brute.
* <b>Location Information</b> — Information can be easily accessed through the [Ipinfo](https://ipinfo.io//) API, which is integrated into Brute. This integration allows for retrieval of detailed data from the individual's IP address.
<br><br>
## Installation
*Ubuntu 22.04 was used*

Before we begin setting up and installing Brute & OpenSSH, ensure you have downloaded the following libraries and tools.
```bash
sudo apt install build-essential zlib1g-dev libssl-dev libpq-dev 
sudo apt install libcurl4-openssl-dev libpam0g-dev
sudo apt install autoconf checkinstall
```
### OpenSSH
```bash
git clone https://github.com/notpointless/openssh-9.8-patched
cd openssh-portable
autoreconf
./configure --with-pam --with-privsep-path=/var/lib/sshd/ --sysconfdir=/etc/ssh
```
```
sudo make
sudo checkinstall
```
After you run the commands above, a UI will show up. Make sure that your settings look like this:
```
0 -  Maintainer: [Your Name <you@example.com>]
1 -  Summary: [OpenSSH Portable]
2 -  Name:    [openssh-portable]
3 -  Version: [9.8p1]   <-- Enter your chosen version here
4 -  Release: [1]
5 -  License: [GPL]
6 -  Group:   [checkinstall]
7 -  Architecture: [amd64]
8 -  Source location: [openssh-8.9p1]
9 -  Alternate source location: [ ]
10 - Requires: [ ]
11 - Provides: [openssh-portable]
12 - Conflicts: [ ]
13 - Replaces: [ ]
```
After building and installing, now you need to replace the old SSH with the new one. You can do that with these commands.
```
sudo systemctl stop ssh

# before we run these commands we should make a backup first.
sudo cp -R /etc/ssh /etc/ssh_backup

sudo mv /usr/bin/ssh /usr/bin/ssh_old
sudo ln -s /usr/local/bin/ssh /usr/bin/ssh

sudo mv /usr/sbin/sshd /usr/sbin/sshd_old
sudo ln -s /usr/local/sbin/sshd /usr/sbin/sshd
```
Now run ```sudo systemctl start ssh``` and run ```ssh -V```. If the following message pops up that means you successfully setup OpenSSH.
```
(Brute) OpenSSH_9.8...
```
Before we proceed to the next section, we need to disable the penalty system that was recently introduced in OpenSSH.
```
sudo nano /etc/ssh/sshd_config

# Add these fields to sshd_config (first one is all you need.)
PerSourcePenalties no
PerSourcePenaltyExemptList 0.0.0.0/0
PerSourcePenaltyExemptList 0:0:0:0:0:0:0:0/0

sudo sshd -t
sudo systemctl restart sshd
```

### Pam
Before proceeding with the next steps, we need to compile the PAM module. To do that, follow these instructions:
```
sudo git clone https://github.com/notpointless/brute_pam
cmake .
make
# the file should now be in /lib/pam_module.so ... you can rename it to pam_brute.so if you wish.
```
*note: before compiling ensure you set BRUTE_BEARER_TOKEN and BRUTE_POST_URL accordingly inside the library.c file.*

All PAM modules should be located in ```/lib/x86_64-linux-gnu/security/```. Simply place the brute PAM module (pam_brute.so) in this directory.
You can use ```scp pam_brute.so {username}@{address}:/lib/x86_64-linux-gnu/security/``` from your local computer to copy the file into there. Before running SCP command ensure you have write access to the directory.
```bash
sudo nano /etc/pam.d/common-auth

original /etc/pam.d/common-auth
# here are the per-package modules (the "Primary" block)
auth    [success=1 default=ignore]      pam_unix.so nullok
# here's the fallback if no module succeeds
auth    requisite                       pam_deny.so

to

# here are the per-package modules (the "Primary" block)
auth    [success=2 default=ignore]      pam_unix.so nullok
# enable Brute.
auth    optional                        pam_brute.so
# here's the fallback if no module succeeds
auth    requisite                       pam_deny.so
```
### Brute
Now we can compile the actual Brute application. Ensure you have rustup & cargo installed.
```
# we need this for cargo.
sudo apt install pkg-config

git clone https://github.com/bruteexposed/brute.git
cd brute
sudo cargo build
./brute
```

## Integrating your own metrics
You can quickly add your metrics by following these steps: create a table using Diesel CLI, add the metric model to ```model.rs```, implement the required functionality, and then integrate it into the ```start_report()``` function located in ```system::reporter```.

### Generating a migration for a table
```up.sql``` and ```down.sql``` were generated by running the following command: ```sqlx migration add -r top_usr_pass_combo```
```sql
-- Brute should automatically run the migraitons
```
```sql
-- up.sql
CREATE TABLE top_usr_pass_combo (
    id VARCHAR(255) PRIMARY KEY,
    username VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    amount INTEGER NOT NULL DEFAULT 0,
    CONSTRAINT unique_username_password UNIQUE (username, password)
);
```
```sql
-- down.sql
DROP TABLE top_usr_pass_combo
```
### Adding the table to models.rs
```rust
#[derive(Debug, sqlx::FromRow, Getters)]
pub struct TopUsrPassCombo {
    id: String,
    username: String,
    password: String,
    amount: i32,
}
```
### Implementing functionality
```rust
// brute.rs
// the function should not be pub and should be async.
impl Reportable<BruteReporter<BruteSystem>, ProcessedIndividual> for TopUsrPassCombo {
    async fn report(
        reporter: BruteReporter<BruteSystem>,
        mut model: ProcessedIndividual,
    ) -> anyhow::Result<Self> {
        todo!()
    }
}
```
### Integrating the metric to .start_report()
```rust
// brute.rs
// locate add(..) and add the metric.
pub async fn start_report(&self, payload: Individual) {
    let individual = Individual::report(self.clone(), payload).await.unwrap();
    let processed_individual = ProcessedIndividual::report(self.clone(), individual).await.unwrap();
    // new one is her...
    TopUsrPassCombo::report(self.clone(), processed_individual).await.unwrap();
}
```
