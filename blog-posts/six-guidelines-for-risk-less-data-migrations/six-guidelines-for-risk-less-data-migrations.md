---
published: false
title: '6 guidelines for risk-less data migrations'
cover_image: ''
description: '6 main points to watch out when migrating data in databases.'
tags: database, dynamodb
series:
canonical_url:
---

> *“The majority of project issues I have seen come from databases, whatever is the technology”* said one day one my experienced engineering manager...

This is why I have built 6 guidelines I try to follow when modifying data on tech projects. They are especially helpful when working with databases like DynamoDB with few tooling (no ORM, etc.).

> **Data is key in business… and it is where it often fails.**

Almost every app has a **database** with more or less structured data. These data items may evolve during app development, with the addition of new fields, changes to field structure, data updates, or removal of fields. These evolutions, often referred to as **migrations**, are critical as they enable the app to evolve but can also introduce bugs if not executed properly.
A data migration **plan** is a perfect tool to mitigate the risks of these operations, as it avoids common pitfalls and ensures reliability.  

## 📋️ First thing: have a migration protocol defined

### Define a protocol to follow in advance

- Define **who** is responsible of what. You can use a responsibility matrix ([**RACI**](https://en.wikipedia.org/wiki/Responsibility_assignment_matrix))
- Define **when** the migration should be planned
- Define **what** should be done in precise steps (manually deploy a function, trigger a Continuous Deployment workflow, etc.) ?

### Follow the protocol

A simple advice... to avoid acting in panic if something goes wrong…

### Upgrade the protocol with your learnings

There are often recurring patterns in migrations: making a field non nullable, updating a data type, etc. To capitalize on them, build a list of common migrations with explanations of the best implementation strategy based on past experiences.

## ↔️ Have a strategy to handle the transition state

### **Estimate the duration and load** (scalability) of the migration process, in all environments

- What specific cases should be anticipated in production?
  - What differences should be taken into account when estimating the migration sizing (database size, environment configuration)?
  - Are there API/database quotas that you should plan for?
- Apply a margin for the manual parts (launching the migration, etc.)

### Migration strategies

Given that estimation and business concerns, choose between these **2 main strategies**:

#### 💥 **“Big Bang” Migration**

Only one version of a migration set (pieces of data to modify) can exist at application uptime.

- Plan a **service interruption** when data will be unavailable.

  ✍️ Example of a user-friendly service interruption: during the migration process, the frontend application displays a "maintenance banner", and the backend is programmatically locked, to ensure no side-effect can corrupt data.
- ✳️ Pros: quicker method, advised when building a project with low risks if some data is lost/corrupted
- ⚠️ Risks:
  - possibly long downtime,
  - risk of database throttling (in case of naive parallel read/write implementation for instance)
  - more difficult to write migrations with a reliable rollback

#### 🌊 **Migration in two (or more) steps**

Handle old and new versions inside the migration set.

- Data migration and new code deployment must be uncoupled and done in the proper order.
- Examples :
  - if you **remove** a column (or a table, etc.), remove the code using this column and then migrate the data
  - if you **add** a column, migrate the data first and then deploy the code using it
  - if you **update** a column, add a new column and, after it has been fulfilled progressively, remove the old column
- 💡 You can run integration tests with both versions to ensure the application works in both cases
- ✳️ Pros: reliable method, that is error proof (if the migration fails, the system can continue running without interruption)
- ⚠️ Risks:
  - more complex to develop
  - longer to fulfill completely
  - risk of maintaining multiple versions over a long period of time.

## 🔂 Make a plan reproducible in multiple environments

### Write idempotent migrations (i.e. a migration can be applied multiple times with the same result)

- There are many reasons for needing to re-apply a migration: migration partially completed, stopped or failed, etc.
- The migration script should work without any state-full input

✍️ Simple example of **idempotent migrations**

- ❌ **Bad**:   if you run the migration twice, the data will be corrupted.

  ```python
  def perform_migration(item):
      item.count += 1

  def not_idem_potent_migration(new_version):
          perform_migration(item)
          item.version = new_version
  ```

- ✅  **Good**: even if the migration itself (incrementation) is not idem-potent, it is not possible to run it twice with this migration.

  ```python
  def idem_potent_migration(new_version: int):
      if (item.version >= new_version):
          return
      else:
          perform_migration(item)
          item.version = new_version
  ```

### Save the migration scripts

- Committing migration scripts ensures the same process can be applied to another environment easily and with known and reproducible steps.
- In case of error, it will help understanding what happened.

### Test your scenario

- Test the migration script (with unit tests) to verify its functionality.
  - Check that data items of the migration set are correctly selected
  - Check all edge cases
- Use dedicated environments to test the migration plan
  - Non-production environments can help to find bugs, all the more if data is ISO-prod.
  - If it is not possible to test a scenario with production data, you can at least run scripts to check for errors, or run migrations in dry-run

## 📆 The plan includes communication with stakeholders

Running a migration can have impacts on the underlying business, so it should not remain in the technical sphere, but be shared to stakeholders.

### Communicate to business owners and stakeholders

Explicit the risks of the migration and consider their concerns

- Stakeholders should be involved in the decision to migrate data, as they know the system and are responsible of it.
- They may help find edge cases in the migration, like **dependencies to other teams**, a **specific business case** not handled, etc.
- ✍️❌ **Common pitfall**: Delete a deprecated column that is still used by another team
- ✍️✅ **Good practice**: Continue supporting legacy versions until they are no longer in use, and establish a deadline to cease support.

### Choose the right moment to run the migration

- ✍️❌ **Common pitfall**: Run a resource-consuming migration during peak hours (example: adding a non-nullable column of a table in PostgreSQL that will lock it).
- ✍️✅ **Good practice**: Run a “big bang” migration just after the deployment to reduce the service interruption lead time. Run it during off-peak hours and when a development team is available in case of error.

## ↩️ A rollback strategy is defined

### Backup your data and practice restoring it

If anything goes wrong, you should be *able to restore* your database to a correct state.

- Cloud providers have services dedicated to backups (e.g. [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html) if you are using AWS, [Actifio](https://www.actifio.com/) on GCP, etc.) and database systems often come with their own backup solution.
- Practice restoring your data
- Be pragmatic. Can you afford to restore your data?
  - Some data items might have been updated during the process
  - Is it worth to rollback, if only a few items have an issue?
  - What is the risk of a data loss?

### Write a “down” migration, as well as the “up” migration… or at least a Plan B

How many times I've heard (or said) "It won't fail, no need of plan B"... and it finally failed ?
I won't go through implementation details here (and a lot of ORMs have dedicated tools for this), the main point is "Don't be overconfident"!

## 🧱 The plan includes a check that everything is ok in the end

### Keep track of migration states with versioning

- If possible, save the changes and the version of items during the migration process.
- Keep track of migration applications in each environment.
- The versions must be strictly increasing (in a specific order relationship) and deterministic to be able to compare versions. For instance, with an incremented integer or a timestamp and a reference to the previous version.

### Remove data after complete validation only

Although, this does not guarantee that you will end up with a successful migration after the removal operation, this allows you to detect potential issues you have not forecast and facilitate reverting the changes.

✍️ E.g. only remove ‘address’ after you have correctly added ‘street_name’, ‘city’, ‘postcode’.

### Check that everything is ok **after** the migration

- **Audit the database after migration**
  - Use a monitoring system to keep track of new errors.
- Not seeing code or monitoring issue do not mean there is no issue ! Check it with **business owners**.
- You can set-up end-to-end tests to avoid regressions.

## 🌟 What to remember?

These 6 guidelines are just an attempt to sum-up what to care about when applying data migrations. They can also apply to application deployments or whatever operation that introduces a breaking change. But the main learning could also be **capitalize on knowledge** to avoid reproducing the same mistakes !

Want to share your own tips or tech convictions? Don't hesitate to comment on this post 😉
