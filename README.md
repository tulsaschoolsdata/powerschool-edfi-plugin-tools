# PowerSchool Ed-Fi Plugin


## Installation

#### Prep The System
- Go to [***(Setup) System ➜ (Server) System Settings ➜ Server Array Settings ➜ Server List***](https://powerschool.tulsaschools.org/admin/systemsettings/serverarray/serverlist.html).
- Make note of the "APP NODE" listed at the bottom left of the page.
- Find the corresponding entry in the server list and make note of the hostname.
- Pull up `\\[hostname]\c$\Program Files\PowerSchool\application\components\reporting-common-[version-number]\config` and drop in the contents of the (PowerSchool-provided) Ed-Fi integration plugin ZIP file, replacing any existing files.

#### Install The Plugin
- Go to [***(Setup) System ➜ (Data Management) Special Operations***](https://powerschool.tulsaschools.org/admin/tech/specop.html).
- Select "*Operation*" `Load Server Reports`, enter "*Code*" `solo flight`, and *Submit*.
- This may take anywhere between seconds (if plugin files could not be found) to ~1 minute (if the plugin files have already been loaded) to ~10 minutes (if the plugin is new).

#### Create A New DEX Profile
- Go to [***(Setup) System ➜ (Server) System Settings ➜ Plugin Management Configuration***](https://powerschool.tulsaschools.org/admin/pluginconsole/plugInConsole.action).
- Verify there is an entry for "*PowerSchool Data Exchange(DEX)*" and that the "*Enable*" box is checked.
- Click the "*PowerSchool Data Exchange (DEX)*" entry.
- Click on "*Profile Configuration*".
- Click on "*Add Profile +*", give the profile a name, select "Profile Type" `EDFI_TPS`, make sure it is enabled, and *Save*.
- Verify there is now an entry for this new profile's name under ***(Data Exchange)*** on the left-hand menu.

#### Configure The New DEX Profile
- Go to [***(Setup) System ➜ (Data Exchange) General Setup***](https://powerschool.tulsaschools.org/admin/tech/dexConfigs.html).
- Verify "*System Enabled*" is `On`.
- Select the newly-created (and named) profile.
- Enter "*Data Exchange URL*" `https://edfi.tulsaschools.org/api/api/v2.0`.
- Enter "*Authentication URL*" `https://edfi.tulsaschools.org/api`.
- Select the relevant "*School Years*" and "*Service Options*" boxes.
- Click "*Configure*" in the "LEA Name" section.
- Make sure "*LEA Enabled*" is set to `On`.
- Enter the "*Authentication Key*" and "*Authentication Secret*".
- Run *Test Connection* to verify connectivity, then *Save* and *Close* that dialog.
- *Save*.

#### Trouleshooting

To validate the plugin files were correctly prepped and loaded, run the following queries. Each should return a total of 19 records.

```sql
-- Determine list of reports defined in config/reportSDK.
SELECT * FROM CST_REPORTCONFIG WHERE REPORTNAME LIKE 'EDFI_TPS%'
```

```sql
-- Determine list of reports defined in config/dashboard/reports.
SELECT * FROM CST_DSHREPORTS WHERE REPORT_TITLE LIKE 'EDFI_TPS%'
```

```sql
-- Determine list of dashboard entries for district office using report definitions.
SELECT DISTINCT CST_DSHREPORTSUBMISSIONS.*
FROM
  CST_DSHREPORTSUBMISSIONS
  INNER JOIN
  CST_PUBPROFILE
  ON (CST_DSHREPORTSUBMISSIONS.CST_PUBPROFILEID = CST_PUBPROFILE.ID)
WHERE
  (SCHOOLID = 0)
  AND
  (YEARID = 28) -- TWEAK TO MATCH THE SELECTED SCHOOL YEAR(S).
  AND
  (MD_PROFILECODE='EDFI_TPS')
```


---


## Loading Data Into The ODS

#### Mapping Descriptors
- Click on the new profile's name under ***(Data Exchange)*** on the left-hand menu.
- Click on the *Run Now* button for the "Descriptors (Common Codes)" line.
- Refresh the page until the system shows these descriptor records are loaded (it may take quite some time).

###### Core Mappings
- Go to [***(Setup) System ➜ (Data Exchange) Core Set Mappings***](https://powerschool.tulsaschools.org/admin/tech/dexMappings.html).
- Select the new profile's name.
- For each "*Code Set*", fill in all of the "Selected Downloaded State Code" dropdowns and *Save* ¹.

###### Other Mappings
- Go to [***(Setup) System ➜ (Data Exchange) Core Set Mappings (Other)***](https://powerschool.tulsaschools.org/admin/tech/dexMiscMappings.html).
- Select the new profile's name.
- For each "*Code Set*", fill in all of the "Selected Downloaded State Code" dropdowns and *Save* ¹.

#### Initial Data Load
- Click on the new profile's name under ***(Data Exchange)*** on the left-hand menu.

###### "Publish" Data Load
For each of the "Publish Data" lines:
- Click *Run Now*.
- Select "Data Publishing Option" `PUBLISH ALL` from the popup dropdown.
- *Submit*.

Do this for each line, from top to bottom, waiting for each step to finish before moving on to the next. Depending on the size of the district and the volume of data available, **this may take anywhere from several hours to a few days**.

###### "On Demand" Data Load
For each of the "On Demand Data" lines:
- Click *Schedule*.
- Click *OK* on the native JS `alert(…)`.
- Select "Data Publishing Option" `PUBLISH ALL`.
- Check the *Run Now* radio button.
- *Submit*.

Do this for each line, waiting for each step to finish before moving on to the next. Order is not important here.

###### Schedule Ongoing "On Demand" Updates
For each of the "On Demand Data" lines:
- Click *Schedule*.
- Click *OK* on the native JS `alert(…)`.
- Select "Data Publishing Option" `PUBLISH CHANGES`.
- Check the *Schedule* radio button.
- Set a sensible "*Start Date*" and "*Start Time*".
- Check the *Repeat* radio button.
- Ensure the *Daily* and *No end date* radio buttons are checked.
- *Submit*.

Do this for each line, waiting for each step to finish before moving on to the next. Order is not important here.


---

## Troubleshooting

#### Items Stuck On Publishing Queue
The "Publishing" queue is for items scheduled for writing into the target Ed-Fi API. These items should be very short-lived. If ever you identify that the "Publishing" queue is not shrinking, it may be that PowerSchool has lost its connections to the target Ed-Fi API. To correct this:
- Go to [***(Setup) System ➜ (Data Exchange) General Setup***](https://powerschool.tulsaschools.org/admin/tech/dexConfigs.html).
- Select the relevant profile.
- Click "*Configure*" in the "LEA Name" section.
- Run *Test Connection* to verify connectivity, then *Save*.

This forces PowerSchool to re-establish the necessary API connections, and you should see the dashboard's "Publishing" queue start to shrink shortly thereafter.

#### Errors On "Publish" Data
Lines on the "Publish Data" table are arranged such that errors on one line may propagate downward as dependencies, so you should always work from top to bottom when addressing issues.

If anything ends up on the "Dependencies" queue or raises errors, click on that line's *Review* or *Errors* button to go to its "Review" page ². From there -- via the proper "*Choose category*" and "*Choose data view*" options, you should be able to see which school/term/class/section/student/staff/etc records are causing problems.

Sometimes certain cells on the table may be clickable for additional info. In some cases, you'll get the failed JSON payload or the error returned; in other cases, you simply get a link to the relevant resource (e.g. the student). When linked to a student, go to the ***(Data Exchange) Publishing*** link on the left-hand menu. This will give you access to more detailed error information and you are more likely to find JSON payloads and/or "Resource Details" tables to help identify missing data, etc.

---

¹ The "*descriptor-mappings.js*" file alongside this guide contains a script runnable in-browser that will set all dropdowns for the currently-displayed "Code Set". Be sure to consult the more detailed usage instructions in the JS file.

² When working in "Review" pages, *always* click *Clear Cache* immediately after loading the page or changing either the "*Choose category*" or "*Choose data view*" dropdowns. **If you fail to do this, you may be looking at old data and attempting to track down or correct a problem that no longer exists.**
