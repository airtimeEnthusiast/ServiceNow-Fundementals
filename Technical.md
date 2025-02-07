# ServiceNow Fundementals 

## Table Of Contents
- [Updates](#updates)
- [ACL](#access)
- [List](#list)
- [Auto-close Timer](#auto-close)
- [Notification](#notification)
- [Scheduled Notifications](#scheduled-notifications)
- [Life Cycle](#lifecycle)
- [Mandatory](#mandatory)

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


## List
#### Description: 
Update the list view of Incidents for the Incident Manager to include 
the following fields.
- Number
- Caller
- Caller's location
- Short Description
- Priority
- State
- Assignment group
- Assigned to
- Created
- Updated

# Inpersonate and navigate to the *Service Desk -> Incidents*
- Inpersonate as 'Incident Manager'
<p align="center">
  <img src="assets/List/asset13.png?raw=true" alt="update set"/>
</p>

<p align="center">
  <img src="assets/List/asset14.png?raw=true" alt="update set"/>
</p>

---

## Auto-Close
#### Description: 
Incidents should automatically be closed after 3 days after being resolved. 

- Navigate to *Incident > Administration > Incident Properties*
- Adjust the time for resolved incidents to close and uncheck 'Close open incident tickets'
<p align="center">
  <img src="assets/Timer/asset15.png?raw=true" alt="update set"/>
</p>



## Access
#### Description: 
Ensure that the number field on Incident is read-only and so that system users aren't able to manually edit a record's number.

#### Navigate to Users
- As system admin you must elevate your role check to make sure the admin has the security_admin role under *Organization -> Users*
<p align="center">
  <img src="assets/ACL/asset9.png?raw=true" alt="update set"/>
</p>
- Elevate the role by selecting the profile icon.
<p align="center">
  <img src="assets/ACL/asset10.png?raw=true" alt="update set"/>
</p>

##### Access the ACL table
- Navigate to *System Security -> Access Control (ACL)*
- Create a new ACL as follows. 
<p align="center">
  <img src="assets/ACL/asset11.png?raw=true" alt="update set"/>
</p>

### Create A script condition
- Restrict all users except admin to modify the number field.
```javascript
    // Deny write access to the Incident number field
    answer = false;
```
- When inpersonating a user, the field should be grayed out where it cannot be modified.
<p align="center">
  <img src="assets/ACL/asset12.png?raw=true" alt="update set"/>
</p>

---




## Notification
#### Description: 
Notify the assignment group manager when a critical incident is assigned to their group. 

- Access the Notification Records under *System Notification -> Email -> Notifications*

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

---

## Scheduled Notifications
#### Description: 
Requested Notification: Application Development incident count

- Send an email every Monday with the number of active Incidents 
assigned to the Application Development Manager at the start of the week.

### Build the flow as follows.
<p align="center">
  <img src="assets/schedule/asset6.png?raw=true" alt=""/>
</p>

### Define an new WorkFlow Action with a script as follows.
```javascript
(function execute(inputs, outputs) {
    // Incident glide record
	var gr = new GlideRecord('incident');

	// Build Query for all active critical incidents assigned from the start of the week to Bushra Akhtar (Application Development Manager)
	gr.addEncodedQuery('assigned_to=506c0f9cd7011200f2d224837e61030f^state!=7^ORstate!=8^ORstate!=6^opened_atONLast week@javascript:gs.beginningOfLastWeek()@javascript:gs.endOfLastWeek()^priority=1');
	gr.query();

    // Create and format dates as MM/DD
	var now = new GlideDateTime(); // Current date and time

    // Get incident links of links
	var count = 0;
	var incidentList = "";
    while(gr.next()){
        count++;
        var incidentNum = gr.number.getDisplayValue();
        var incidentLink = "http://" + gs.getProperty('instance_name') + ".service-now.com/incident.do?sys_id=" + gr.sys_id;
        incidentList += "<li><a href='" + incidentLink + "'>" + incidentNum + "</a></li>\n";
    }

    // Query the user group table
    var grM = new GlideRecord('sys_user_group');
    var managerGR;
    var managerName = "";
    var managerEmail = "";
    // Query for the group with the name "Application Development"
    grM.addQuery('name', 'Application Development');
    grM.query();

    // If the group is found, retrieve the manager's info
    if (grM.next()) {
        managerGR = grM.manager.getRefRecord();
        managerEmail = managerGR.getValue('email');
        managerName = managerGR.getValue('name');

        gs.info("Application Development Manager: " + managerName);
    }   
    else {
        gs.info("Group not found.");
    }

    // Build Email Body
    var emailBody = "<p>Hello " + managerName + ",</p>";
    emailBody += "<p> This week your group has <b>" + count + "</b> active incidents. See below for the list of incidents:</p>";
    emailBody += "<ul>" + (incidentList) + "</ul>";
	outputs.emailbody = emailBody;
    outputs.manageremail = managerEmail;
    
})(inputs, outputs);

```

### Create the script output variables
<p align="center">
  <img src="assets/schedule/asset7.png?raw=true" alt=""/>
</p>

### Map the script output to the Action output variables
<p align="center">
  <img src="assets/schedule/asset8.png?raw=true" alt=""/>
</p>

### 3. Testing: 
 - Execute Flow and check *System Mailbox -> Outbox*

---


## Lifecycle
#### Description
Enforce a lifecycle for incident states from the diagram below
<p align="center">
  <img src="assets/lifecycle/asset16.png?raw=true" alt=""/>
</p>

#### Create a Business Rule for Incident Transitions
<p align="center">
  <img src="assets/lifecycle/asset17.png?raw=true" alt=""/>
</p>

```javascript
(function executeRule(current, previous /*null when async*/) {
	// Valid state transitions
	var validTransitions = {
    	'1': ['2', '8'], // New → In Progress or Canceled
    	'2': ['3', '6'], // In Progress → Resolved or On Hold
    	'3': ['2', '8', '7'], // Resolved → In Progress, Canceled, or Closed
    	'6': ['2'], // On Hold → In Progress
	};

	// Get previous and new state values
	var prevState = previous.state.toString();
	var newState = current.state.toString();

	// Check if transition is valid
	if (validTransitions[prevState] && validTransitions[prevState].includes(newState)) {
    	gs.info('Valid transition from ' + prevState + ' to ' + newState);
	} 
	else {
    	gs.addErrorMessage('Invalid transition from ' + prevState + ' to ' + newState);
    	current.state = previous.state; // Revert to previous state
    	current.setAbortAction(true);
	}
})(current, previous);
```


## Mandatory
#### Description
Enforce the following rules and create additional fields for On Hold reason: 

- If the On Hold reason is Awaiting Change, the Change Request field is mandatory.
- If the On Hold reason is Awaiting Problem, the Problem field is mandatory.
- If the On Hold reason is Awaiting Vendor, the agent is able to associate the Incident with a Vendor and the field is mandatory.
#### Check hold reasons
- Inspect the html element of the Hold Reason Choice List
<p align="center">
  <img src="assets/mandatory/asset19.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/mandatory/asset20.png?raw=true" alt=""/>
</p>

- Right click on the fields to find their name
<p align="center">
  <img src="assets/mandatory/asset22.png?raw=true" alt=""/>
</p>

### Create Vendor Field in Incident Form
- Configure the field as a reference to the *core_comopany* table.
<p align="center">
  <img src="assets/mandatory/asset21.png?raw=true" alt=""/>
</p>


#### Create an OnChange Client Script 
```javascript
function onChange(control, oldValue, newValue, isLoading, isTemplate) {
        if (isLoading || newValue == '') {
        return;
    }

    // Ensure the On Hold Reason field exists
    if (!g_form.hasField('hold_reason')) {
        return;
    }

    var onHoldReason = g_form.getValue('hold_reason'); 

    // Awaiting Change (Value = 5) -> Make Change Request mandatory
    if (onHoldReason == '5') {
        g_form.setMandatory('rfc', true);
        g_form.showFieldMsg('rfc', 'Change Request is required when On Hold Reason is Awaiting Change.', 'error');
    } else {
        g_form.setMandatory('rfc', false);
        g_form.hideFieldMsg('rfc');
    }

	// Awaiting Problem (Value = 3) -> Make Problem field mandatory
    if (onHoldReason == '3') {
        g_form.setMandatory('problem_id', true);
        g_form.showFieldMsg('problem_id', 'Problem is required when On Hold Reason is Awaiting Problem.', 'error');
    } else {
        g_form.setMandatory('problem_id', false);
        g_form.hideFieldMsg('problem_id');
    }

	// Awaiting Vendor (Value = 4) -> Make Vendor field mandatory
    if (onHoldReason == '4') {
        g_form.setMandatory('u_vendor', true);
        g_form.showFieldMsg('u_vendor', 'Vendor is required when On Hold Reason is Awaiting Vendor.', 'error');
    } else {
        g_form.setMandatory('u_vendor', false);
        g_form.hideFieldMsg('u_vendor');
    }
}
```