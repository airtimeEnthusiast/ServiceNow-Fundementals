#### Return an On Hold Incident to In Progress 
Have the system automatically return an Incident to In Progress if it's currently 
On Hold when the reason it's on hold has been resolved. The logic for cases 1 & 4 is untested 

## Following Cases
1. If the reason is Awaiting Caller, after 3 business days (M-F) if not updated by the Caller
2. If the reason is Awaiting Change, when the related Change is closed
3. If the reason is Awaiting Problem, when the related Problem is closed
4. If the reason is Awaiting Vendor, after 7 business days (M-F)


### Option 2A: Create a scheduled job
- Schedule the job to execute daily to handle cases 1 & 4.
```javascript
(function executeOnHold(){
	// Retrieve the business day with holiday exclusion schedule. 
	var sgr = new GlideRecord('cmn_schedule');
	sgr.addQuery('name','8-5 weekdays excluding holidays');
	sgr.query();
	// Glide Schedule
	var schedule = new GlideSchedule(sgr.sys_id);
    
  // Calculate the time from 3 & 7 business days ago
  var now = new GlideDateTime();
  var from3days = schedule.getPreviousStartDate(now, new GlideDuration('72:00:00')); // 72 hours
	var from7days = schedule.getPreviousStartDate(now, new GlideDuration('168:00:00')); // 168 hours


  // Retrieve the glide record that has been awaiting caller for 3 business days.
  var grCaller = new GlideRecord('incident');
    grCaller.addQuery('state', '3');
    grCaller.addQuery('hold_reason', 'Awaiting Caller');
    grCaller.addQuery('sys_updated_on', '<=', from3days);
    grCaller.query();
	while(grCaller.next()) {
    	grCaller.state = '2';
		grCaller.update();
    }

	// Retrieve the glide record that has been awaiting vendor for 7 business days.
    var grVendor = new GlideRecord('incident');
    grCaller.addQuery('state', '3');
    grCaller.addQuery('hold_reason', 'Awaiting Vendor');
    grCaller.addQuery('sys_updated_on', '<=', from7days);
    grCaller.query();
	while(grCaller.next()) {
		grCaller.state = '2';
		grCaller.update();
	}

})();
```
### Option 2B: Create a business rule
- Execute the script to handle cases 2 & 3 when an incident has been updated to an On-Hold incident.
```javascript
(function executeRule(current, previous /*null when async*/) {

	//CASE 3: If the reason is Awaiting Change, when the related Change is closed
	if(current.hold_reason == 'Awaiting Change' && current.rfc){
		var changeGR = new GlideRecord('change_request');
		if(changeGR.get(current.rfc)){
			current.state = '2'; // Change state to 'In Progress'
		}
	}
	//CASE 4: If the reason is Awaiting Problem, when the related Problem is closed
	if(current.hold_reason == 'Awaiting Problem' && current.problem_id){
		var problemGR = new GlideRecord('problem');
		if(problemGR.get(current.problem_id)){
			current.state = '2'; // Change state to 'In Progress'
		}		
	}
})(current, previous);
```