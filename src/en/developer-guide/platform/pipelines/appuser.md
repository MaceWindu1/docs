---
title: Automated App User Import
tagline: Setting up a pipeline for automated deploy of appuser
author: Zdeněk Šrejber
---

# **The set up**
## The goal
We want to import application user automatically via pipeline, have same unique identifier on each environment and keep the process as simple as possible.

## The problem
We can not use data package, because when you create app user you need to provide business unit id. Microsoft does this for you when you create application users manually through [admin.powerplatform.microsoft.com](https://www.admin.powerplatform.microsoft.com). However with pipeline you have to use API, Microsoft doesnt do anything for you and the businessunitid is always different for every environment.

## The solution
INT0014-Platform was expanded by a function which finds a business unit to be used and creates the app user with found business unit. You can call this function via 
```C#
DataImport.CreateAppUser(crmServiceClient, AppUser data)
```
- crmServiceClient is a client to connect to.
- AppUser is a simple class of two guid parameters which you need to provide.
```C#
public class AppUser
        {
            public Guid applicationid;
            public Guid systemuserid;
        }
```

## The sample use in project
The guids for app users can be stored in JSON file, named Setting.json in our case.
```JSON
{
   "Data": {
      "AppUsers": [
         { //App User 1
            "systemuserid": "0060dea9-5ea1-4295-af02-4d8639827980",
            "applicationid": "af5bc2bd-f2cb-4ac2-b7bf-18b7a8d261e7"
         },
         { //App User 2
            "systemuserid": "d3a8c139-f01a-47d7-ad6f-b9a17c2be85b",
            "applicationid": "6c7fe6d1-7f3b-4c0c-98fb-374d0043e7b2"
         }
      ]
   }
}
```
In the code you then need to read the file, parse it and call CreateAppUser, eg.
```C#
private string _pathToSettingsJson
    {
      get
      {
        return string.Format("{0}\\{1}\\settings.json", Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location), GetImportPackageDataFolderName);
      }
    }

    private JObject _settings;

    public JObject Settings
    {
      get
      {
        if (_settings == null || _settings == default(JObject))
        {
          _settings = JObject.Parse(File.ReadAllText(_pathToSettingsJson));
        }

        return _settings;
      }
    }

private void CreateAppUsers()
    {
      JArray AppUsers = (JArray)Settings["Data"]["AppUsers"];
      AppUsers.ToList().ForEach(
          x =>
          {
            var data = new DataImport.AppUser()
            {
              applicationid = new Guid((string)x["applicationid"]),
              systemuserid = new Guid((string)x["systemuserid"])
            };
            DataImport.CreateAppUser(CrmSvc, data);
          });
    }
```

## Limitations
Currently, the process was developed for use with TALXIS Portal solutions, where business unit entity is not used. Thus there is always just one unit, generated by Microsoft. If your project uses multiple business units, the implementation might need to be changed so that the function adds the app user under correct business unit. Suggested solution would be to order business units by 'CreatedOn' and get the oldest.
## Suggested improvements
The CreateAppUser function could be customized so that systemuserid is optional parameter.