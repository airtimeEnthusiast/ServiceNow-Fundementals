# ServiceNow Fundementals 

## Table Of Contents
- [Updates](#updates)
- [Notification](#notifications)
- [Scheduled Notifications](#scheduled-notifications)
- [User Administration](#user-administration)
---
## Updates
- Every task is created with an Update Set. Access the local sets Under *System Update Sets -> Local Update Sets*

<p align="center">
  <img src="assets/asset1.png?raw=true" alt="update set"/>
</p>

- Tasks are marked as completed once the changes have been added below

<p align="center">
  <img src="assets/asset2.png?raw=true" alt="update set record"/>
</p>





## Notification
#### Description: 
Notify the assignment group manager when a critical incident is assigned to their group. 
``` xml
As the Incident Manager, I want group managers to be alerted 
whenever a critical Incident is assigned to their group so that 
they maintain visibility into critical incidents assigned to their group(s)
```
#### Access the Notification Records under *System Notification -> Email -> Notifications*
ass
#### Build the notification as follows.
<p align="center">
  <img src="assets/notification/asset3.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/notification/asset4.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/notification/asset5.png?raw=true" alt=""/>
</p>

### 3. Testing: 
 - Assign a Critical Priority Incident to the Assignment Group.
 - Check the Sent Emails log to confirm delivery:
 - Navigate to System Mailboxes → Outbound → Outbox



## Scheduled Notifications
#### Description: 
Requested Notification: Application Development incident count
``` xml
As the manager of the Application Development group, 
I would like to receive an email every Monday with 
the number of active Incidents assigned to me at the 
start of the week so that I can prioritize and 
balance team workload.
```
### Access the Notification Records 
under *System Definition -> Scheduled Jobs*
### Build the notification as follows.
<p align="center">
  <img src="assets/schedule/asset6.png?raw=true" alt=""/>
</p>

### Script as follows.
```javascript
(function executeIncidentCount(){
	// Incident glide record
	var gr = new GlideRecord('incident');

	// Build Query for all active critical incidents assigned from the start of the week to Bushra Akhtar (Application Development Manager)
	gr.addEncodedQuery('assigned_to=506c0f9cd7011200f2d224837e61030f^state!=7^ORstate!=8^ORstate!=6^opened_atONLast week@javascript:gs.beginningOfLastWeek()@javascript:gs.endOfLastWeek()^priority=1');
	gr.query();

	// Create and format dates as MM/DD
	var now = new GlideDateTime(); // Current date and time
	var startOfWeek = new GlideDateTime();
	startOfWeek.setDisplayValue(now.getWeekOfYearLocalTime() + "-1");// Start of the week (Sat) 
	var startFormatted = startOfWeek.getMonthUTC() + 1 + "/" + startOfWeek.getDayOfMonthUTC();
	var endOfWeek = new GlideDateTime();
	endOfWeek.setDisplayValue(now.getWeekOfYearLocalTime() + "-7");// End of the week (Sun)
	var endFormatted = endOfWeek.getMonthUTC() + 1 + "/" + endOfWeek.getDayOfMonthUTC();


	// Get Count
	var count = gr.getRowCount();

	// Format Email
	if(count > 0 ){
		var email = new GlideEmailOutbound();
		email.setSubject("Your Critical Incidents Count for the Week of " + startFormatted + " - " + endFormatted + ".");
		email.setBody("There are currently " + count + " open critical incidents assigned to you.");
		email.setTo("bushra.akhtar@example.com");
		email.send();
	}
	// Log script
	gs.info("Scheduled Job executed: " + count + " critical incidents found for " + startFormatted + " - " + endFormatted);
})();
```

### 3. Testing: 
 - Execute Script and check logs: *System Logs → All (syslog.list)*



## User Administration
#### Description: 
As the system admin, I want groups created with the appropriate roles to support three levels of reporting power user, so that we follow best practice for role management
- Users able to view reports shared with them and create reports for themselves
- Users able to create and share reports with groups they are members of
- Users able to create, share, and publish reports