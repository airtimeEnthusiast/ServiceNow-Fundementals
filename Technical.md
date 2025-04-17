# ServiceNow Fundamentals
- A collection of tasks using a service-now PDI. 
## Table of Contents
- [Updates](#updates)
- [Access Control List (ACL)](#access-control-list-acl)
- [Auto-Close Timer](#auto-close-timer)
- [Notifications](#notifications)
- [Scheduled Notifications](#scheduled-notifications)
- [Incident Lifecycle](#incident-lifecycle)
- [Mandatory Fields](#mandatory-fields)
- [Scheduled Incident Updates](#scheduled-incident-updates)
- [Update Incident Priority Calculation](#update-incident-priority-calculation)
- [Update Incident Priority For VIPs](#update-incident-priority-for-vips)
- [Create a Catalog Item To Elevate Role](#create-a-catalog-item-to-elevate-role)
- [Make changes to update Incident SLAs](#make-changes-to-update-incident-slas)
- [Create Reporting User Groups](#create-reporting-user-groups)

---

## Updates
- Every task is created within an Update Set. Access local sets under **System Update Sets -> Local Update Sets**.

<p align="center">
  <img src="assets/asset1.png?raw=true" alt="Update Set"/>
</p>

- Tasks are marked as completed once the changes have been added below.

<p align="center">
  <img src="assets/asset2.png?raw=true" alt="Update Set Record"/>
</p>

---


## Access Control List (ACL)
### Description
Ensure that the **Incident Number** field is read-only so that system users cannot manually edit the record number.

### Steps
#### 1. Verify User Permissions
- As a **System Administrator**, elevate your role to ensure you have the `security_admin` role.
- Navigate to **Organization -> Users** and verify the role assignment.

<p align="center">
  <img src="assets/ACL/asset9.png?raw=true" alt="Verify Admin Role"/>
</p>

- Elevate the role by selecting the profile icon.

<p align="center">
  <img src="assets/ACL/asset10.png?raw=true" alt="Elevate Role"/>
</p>

#### 2. Configure ACL
- Navigate to **System Security -> Access Control (ACL)**.
- Create a new ACL as shown below:

<p align="center">
  <img src="assets/ACL/asset11.png?raw=true" alt="Create ACL"/>
</p>

#### 3. Add a Script Condition
- Restrict all users except admins from modifying the **Incident Number** field.

```javascript
// Deny write access to the Incident number field
answer = false;
```

- When impersonating a user, the field should be grayed out, preventing modification.

<p align="center">
  <img src="assets/ACL/asset12.png?raw=true" alt="Read-only Field"/>
</p>

---

## Auto-Close Timer
### Description
Incidents should automatically close **3 days after being resolved**.

### Steps
1. Navigate to **Incident > Administration > Incident Properties**.
2. Adjust the time for resolved incidents to close and **uncheck** 'Close open incident tickets'.

<p align="center">
  <img src="assets/timer/asset15.png?raw=true" alt="Auto-Close Timer"/>
</p>

---

## Notifications
### Description
Notify the **Assignment Group Manager** when a critical incident is assigned to their group.

### Steps
1. Access **System Notification -> Email -> Notifications**.
2. Configure the notification settings as follows:

<p align="center">
  <img src="assets/notification/asset3.png?raw=true" alt="Notification Setup"/>
</p>
<p align="center">
  <img src="assets/notification/asset4.png?raw=true" alt="Notification Settings"/>
</p>
<p align="center">
  <img src="assets/notification/asset5.png?raw=true" alt="Notification Example"/>
</p>

#### Testing
- Assign a **Critical Priority Incident** to an assignment group.
- Check the sent emails log:
  - Navigate to **System Mailboxes -> Outbound -> Outbox**.

---

## Scheduled Notifications
### Description
Send a **weekly email** (every Monday) to the **Application Development Manager** with the count of active incidents.

### Steps
1. Configure the **Flow Designer** as shown below:

<p align="center">
  <img src="assets/schedule/asset6.png?raw=true" alt="Flow Designer"/>
</p>

2. Define a **New Workflow Action** with the following script:
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


## Incident Lifecycle
### Description
Ensure incidents follow the state transitions as per the diagram below:

<p align="center">
  <img src="assets/lifecycle/asset16.png?raw=true" alt="Incident Lifecycle"/>
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


## Mandatory Fields
### Description
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


#### Client Script
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


## Scheduled Incident Updates
### Description
Automatically move **On Hold** incidents back to **In Progress** based on predefined conditions.

#### Following Cases
1. If the reason is Awaiting Caller, after 3 business days (M-F) if not updated by the Caller
2. If the reason is Awaiting Change, when the related Change is closed
3. If the reason is Awaiting Problem, when the related Problem is closed
4. If the reason is Awaiting Vendor, after 7 business days (M-F)
#### Flow Implementation
- Create flow that triggers daily at 12pm with the following structure.
<p align="center">
  <img src="assets/schedule/asset24.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/schedule/asset25.png?raw=true" alt=""/>
</p>

- For each case, add the following flow logic and action to update the queried records.
<p align="center">
  <img src="assets/schedule/asset23.png?raw=true" alt=""/>
</p>

#### CASE 1: Script the 'Look Up Records' condition
```javascript
  // Retrieve the business day with holiday exclusion schedule. 
  var sgr = new GlideRecord('cmn_schedule');
  sgr.addQuery('name','8-5 weekdays excluding holidays');
  sgr.query();

  if (sgr.next()) {
    var schedule = new GlideSchedule(sgr.sys_id);
    
    // Calculate the time from 3 business days ago
    var now = new GlideDateTime();
    var startTime = schedule.getPreviousStartDate(now, new GlideDuration('72:00:00')); // 72 hours (3 business days)


    // Retrieve the glide record that has been awaiting caller for 3 business days.
    var gr = new GlideRecord('incident');
    gr.addQuery('state', '3');
    gr.addQuery('hold_reason', 'Awaiting Caller');
    gr.addQuery('sys_updated_on', '<=', startTime);
    gr.query();

    var records = [];

    while (gr.next()) {
        records.push(gr.getEncodedQuery());
        gs.info('Incident found: ' + gr.number + ' | Last updated: ' + gr.sys_updated_on);
    }
    return records.join('^OR'); // Return an OR-separated encoded query string
  }
  gs.info('No Incidents found');
  return "";
```

#### CASES 2-3: Use the Conditional Builder
- Case 2: Awaiting Change, when the related Change is closed
<p align="center">
  <img src="assets/schedule/asset26.png?raw=true" alt=""/>
</p> 

- Case 3: Awaiting Problem, when the related Problem is closed
<p align="center">
  <img src="assets/schedule/asset27.png?raw=true" alt=""/>
</p> 


#### CASES 4: Script the 'Look Up Records' condition
```javascript
// Retrieve the business day with holiday exclusion schedule. 
var sgr = new GlideRecord('cmn_schedule');
sgr.addQuery('name','8-5 weekdays excluding holidays');
sgr.query();

if (sgr.next()) {
    var schedule = new GlideSchedule(sgr.sys_id);
    
    // Calculate the time from 7 business days ago
    var now = new GlideDateTime();
    var startTime = schedule.getPreviousStartDate(now, new GlideDuration('168:00:00')); // 168 hours (7 business days)


    // Retrieve the glide record that has been awaiting caller for 3 business days.
    var gr = new GlideRecord('incident');
    gr.addQuery('state', '3');
    gr.addQuery('hold_reason', 'Awaiting Caller');
    gr.addQuery('sys_updated_on', '<=', startTime);
    gr.query();

    var records = [];

    while (gr.next()) {
        records.push(gr.getEncodedQuery());
        gs.info('Incident found: ' + gr.number + ' | Last updated: ' + gr.sys_updated_on);
    }
    return records.join('^OR'); // Return an OR-separated encoded query string
}
gs.info('No Incidents found');
return "";
```


## Update Incident Priority Calculation
The Incident priority calculation will be updated so that an Incident with a **Medium impact** does not have a priority less than **Moderate**
An Incident priority calculation should be updated so that when it considered a medium impact when:
1. The Urgency is High, the Priority is Critical
2. The Urgency is Low, the Priority is Moderate

To update the incident priority calculation by first accessing **Alls-> System Policy -> Rules -> Priority Lookup Rules**
<p align="center">
  <img src="assets/priority/asset28.png?raw=true" alt=""/>
</p>

- In the proirity data lookup table, records, *100*, *300*, 
*600* will be modified to reflect the rules.
<p align="center">
  <img src="assets/priority/asset29.png?raw=true" alt=""/>
</p>

#################### UPDATE GRAMMER 

## Update Incident Priority For VIPs
### Description
As the Incident Manager, I want the system to always set the Priority to Critical if the Incident's caller is a VIP so that their tickets are always treated with urgency.

#### Flow Implementation
- Create flow that triggers when an incident is generated that will update the priority to critical if the caller is a VIP.

<p align="center">
  <img src="assets/vip/asset30.png?raw=true" alt=""/>
</p> 

<p align="center">
  <img src="assets/vip/asset31.png?raw=true" alt=""/>
</p> 

<p align="center">
  <img src="assets/vip/asset32.png?raw=true" alt=""/>
</p> 


## Secure User Records
### Description 
Prevent other sys_users from editing records apart from their own. 

#### Create a new write ACL
<p align="center">
  <img src="assets/secure_records/asset33.png?raw=true" alt=""/>
</p> 
<p align="center">
  <img src="assets/secure_records/asset34.png?raw=true" alt=""/>
</p> 

#### Script Implementation
```javascript
  // Allow if user is admin, user_admin, or updating their own record
if (gs.hasRole('admin') || 
	gs.hasRole('user_admin') || 
	current.sys_id == gs.getUserID()) {
    answer = true;
}
else{
	answer = false;
}
```

#### Disable other ACL's that prevent writes
Other ACL's that act on the fields of the sys_user need to be disabled.
<p align="center">
  <img src="assets/secure_records/asset35.png?raw=true" alt=""/>
</p>

#### Disable ACL user* read 
Disabling this ACL allows other users to read other user's records.
<p align="center">
  <img src="assets/secure_records/asset36.png?raw=true" alt=""/>
</p> 

#### Create a new Write ACL
<p align="center">
  <img src="assets/secure_records/asset37.png?raw=true" alt=""/>
</p> 

```javascript
if (gs.getUserID() == current.sys_id || gs.getUser().hasRoles()) 
    answer = true;
else 
    answer = false;
```


## Create a Catalog Item To Elevate Role
#### Description 
Create a catalog item that will automate the addition of users to specific groups, granting them additional access within the platform, so this process does not have to be managed manually.
<p align="center">
  <img src="assets/catalog/asset0.png?raw=true" alt=""/>
</p> 
Catalog item should implement a flow that confirms to the following diagram.
<p align="center">
  <img src="assets/catalog/assetflow.png?raw=true" alt=""/>
</p> 

#### Create a Catalog Item
Navigate to *Service Catalog->Catalog->Maintain Items* to create a new catalog item.
<p align="center">
  <img src="assets/catalog/asset1.png?raw=true" alt=""/>
</p> 
Modify the portal settings to include an on-submit button.
<p align="center">
  <img src="assets/catalog/asset2.png?raw=true" alt=""/>
</p>

### Create the Catalog Variables in the following order
<p align="center">
  <img src="assets/catalog/asset3.png?raw=true" alt=""/>
</p>

#### Requested_For Variable
This variable references the sys_user table and will load the current user into the field.
<p align="center">
  <img src="assets/catalog/asset4.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/catalog/asset5.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/catalog/asset6.png?raw=true" alt=""/>
</p>

#### Autopopulate Variables
The following variables ```phone```, ```department```, and ```email``` all implement the autopopulate function that is dependent on the ```requested_for``` field.
<p align="center">
  <img src="assets/catalog/asset7.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/catalog/asset8.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/catalog/asset9.png?raw=true" alt=""/>
</p>

#### Request Access Variable
References the *sys_user_group* table.
<p align="center">
  <img src="assets/catalog/asset10.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/catalog/asset11.png?raw=true" alt=""/>
</p>

#### Client Script to Autopopulate fields
<p align="center">
  <img src="assets/catalog/asset12.png?raw=true" alt=""/>
</p>

```javascript
function onChange(control, oldValue, newValue, isLoading) {
   if (isLoading || newValue == '') {
      return;
   }
    // Get reference field (sys_user)
   var req_for = g_form.getReference('requested_for', setFields);

   //Set the fields of the catalog form
   function setFields(req_for){
        var deptID = req_for.department;	// Retrieve the department sys_id

		// Intialize and fetch the department record
        var grDept = new GlideRecord('cmn_department');
		if(grDept.get(deptID)){
			g_form.setValue('department', grDept.name); // Set department name.
		}
		// Set email and phone
		g_form.setValue('email',req_for.email);
		g_form.setValue('phone',req_for.mobile_phone);
	}
}
```

#### Client Script to Get the Current User
<p align="center">
  <img src="assets/catalog/asset13.png?raw=true" alt=""/>
</p>

```javascript
function onLoad() {
   //Set the g_form field 'requested_for' to the current user. 
   g_form.setValue('requested_for', g_user.userID);
}
```

#### Catalog Item Form<p align="center">
- Create and customize the layout of the form by accessing: *Service Catalog->Catalog Builder*
  <img src="assets/catalog/asset15.png?raw=true" alt=""/>
</p>

#### Implement the Flow With a Catalog Item Trigger
- Create a flow variable approval_count (integer) to count the number of manager approvals.
<p align="center">
  <img src="assets/catalog/assetFlowDesign.png?raw=true" alt=""/>
</p>


## Make changes to update Incident SLAs 
- The Incident SLAs need to be updated to run for the times outline below. Additionally, I do not want SLAs to run outside of business hours, so that our metrics accurately reflect our hours of operation (8AM to 5PM). SLAs should also not run during US holidays.

- Priority 1 resolution: 4 hours
- Priority 2 resolution: 1 business days
- Priority 3 resolution: 2 business days
- Priority 4 resolution: 5 business days

Modify the SLA definitions by accesing *Service Level Mangement -> SLA -> SLA Definitions*
<p align="center">
  <img src="assets/SLAs/asset1.png?raw=true" alt=""/>
</p>
<p align="center">
  <img src="assets/SLAs/asset2.png?raw=true" alt=""/>
</p>


## Create Reporting User Groups

Create the following groups:

<p align="center">
  <img src="assets/Groups/asset2.png?raw=true" alt=""/>
</p>

- Users able to view reports shared with them and create reports for themselves
<p align="center">
  <img src="assets/Groups/asset1.png?raw=true" alt=""/>
</p>

- Users able to create and share reports with groups they are members of
<p align="center">
  <img src="assets/Groups/asset3.png?raw=true" alt=""/>
</p>

- Users able to create, share, and publish report
<p align="center">
  <img src="assets/Groups/asset4.png?raw=true" alt=""/>
</p>

## Prevent Incident state from being edited in the list view
- Create an onCellEdit *Client Script* 
<p align="center">
  <img src="assets/ACL/asset13.png?raw=true" alt=""/>
</p>

- Create the script with the following newValue condition.
```javascript
function onCellEdit(sysIDs, table, oldValues, newValue, callback) {
  var saveAndClose = true;
  //Type appropriate comment here, and begin script below
  if(newValue.name === 'state'){
	  alert("Unable to edit the 'State' field from the list view is not allowed.");
	  var saveAndClose = false;
 }
 
 callback(saveAndClose); 
}
```

## Update the Service Portal
- Update the layout of the service portal layout so users can see their open