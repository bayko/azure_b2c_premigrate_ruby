# azure_b2c_premigrate_ruby

This repository contains a ruby implementation of the `Seamless migration strategy` Powershell scripts provided by Azure for migrating from a legacy identity provider: https://github.com/azure-ad-b2c/user-migration/tree/master/seamless-account-migration 

Rake tasks can be modified and incorporated into a rails application to handle pre-migration of users. Httparty is used to interact with the MS Graph API.

- Register the MS Graph extension `requiresMigration`
- Check the extension status for a particular user
- Premigrate existing users into B2C with dummy password
