I am conducting a data source audit of this Microsoft Sentinel workspace.

I need your help analyzing the SecurityEvent table. Please do the following:

1. Pull the schema for the SecurityEvent table and identify the most important fields for security analysis — specifically fields that identify the source machine, the type of event, the user account involved, and any fields that distinguish different log sources writing to this table.

2. Tell me which field I should use to identify individual machines writing to this table so I can break out one row per log source.

3. Identify which EventIDs are present in this table over the last 30 days and how many records each one has. Group them so I can understand what security coverage this table currently provides.

4. Tell me if there are any unusual or unexpected fields in this schema that might indicate custom DCR transformations or non-standard configurations.

The goal is to understand exactly what is in this table, who is writing to it, and what security value it provides so I can document it in our data source inventory.