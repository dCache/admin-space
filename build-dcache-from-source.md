# Building and updating dCache from Source

This guide describes how to set up your build environment, compile dCache, and install or update it on your system. This description has been tested on Alma 9.


## 1. Install the required packages

First, make sure you have the necessary development tools and dependencies installed:

```bash
dnf group install -y "Development Tools"
dnf install -y maven oidc-agent-cli java-21-openjdk-devel
```

We assume you already have a functional dCache configuration and database. If you haven't, please follow the steps in https://www.dcache.org/manuals/Book-10.2/install.shtml, or use the [dCache-aio](https://github.com/sara-nl/dcache_aio) script to quickly set up a simple dCache instance. 

## 2. First time

If you haven't yet checked out the source with git, please do this:

```bash
cd
git clone https://github.com/dCache/dcache.git
```

## 3. Prepare to build

Make sure you have the latest greatest master snapshot of the dCache source:

```bash
cd ~/dcache
git pull
```

If there is a previously built RPM package, copy it to a safe place (in case you want to do a roll back later)

```bash
mkdir -p ~/dcache-backups
cp --verbose packages/fhs/target/rpmbuild/RPMS/noarch/dcache*.rpm ~/dcache-backups/
```

Select OpenJDK 21 for the build process (required with the current master, at ~11.2)

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
export PATH="$JAVA_HOME/bin:$PATH"

# Verify you have the correct Java
java --version
```

## 4. Build

The build process can take ~10 minutes. You may want to grab a cup of tea.

```bash
mvn clean package -am -pl packages/fhs -P rpm

# Check the resulting package
ls -l packages/fhs/target/rpmbuild/RPMS/noarch/dcache*.rpm
```

If something went wrong, please reach out on the dCache User Forum mailing list. There will probably be someone who can assist.

If you're certain there is an error in the source code and not in this manual, you could also submit an issue in the dcache/dcache github repo.

## 5. Upgrade to the newly built version

Now that we have a fresh RPM, we can carefully follow these steps to install it.

```bash
# Check if the config files contain any issues, as a baseline. The output should be empty.
dcache check-config

# Stop dCache
systemctl stop dcache.target

# If this dCache contains data you care about, you may want to back up your Postgres database, with a command that looks like this:
/usr/pgsql-17/bin/pg_dumpall -U postgres | gzip > postgres-backup.sql.gz

# Upgrade to the new version
yum localinstall packages/fhs/target/rpmbuild/RPMS/noarch/dcache*.rpm

# Check again if the config files contain issues; there might be deprecated settings.
dcache check-config

# Let dCache apply database schema updates, if there are any. 
dcache database update

# Refresh the systemd information (it may have changed because of the update)
systemctl daemon-reload

# Start dCache
systemctl start dcache.target

# Verify that dCache is running
dcache_status
```

## 6. Troubleshooting

If one or more dCache services are stuck in an "activating" restart loop, check the Java version. The Java version that dCache needs to run, might be a different from the version needed to build!

```bash
java -version
# If it's not the version that dCache needs, run this:
update-alternatives --config java
```

If the Java version was not the problem, please look in the logs (probably /var/log or /var/log/dcache, or view with `journalctl -u name-of-dcache-domain`) for errors.

## 7. Roll back

If for some reason you want to roll back to the previous build, here's a high level recipe:

1. Stop dCache
1. With the NEW RPM still installed, do a dcache database rollback. This will undo schema changes. This command will probably take you back to where you want: `dcache database rollbackToDate $(date --date yesterday +%FT%T)`.
1. Downgrade the RPM to the previous, that you backed up: `dnf downgrade ~/dcache-backups/dcache-<version>.rpm`
1. `dcache version`
1. `dcache check-config`
1. Start dCache
