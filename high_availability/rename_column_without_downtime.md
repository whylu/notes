# rename column without downtime

## database supports trigger
1. add new new-column
2. copy old-column data to new-column
3. add trigger for sync this two column's value
4. rolling update application
    - 4.1 old app using old-column
    - 4.2 new app using new-column
5. remove old-column


## database unsupports trigger
1. add new new-column, old-column-update-version, new-column-update-version
2. copy old-column data to new-column
3. rolling update application
    - 3.1 old app: 
        - read old-column
        - write old-column, update old-column-update-version

    - 3.2 new appV1:
        - read old-column, new-column, chose one by greater(old-column-update-version, new-column-update-version)
        - write old-column and new-column, update old-column-update-version, new-column-update-version

4. copy old-column to new-column if old-column-update-version > new-column-update-version  
    after this step, old-column == new-columnm, old-column-update-version == new-column-update-version,

5. rolling update application
    - 5.1 new appV1: the same as step-3.2 appV1
    - 5.2 new appV2: 
         - read new-column
         - write new-column 

6. remove old-column, old-column-update-version, new-column-update-version
