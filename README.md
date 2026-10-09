# IT Request Hub

A Power Apps and SharePoint system I built during my Systems Administration and ICT Support internship at EverAbility Group in Perth. It replaced third party web forms for staff onboarding and offboarding requests with one app that saves every request to SharePoint and emails the IT team automatically.

**Stack:** Power Apps (canvas app), SharePoint lists, Power Automate, Power Fx, HTML email templates

> This repository is a sanitised write up of my own work. It contains no company data, internal links, staff names or email addresses.

## The problem

IT requests for new starters and leavers came in through external web forms. Each request landed in an inbox as a block of text. There was no record in one place, no way to see how urgent a request was, and the dropdown options (departments, locations, systems) had to be edited inside the form tool.

## What I built

1. **Two request forms in a single Power Apps app.**
   - Onboarding: new starter details, access requests, equipment and building access.
   - Offboarding: finish date, mailbox and OneDrive handling, and which systems to disable.
2. **Three SharePoint lists.**
   - `Onboarding_Requests` and `Offboarding_Requests` store each submission.
   - `Form_Options` holds every dropdown and checkbox option, so IT can change them without touching the app.
3. **Two Power Automate flows** that email the IT team when a request is created. The subject line starts with a coloured circle for priority, so the urgent ones stand out in the inbox.
4. **Access control** using item level permissions on the lists and a restricted test page on the ICT Hub site while the system was in development.

```
Power App (Submit)  →  SharePoint list  →  Power Automate flow  →  Email to IT
```

## Features

- **Automatic priority.** The app works out how many days remain until the key date (finish date or action by date). Under 3 days is Severe, under 7 is High, otherwise Normal. A heads up message shows on the form when the date is close.
- **Options driven by data.** Dropdowns and checkbox groups read from one list, filtered by category, active flag and sort order. Adding an option is a row in a list, not an app change.
- **Conditional sections.** The access, equipment and building sections only appear when the request type needs them.
- **Form resets.** Each form clears itself when a user returns, so nobody sees the last person's answers.
- **Clear errors.** A failed save shows the real error message instead of failing silently.

## How it works

### Priority

```
If(DateDiff(Today(), dpAction.SelectedDate, TimeUnit.Days) < 3, "Severe",
   DateDiff(Today(), dpAction.SelectedDate, TimeUnit.Days) < 7, "High",
   "Normal")
```

The result is saved to a SharePoint choice column, which means it is written as `{Value: "Severe"}` rather than plain text.

### Options from a list

On start, the app loads the options list into a collection:

```
ClearCollect(colOptions, Form_Options)
```

Each dropdown then filters that collection:

```
Distinct(Sort(Filter(colOptions, Category = "Department", Active = "Yes"), SortOrder), Title)
```

### Email subject with a severity colour

```
concat(
  if(equals(triggerOutputs()?['body/field_6/Value'],'Severe'),'🔴 Severe',
  if(equals(triggerOutputs()?['body/field_6/Value'],'High'),'🟠 High','🟢 Normal')),
  ': Offboarding request for ', coalesce(triggerOutputs()?['body/Title'],''))
```

## Problems I solved

| Problem | Cause | Fix |
|---|---|---|
| Dropdown showed no options | The options collection loaded once at app start | Re run the start formula after changing the list |
| Submit did nothing | The formula read checkbox slots that did not exist | Used `Last(FirstN(...)).Title` to read only the slots in use |
| Flow failed with a null field error | Lists created from Excel use internal names like `field_6` in Power Automate, not the display names | Mapped each field by its internal name |
| Emails showed dates one day early | SharePoint stores dates in UTC | Converted to Perth time with `convertTimeZone` |
| Severity always showed as Normal | The choice column value is nested under `/Value` | Read `field_N/Value` instead of `field_N` |
| New flow sent no emails | A flow made with Save As starts switched off | Turned the flow on and added it to my go live checklist |
| Page showed old answers when revisited | No reset when the screen opened | Added a reset on each screen's `OnVisible` |

## What I learned

- Designing a data model first (lists, column types, choice fields) makes the app and the automation much easier to build.
- Keeping configuration in a list instead of hard coding it means the person who owns the process can maintain it.
- Time zones and internal field names are the two things most likely to break an automation quietly, so I test with real submissions before calling a flow done.
- Permissions are part of the build. I tested with a restricted page and item level permissions before anything was shared with staff.
- Writing the setup guide as a step by step handover document forced me to check every step actually works for someone else.

## Documentation

I also wrote a full handover guide covering list setup, the app, the flows, sharing and troubleshooting, so the IT team can rebuild or maintain it without me.

## Status and next steps

The system is working in testing. Still to do: read email recipients from a routing table instead of the flow, add approval steps, and extend the app with more request types.

## About me

Himel Banik, cybersecurity graduate (Master of Cybersecurity, Edith Cowan University) based in Perth, working in IT support and systems administration.

<!--
Add screenshots to the screenshots folder (with any names, emails and company data blurred), then uncomment:
![Home screen](screenshots/home.png)
![Onboarding form](screenshots/onboarding.png)
![Email notification](screenshots/email.png)
-->
