---
title: "Using DNF5 API after running the transaction"
date: "2023-04-17"
summary: "What happens to the existing sack and how to deal with that"
tags: ["dnf"]
ShowCodeCopyButtons: true
ShowToc: true
ShowReadingTime: true
ShowPostNavLinks: true
---

## Intro

A common use case for DNF5 API is installing packages, which is quite 
straightfoward.

First, we need to create a `Base`, the core object that holds a runtime 
environment. We load its configuration from the system and run the `setup` 
method to prepare the environment.

Next, we can prepare a repository sack, which holds information about 
configured repositories and the state of local and remote packages, while 
potentially refreshing metadata from remote servers if needed.

With the setup complete, we can tell DNF5 what we want to do, in this case,
install our package. This is done by configuring the `Goal` object.

After defining our intention, we can proceed to calculate the transaction,
determining the necessary actions to achieve our goal. Finally, we can perform
the resulting action, which is downloading and installing the packages.

An example Python script that accomplishes this might look like this:

```python
import libdnf5

base = libdnf5.base.Base()
base.load_config_from_file()
base.setup()

sack = base.get_repo_sack()
sack.create_repos_from_system_configuration()
sack.update_and_load_enabled_repos(True)

goal = libdnf5.base.Goal(base)
goal.add_install('my-awesome-package')

transaction = goal.resolve()
transaction.download()
transaction.run()
```

After this point, if everything went well, the package should be successfully installed
on the system.

What could be tricky is when trying to use the API after the transaction. The problem is that
the existing repository sack does not reflect the updated state after the transaction was
executed. This is because managing that state with connected third-party libraries would be
very difficult.

---

## Use information from the Transaction object

If you only need to query information about the post-transaction state, you can use the data 
provided by the `Transaction` object.

The `get_transaction_packages()` method can be particularly useful for this purpose. It allows 
us to query which packages were involved in the transaction, the specific actions taken with 
these packages, and any packages they may have been replaced with.

For example, let's say we want to retrieve a list of new files that were installed during the 
transaction. In this case, we'll need to pre-load the filelists metadata in DNF5 before loading 
the repository sack. We can do this by adding the following code:

```python
base.get_config().get_optional_metadata_types_option().add_item('filelists')
```

Then, you can use the helper function `transaction_item_action_is_inbound` to filter only 
inbound packages from the transaction. Finally, you can query the package files contained
in the transaction:

```python
newly_installed_files = set()
for transaction_package in transaction.get_transaction_packages():
    action = transaction_package.get_action()
    if libdnf5.base.transaction.transaction_item_action_is_inbound(action):
        package_files = transaction_package.get_package().get_files()
        newly_installed_files |= set(package_files)
```

---

## Use the RPM API

Another alternative is to use the underlying RPM API. This should be the most
effective way for querying information about installed packages, as we are directly
reading the data from the SQLite database: 

```python
import rpm

# Prepare the transaction set while ignoring package signatures verification
transaction_set = rpm.TransactionSet()
transaction_set.setVSFlags(rpm._RPMVSF_NOSIGNATURES)

# Find the newest package in the database
last_package = max(transaction_set.dbMatch(), 
                   key=lambda package: package[rpm.RPMTAG_INSTALLTIME])

# Get packages only related to the latest transaction
last_packages = transaction_set.dbMatch(rpm.RPMTAG_INSTALLTID, 
                                        last_package[rpm.RPMTAG_INSTALLTID])

# Aggregate all related files
files = set()
for package in last_packages:
    files |= set(package[rpm.RPMTAG_FILENAMES])

```

---

## Start with the new Base

When we want to perform multiple transactions in a row, the easiest way is probably to create
a fresh new `Base` each time. If we would continue with the use case of installing packages, 
we can prepare a helper method in our script for this purpose:

```python
import libdnf5

def install_package(spec):
    base = libdnf5.base.Base()
    base.load_config_from_file()
    base.setup()

    sack = base.get_repo_sack()
    sack.create_repos_from_system_configuration()
    sack.update_and_load_enabled_repos(True)

    goal = libdnf5.base.Goal(base)
    goal.add_install(spec)

    transaction = goal.resolve()
    transaction.download()
    transaction.run()

for package in ['my-awesome-package', 'another-great-package']:
    install_package(package)
```

---

## References

- [DNF5 upstream](https://github.com/rpm-software-management/dnf5)
- [RPM upstream](https://github.com/rpm-software-management/rpm)
