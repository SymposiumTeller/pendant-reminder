# Life Logs Calendar Integration

A Google Apps Script that automatically extracts calendar events and tasks from Limitless Life Logs and adds them to your Google Calendar and Google Tasks.



## Overview

This script uses AI to analyze your Life Logs transcripts, identify tasks and appointments, and automatically add them to your Google Calendar and Tasks list. It features a customizable scoring system, an approval workflow, and machine learning capabilities to improve its accuracy over time.

## Features

- 🤖 **AI-Powered Event Detection**: Uses OpenAI to analyze transcripts and extract calendar events and tasks
- ✅ **Approval Workflow**: Review and approve detected events before they're added to your calendar
- 📝 **Google Calendar & Tasks Integration**: Automatically adds appointments to Calendar and to-do items to Tasks
- 🧠 **Learning System**: Improves over time based on your feedback
- 📊 **Smart Scoring**: Sophisticated algorithm to reduce false positives
- 📱 **Email Notifications**: Get notified when new items are detected
- 🔄 **Automatic Processing**: Runs on a schedule to process new Life Logs

## How It Works

1. The script periodically checks for new Life Logs from the Limitless API
2. Each log is processed through OpenAI to extract potential events and tasks
3. A scoring system evaluates each potential event to filter out false positives
4. Detected events are presented for your approval in a Google Sheet
5. Once approved, events are added to your Google Calendar or Google Tasks
6. The system learns from your feedback to improve future suggestions

## Setup

### Prerequisites

- A Google account
- A Limitless API key
- An OpenAI API key
- Google Apps Script editor access

### Installation

1. Open [Google Apps Script](https://script.google.com/)
2. Create a new project
3. Copy the entire code into the script editor:

```
   // Configuration - Replace with your actual API keys
const LIMITLESS_API_KEY = "INSERT YOUR API CODE HERE BUCKO";
const OPENAI_API_KEY = "API BABAY"; // Get this from https://platform.openai.com/api-keys

// How many hours back to check for new logs
const HOURS_LOOKBACK = 24;

// Calendar settings
const CALENDAR_NAME = "Life Logs Appointments"; // For appointments only
const DEFAULT_REMINDER_MINUTES = 30; // Default reminder time in minutes

// Approval workflow settings
const ENABLE_APPROVAL_WORKFLOW = true; // Set to false to bypass approval process
const SPREADSHEET_ID = ""; // Leave empty to create a new one, or set to an existing spreadsheet ID
const PENDING_ITEMS_SHEET_NAME = "PendingLifeLogsItems";
const PROCESSED_LOGS_SHEET_NAME = "ProcessedLogs";

// New scoring system settings
const SCORING_HISTORY_SHEET_NAME = "ScoringHistory";
const USER_PREFERENCES_SHEET_NAME = "UserPreferences";
const DEFAULT_SCORE_THRESHOLD = 80; // Default threshold for creating a reminder
const LEARNING_ENABLED = true; // Enable learning from feedback

// Main function that runs the entire process
function processLifeLogs() {
  console.log("Starting Life Logs processing...");
    
  // Initialize scoring system
  initializeScoring();
    
  // First, process any pending approved items
  if (ENABLE_APPROVAL_WORKFLOW) {
    processPendingApprovals();
  }

  // Step 1: Get recent life logs
  const recentLogs = getRecentLifeLogs();
  if (!recentLogs || recentLogs.length === 0) {
    console.log("No recent logs found.");
    return;
  }
    
  console.log(`Found ${recentLogs.length} recent logs.`);
    
  // Step 2: Process logs with OpenAI to extract events
  for (const log of recentLogs) {
    const logContent = log.markdown || "";
    if (!logContent.trim()) {
      console.log(`Log ${log.id} has no content, skipping.`);
      continue;
    }
      
    // Skip logs we've already processed
    if (hasLogBeenProcessed(log.id)) {
      console.log(`Log ${log.id} has already been processed, skipping.`);
      continue;
    }
      
    console.log(`Processing log: "${log.title}"`);
    const events = extractEventsWithOpenAI(logContent);
      
    // Step 3: Apply scoring logic to events
    const scoredEvents = [];
    if (events && events.length > 0) {
      console.log(`Found ${events.length} potential events/tasks in log.`);
        
      // Score each event and filter by threshold
      for (const event of events) {
        const scoringResult = scoreEvent(event, logContent);
          
        // Record the score in history for learning
        recordEventScore(event, scoringResult, log.id);
          
        // Only keep events that meet the threshold
        if (scoringResult.totalScore >= getCurrentScoreThreshold()) {
          scoredEvents.push({
            ...event,
            score: scoringResult.totalScore,
            scoringFactors: scoringResult.factors
          });
        } else {
          console.log(`Event "${event.title}" scored ${scoringResult.totalScore}, below threshold of ${getCurrentScoreThreshold()}, skipping.`);
        }
      }
        
      console.log(`After scoring, ${scoredEvents.length} events/tasks met the threshold.`);
        
      // Step 4: Process events that met the threshold
      if (scoredEvents.length > 0) {
        processEvents(scoredEvents, log);
      } else {
        console.log("No events or tasks met the scoring threshold in this log.");
      }
    } else {
      console.log("No events or tasks found in this log.");
    }
      
    // Mark this log as processed regardless of whether events were found
    markLogAsProcessed(log.id);
  }
    
  // If learning is enabled, check if we need to adjust thresholds
  if (LEARNING_ENABLED) {
    adjustScoreThresholdBasedOnFeedback();
  }
    
  console.log("Life Logs processing completed.");
}
// Process events with either approval workflow or direct addition
function processEvents(events, log) {
  if (!events || events.length === 0) return;
    
  // Make sure the approval sheet has scoring columns
  if (ENABLE_APPROVAL_WORKFLOW) {
    updateApprovalSheetWithScoring();
  }
    
  if (ENABLE_APPROVAL_WORKFLOW) {
    // Store items for approval instead of directly adding them
    storeItemsForApproval(events, log);
      
    // Send approval email
    sendApprovalEmail(events, log);
      
    console.log(`${events.length} items stored for approval and notification sent.`);
  } else {
    // Original direct processing logic
    let appointmentsAdded = 0;
    let tasksAdded = 0;
      
    // Keep track of newly added events for notification
    const addedEvents = [];
      
    for (const event of events) {
      try {
        if (event.isReminder) {
          // Add to Google Tasks
          if (addToGoogleTasks(event, log)) {
            tasksAdded++;
            addedEvents.push(event);
          }
        } else {
          // Add to Google Calendar
          if (addToGoogleCalendar(event, log)) {
            appointmentsAdded++;
            addedEvents.push(event);
          }
        }
      } catch (error) {
        console.error(`Error processing event "${event.title}":`, error);
      }
    }
      
    console.log(`Added ${appointmentsAdded} appointments to calendar and ${tasksAdded} tasks to Google Tasks.`);
      
    // Send email notification if anything was added
    if (addedEvents.length > 0) {
      sendEmailNotification(addedEvents, log);
    }
  }
}

// Creates or gets the approval spreadsheet and saves its ID to properties
function getApprovalSpreadsheet() {
  // First check if we have an ID stored in script properties
  const scriptProperties = PropertiesService.getScriptProperties();
  let spreadsheetId = scriptProperties.getProperty('APPROVAL_SPREADSHEET_ID');
    
  // If a fixed ID is provided in the constants, use that
  if (SPREADSHEET_ID) {
    spreadsheetId = SPREADSHEET_ID;
  }
    
  let spreadsheet;
    
  if (spreadsheetId) {
    try {
      // Try to open the existing spreadsheet
      spreadsheet = SpreadsheetApp.openById(spreadsheetId);
      console.log(`Using existing approval spreadsheet: ${spreadsheet.getUrl()}`);
    } catch (e) {
      console.error(`Could not open existing spreadsheet with ID ${spreadsheetId}: ${e}`);
      spreadsheetId = null;
    }
  }
    
  // If no spreadsheet found, create a new one
  if (!spreadsheetId) {
    try {
      spreadsheet = SpreadsheetApp.create("Life Logs Approval Tracker");
      spreadsheetId = spreadsheet.getId();
        
      // Store the ID for future use
      scriptProperties.setProperty('APPROVAL_SPREADSHEET_ID', spreadsheetId);
      console.log(`Created new approval spreadsheet: ${spreadsheet.getUrl()}`);
        
      // Share the spreadsheet with the current user
      try {
        const userEmail = Session.getActiveUser().getEmail();
        if (userEmail) {
          DriveApp.getFileById(spreadsheetId).addEditor(userEmail);
          console.log(`Shared spreadsheet with ${userEmail}`);
        }
      } catch (e) {
        console.error(`Could not share spreadsheet: ${e}`);
      }
    } catch (e) {
      console.error(`Could not create new spreadsheet: ${e}`);
      throw new Error("Failed to create or access approval spreadsheet");
    }
  }
    
  return spreadsheet;
}
// Get or create the sheet for pending items
function getApprovalSheet() {
  // Get the spreadsheet
  const spreadsheet = getApprovalSpreadsheet();
    
  // Look for the sheet
  let sheet = spreadsheet.getSheetByName(PENDING_ITEMS_SHEET_NAME);
    
  // If sheet doesn't exist, create it
  if (!sheet) {
    sheet = spreadsheet.insertSheet(PENDING_ITEMS_SHEET_NAME);
      
    // Add headers
    sheet.appendRow([
      "ID", "Status", "Timestamp", "Title", "Date", "Time", "Type",
      "Details", "LogID", "LogTitle", "LogTimestamp"
    ]);
      
    // Format headers
    sheet.getRange(1, 1, 1, 11).setBackground("#f3f3f3").setFontWeight("bold");
    sheet.setFrozenRows(1);
      
    // Add data validation for the Status column
    const statusRange = sheet.getRange("B2:B1000");
    const rule = SpreadsheetApp.newDataValidation()
      .requireValueInList(['pending', 'approved', 'rejected', 'processing', 'processed', 'error'], true)
      .build();
    statusRange.setDataValidation(rule);
      
    // Add conditional formatting for different statuses
    const conditionalFormatRules = sheet.getConditionalFormatRules();
      
    // Pending - yellow
    const pendingRule = SpreadsheetApp.newConditionalFormatRule()
      .whenTextEqualTo("pending")
      .setBackground("#FFFDE7")
      .setRanges([sheet.getRange("B2:B1000")])
      .build();
    conditionalFormatRules.push(pendingRule);
      
    // Approved - green
    const approvedRule = SpreadsheetApp.newConditionalFormatRule()
      .whenTextEqualTo("approved")
      .setBackground("#E8F5E9")
      .setRanges([sheet.getRange("B2:B1000")])
      .build();
    conditionalFormatRules.push(approvedRule);
      
    // Rejected - red
    const rejectedRule = SpreadsheetApp.newConditionalFormatRule()
      .whenTextEqualTo("rejected")
      .setBackground("#FFEBEE")
      .setRanges([sheet.getRange("B2:B1000")])
      .build();
    conditionalFormatRules.push(rejectedRule);
      
    // Processed - blue
    const processedRule = SpreadsheetApp.newConditionalFormatRule()
      .whenTextEqualTo("processed")
      .setBackground("#E3F2FD")
      .setRanges([sheet.getRange("B2:B1000")])
      .build();
    conditionalFormatRules.push(processedRule);
      
    // Apply rules
    sheet.setConditionalFormatRules(conditionalFormatRules);
      
    // Auto-resize columns
    sheet.autoResizeColumns(1, 11);
      
    console.log(`Created and formatted ${PENDING_ITEMS_SHEET_NAME} sheet`);
  }
    
  return sheet;
}

// Get or create sheet to track processed logs
function getProcessedLogsSheet() {
  const spreadsheet = getApprovalSpreadsheet();
  let sheet = spreadsheet.getSheetByName(PROCESSED_LOGS_SHEET_NAME);
    
  if (!sheet) {
    sheet = spreadsheet.insertSheet(PROCESSED_LOGS_SHEET_NAME);
    sheet.appendRow(["LogID", "ProcessedTimestamp"]);
    sheet.setFrozenRows(1);
      
    // Format headers
    sheet.getRange(1, 1, 1, 2).setBackground("#f3f3f3").setFontWeight("bold");
      
    console.log(`Created ${PROCESSED_LOGS_SHEET_NAME} sheet to track processed logs`);
  }
    
  return sheet;
}

// Mark a log as processed
function markLogAsProcessed(logId) {
  const sheet = getProcessedLogsSheet();
  sheet.appendRow([logId, new Date().toISOString()]);
  console.log(`Marked log ${logId} as processed`);
}

// Check if a log has already been processed
function hasLogBeenProcessed(logId) {
  const sheet = getProcessedLogsSheet();
  const data = sheet.getDataRange().getValues();
    
  // Skip header row
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === logId) {
      return true;
    }
  }
    
  return false;
}
// Store items in Google Sheet for approval
function storeItemsForApproval(events, log) {
  // Get the approval sheet
  const sheet = getApprovalSheet();
    
  // Generate unique IDs for each item
  const timestamp = new Date().getTime();
  const logId = log.id || "unknown";
    
  // Add each event to the pending items sheet
  events.forEach((event, index) => {
    const itemId = `${timestamp}-${logId}-${index}`;
    const row = [
      itemId,                       // Unique ID
      "pending",                    // Status (pending, approved, rejected)
      new Date().toISOString(),     // Timestamp added
      event.title,                  // Event title
      event.date,                   // Date string
      event.time || "",             // Time string
      event.isReminder ? "task" : "event",  // Type
      event.details || "",          // Details
      log.id || "",                 // Source log ID
      log.title || "",              // Source log title
      new Date(log.startTime).toISOString() // Source log timestamp
    ];
      
    // Append the row
    sheet.appendRow(row);
      
    // Check if we have a Score column and add the score if available
    const headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    const scoreIndex = headers.indexOf("Score");
      
    if (scoreIndex !== -1 && event.score) {
      const rowIndex = sheet.getLastRow();
      sheet.getRange(rowIndex, scoreIndex + 1).setValue(event.score);
    }
  });
    
  console.log(`${events.length} items stored in pending items sheet`);
}

// Send email with link to the Google Sheet for manual approval
function sendApprovalEmail(events, log) {
  // Get the user's email
  const email = Session.getActiveUser().getEmail();
    
  // Create the email subject
  const subject = `Life Logs: ${events.length} items need your approval`;
    
  // Get the spreadsheet URL for direct access
  let spreadsheetUrl = "";
  try {
    const spreadsheet = getApprovalSpreadsheet();
    spreadsheetUrl = spreadsheet.getUrl();
  } catch (e) {
    console.error("Could not get spreadsheet URL:", e);
  }
    
  // Create the email body with HTML formatting
  let body = `<h2>Events/Tasks Detected in Your Life Logs</h2>`;
  body += `<p>The following ${events.length} items were extracted from your recent Life Log "${log.title}" and need your approval.</p>`;
    
  // Add a table of events
  body += `<table border="1" cellpadding="5" style="border-collapse: collapse;">`;
  body += `<tr style="background-color: #f2f2f2;"><th>Type</th><th>Title</th><th>Date/Time</th><th>Details</th><th>Score</th></tr>`;
    
  // Add each event to the table
  events.forEach((event) => {
    const type = event.isReminder ? "Task" : "Calendar Event";
    const bgColor = event.isReminder ? "#fff0f0" : "#f0f0ff"; // Light red for tasks, light blue for events
    const dateTimeStr = `${event.date} ${event.time || ""}`.trim();
    const score = event.score || "N/A";
      
    body += `<tr style="background-color: ${bgColor};">`;
    body += `<td><strong>${type}</strong></td>`;
    body += `<td>${event.title}</td>`;
    body += `<td>${dateTimeStr}</td>`;
    body += `<td>${event.details || ""}</td>`;
    body += `<td>${score}</td>`;
    body += `</tr>`;
  });
    
  body += `</table>`;
    
  // Add information about the source log
  body += `<p><strong>Source:</strong> "${log.title}" (${new Date(log.startTime).toLocaleString()})</p>`;
    
  // Add instructions for approval
  body += `<div style="margin-top:20px; padding:15px; background-color:#f0f8ff; border:1px solid #4285f4; border-radius:5px;">`;
  body += `<h3 style="margin-top:0;">How to approve or reject items:</h3>`;
  body += `<p>1. <a href="${spreadsheetUrl}" target="_blank">Open the Google Sheet</a> containing pending items</p>`;
  body += `<p>2. Find the items listed above in the sheet</p>`;
  body += `<p>3. To approve an item, select the 'Status' cell and choose "approved" from the dropdown</p>`;
  body += `<p>4. To reject an item, select the 'Status' cell and choose "rejected" from the dropdown</p>`;
  body += `<p>5. The script will process approved items automatically</p>`;
  body += `<p>6. Please provide feedback in the 'Feedback' column to help improve the system</p>`;
  body += `</div>`;
    
  // Send the email
  try {
    MailApp.sendEmail({
      to: email,
      subject: subject,
      htmlBody: body
    });
    console.log(`Approval email sent to ${email}`);
  } catch (error) {
    console.error("Error sending approval email:", error);
  }
}

// Function to check for and process any pending items that have been approved
function processPendingApprovals() {
  console.log("Checking for pending approvals...");
    
  // Get the pending items sheet
  let sheet;
  try {
    sheet = getApprovalSheet();
  } catch (e) {
    console.log("No pending items sheet found:", e);
    return;
  }
    
  const data = sheet.getDataRange().getValues();
  if (data.length <= 1) { // Only header row
    console.log("No pending items found");
    return;
  }
    
  const headers = data[0];
    
  // Find the column indices
  const idIndex = headers.indexOf("ID");
  const statusIndex = headers.indexOf("Status");
  const titleIndex = headers.indexOf("Title");
  const dateIndex = headers.indexOf("Date");
  const timeIndex = headers.indexOf("Time");
  const typeIndex = headers.indexOf("Type");
  const detailsIndex = headers.indexOf("Details");
  const logIdIndex = headers.indexOf("LogID");
  const logTitleIndex = headers.indexOf("LogTitle");
  const logTimestampIndex = headers.indexOf("LogTimestamp");
  const feedbackIndex = headers.indexOf("Feedback");
    
  // Make sure we found all the expected columns
  if (idIndex === -1 || statusIndex === -1 || titleIndex === -1 || dateIndex === -1 ||
      timeIndex === -1 || typeIndex === -1 || detailsIndex === -1) {
    console.error("Some expected columns not found in the sheet");
    return;
  }
    
  // Find items with "approved" status that haven't been processed yet
  let approvedItems = [];
    
  for (let i = 1; i < data.length; i++) {
    if (data[i][statusIndex] === "approved") {
      approvedItems.push({
        rowIndex: i + 1, // +1 because sheets are 1-indexed
        id: data[i][idIndex],
        title: data[i][titleIndex],
        date: data[i][dateIndex],
        time: data[i][timeIndex],
        type: data[i][typeIndex],
        details: data[i][detailsIndex],
        logId: data[i][logIdIndex],
        logTitle: data[i][logTitleIndex],
        logTimestamp: data[i][logTimestampIndex],
        feedback: feedbackIndex !== -1 ? data[i][feedbackIndex] : ""
      });
    }
  }
    
  console.log(`Found ${approvedItems.length} approved items to process`);
    
  if (approvedItems.length === 0) {
    return;
  }
    
  // Process each approved item
  for (const item of approvedItems) {
    try {
      // Mark as "processing" to avoid duplicate processing
      sheet.getRange(item.rowIndex, statusIndex + 1).setValue("processing");
        
      // Create event object
      const event = {
        title: item.title,
        date: item.date,
        time: item.time,
        isReminder: item.type === "task",
        details: item.details
      };
        
      // Create log object for context
      const log = {
        id: item.logId,
        title: item.logTitle,
        startTime: item.logTimestamp
      };
        
      // Process the item based on type
      let result = false;
        
      if (event.isReminder) {
        // Add to Google Tasks
        result = addToGoogleTasks(event, log);
      } else {
        // Add to Google Calendar
        result = addToGoogleCalendar(event, log);
      }
        
      // Process feedback if learning is enabled
      if (LEARNING_ENABLED && item.feedback) {
        processFeedbackForLearning(item.id, item.feedback);
      }
        
      // Mark as "processed" or "error" based on result
      if (result) {
        sheet.getRange(item.rowIndex, statusIndex + 1).setValue("processed");
        console.log(`Successfully processed item: "${item.title}"`);
      } else {
        sheet.getRange(item.rowIndex, statusIndex + 1).setValue("error");
        console.log(`Failed to process item: "${item.title}"`);
      }
    } catch (error) {
      console.error(`Error processing approved item ${item.id}:`, error);
      sheet.getRange(item.rowIndex, statusIndex + 1).setValue("error");
    }
  }
}
// Add an appointment to Google Calendar
function addToGoogleCalendar(event, log) {
  try {
    // Parse the date and time
    const eventDateTime = parseDateTime(event.date, event.time);
    if (!eventDateTime) {
      console.log(`Could not parse date/time for event: ${event.title}`);
      return false;
    }
      
    // Get or create the calendar
    const calendar = getOrCreateCalendar(CALENDAR_NAME);
      
    // Create event details
    const title = event.title;
    const details = `${event.details || ''}\n\nExtracted from Life Log: "${log.title}"\nLog date: ${new Date(log.startTime).toLocaleString()}`;
      
    // Check if this event already exists to avoid duplicates
    if (eventAlreadyExists(calendar, title, eventDateTime)) {
      console.log(`Event "${title}" already exists in calendar, skipping.`);
      return false;
    }
      
    // Debug logging of dates
    console.log(`Creating calendar event: ${title} on ${eventDateTime.start.toLocaleString()} to ${eventDateTime.end.toLocaleString()}`);
      
    // Create the calendar event
    const calendarEvent = calendar.createEvent(
      title,
      eventDateTime.start,
      eventDateTime.end,
      {
        description: details
      }
    );
      
    // Add a reminder
    calendarEvent.addPopupReminder(DEFAULT_REMINDER_MINUTES);
      
    console.log(`Added appointment: "${title}" on ${eventDateTime.start.toLocaleString()}`);
    return true;
  } catch (error) {
    console.error(`Error adding appointment "${event.title}" to calendar:`, error);
    return false;
  }
}

// Add a reminder to Google Tasks
function addToGoogleTasks(event, log) {
  try {
    // Get the default task list
    const taskLists = Tasks.Tasklists.list().items;
    if (!taskLists || taskLists.length === 0) {
      console.error("No task lists found");
      return false;
    }
      
    const taskList = taskLists[0]; // Use the first (default) task list
      
    // Parse the date for the due date
    const dueDate = parseDueDate(event.date, event.time);
    if (!dueDate) {
      console.log(`Could not parse due date for task: ${event.title}`);
      return false;
    }
      
    // Format to RFC 3339 format which Google Tasks API requires
    const dueDateRFC3339 = dueDate.toISOString();
      
    // Create task details
    const title = event.title;
    const notes = `${event.details || ''}\n\nExtracted from Life Log: "${log.title}"\nLog date: ${new Date(log.startTime).toLocaleString()}`;
      
    // Modified task duplication check to be less strict
    if (isExactTaskDuplicate(taskList.id, title, dueDateRFC3339)) {
      console.log(`Task "${title}" already exists, skipping.`);
      return false;
    }
      
    // Add a timestamp to make the task unique
    const taskTitle = "⚠️ " + title + " [" + new Date().toLocaleTimeString() + "]";
      
    // Create the task with high priority
    const task = {
      title: taskTitle,
      notes: notes,
      due: dueDateRFC3339,
      status: 'needsAction'
    };
      
    Tasks.Tasks.insert(task, taskList.id);
      
    console.log(`Added high-priority task: "${title}" due on ${dueDate.toLocaleString()}`);
    return true;
  } catch (error) {
    console.error(`Error adding task "${event.title}" to Google Tasks:`, error);
    return false;
  }
}

// Check for exact task duplicates only
function isExactTaskDuplicate(taskListId, title, dueDate) {
  try {
    // Only check today's tasks for duplicates to avoid blocking repeating tasks on different days
    const today = new Date();
    const todayStr = today.toISOString().split('T')[0];
    const dueDateStr = dueDate.split('T')[0];
      
    // If the due date is not today, don't consider it a duplicate
    if (dueDateStr !== todayStr) {
      return false;
    }
      
    // Get active tasks only
    const activeTasks = Tasks.Tasks.list(taskListId, {
      showCompleted: false
    }).items || [];
      
    // Check for an exact title match only (case insensitive)
    for (const task of activeTasks) {
      // Remove the warning emoji and timestamp if present
      const cleanTaskTitle = task.title.replace(/^⚠️\s*/, '').replace(/\s*\[\d{1,2}:\d{2}:\d{2}.*\]$/, '');
        
      // Only consider it a duplicate if the titles match exactly (ignoring case)
      if (cleanTaskTitle.toLowerCase() === title.toLowerCase()) {
        console.log(`Exact task match "${title}" already exists: "${task.title}"`);
        return true;
      }
    }
      
    return false;
  } catch (error) {
    console.error("Error checking existing tasks:", error);
    return false; // If we can't check, assume it doesn't exist
  }
}

// Get an existing calendar or create a new one with the given name
function getOrCreateCalendar(calendarName) {
  // Try to find an existing calendar with this name
  const calendars = CalendarApp.getAllCalendars();
  for (const cal of calendars) {
    if (cal.getName() === calendarName) {
      return cal;
    }
  }
    
  // If not found, create a new calendar
  const newCalendar = CalendarApp.createCalendar(calendarName);
  console.log(`Created new calendar: ${calendarName}`);
  return newCalendar;
}

// Check if an event with similar details already exists
function eventAlreadyExists(calendar, title, dateTime) {
  const existingEvents = calendar.getEvents(
    new Date(dateTime.start.getTime() - 60 * 60 * 1000), // Look 1 hour before
    new Date(dateTime.start.getTime() + 60 * 60 * 1000)  // Look 1 hour after
  );
    
  for (const event of existingEvents) {
    if (event.getTitle().toLowerCase().includes(title.toLowerCase())) {
      return true;
    }
  }
    
  return false;
}

// Check if a task already exists (including completed tasks)
function taskAlreadyExists(taskListId, title, dueDate) {
  try {
    // Format the due date to match Google Tasks format (removing milliseconds)
    const formattedDueDate = dueDate.split('.')[0] + 'Z';
      
    // Get all incomplete tasks in the list
    const activeTasks = Tasks.Tasks.list(taskListId, {
      showCompleted: false
    }).items || [];
      
    // Get all completed tasks in the list (from the last 7 days)
    const completedTasks = Tasks.Tasks.list(taskListId, {
      showCompleted: true,
      showHidden: true,
      completedMin: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString()
    }).items || [];
      
    // Combine active and completed tasks
    const allTasks = [...activeTasks, ...completedTasks];
      
    // Check for a task with similar title and due date
    for (const task of allTasks) {
      // Try different matching strategies
        
      // 1. Direct title matching (case insensitive)
      const titleMatch = task.title.toLowerCase().includes(title.toLowerCase()) ||
                       title.toLowerCase().includes(task.title.toLowerCase());
        
      // 2. Remove emoji and other markers for comparison
      const cleanTaskTitle = task.title.replace(/^⚠️\s*/, '');
      const cleanMatch = cleanTaskTitle.toLowerCase().includes(title.toLowerCase()) ||
                        title.toLowerCase().includes(cleanTaskTitle.toLowerCase());
        
      // Check if we have a title match and similar due date
      if ((titleMatch || cleanMatch) &&
          (!task.due || task.due.startsWith(formattedDueDate.substring(0, 10)))) {
        console.log(`Task "${title}" already exists (${task.status}): "${task.title}"`);
        return true;
      }
    }
      
    return false;
  } catch (error) {
    console.error("Error checking existing tasks:", error);
    return false; // If we can't check, assume it doesn't exist
  }
}
// Helper function to parse date and time strings into Date objects for calendar events
function parseDateTime(dateStr, timeStr) {
  try {
    const now = new Date();
    let eventDate;
      
    // Parse the date
    if (dateStr.toLowerCase() === 'today') {
      eventDate = new Date(now);
    } else if (dateStr.toLowerCase() === 'tomorrow') {
      eventDate = new Date(now);
      eventDate.setDate(eventDate.getDate() + 1);
    } else if (dateStr.match(/^\d{4}-\d{2}-\d{2}$/)) {
      // ISO format YYYY-MM-DD
      eventDate = new Date(dateStr + 'T00:00:00');
    } else {
      // Try to parse other formats
      const dateParts = dateStr.split(/[\/\-\.]/);
      if (dateParts.length === 3) {
        // Handle various date formats
        let year, month, day;
          
        // Check if first part looks like a year
        if (dateParts[0].length === 4 && !isNaN(dateParts[0])) {
          // YYYY-MM-DD format
          year = parseInt(dateParts[0]);
          month = parseInt(dateParts[1]) - 1; // Month is 0-based in JS
          day = parseInt(dateParts[2]);
        } else {
          // MM/DD/YYYY format (or similar)
          month = parseInt(dateParts[0]) - 1;
          day = parseInt(dateParts[1]);
            
          // Year might be 2 or 4 digits
          year = parseInt(dateParts[2]);
          if (year < 100) {
            year += 2000; // Assume 20xx for 2-digit years
          }
        }
          
        eventDate = new Date(year, month, day);
      } else {
        // Try natural language parsing
        const monthNames = ["january", "february", "march", "april", "may", "june", "july", "august", "september", "october", "november", "december"];
        const monthAbbr = ["jan", "feb", "mar", "apr", "may", "jun", "jul", "aug", "sep", "oct", "nov", "dec"];
          
        // Check for formats like "14th November" or "November 14th"
        let monthIndex = -1;
        let day = -1;
          
        // Look for month name
        const lowerDateStr = dateStr.toLowerCase();
        for (let i = 0; i < monthNames.length; i++) {
          if (lowerDateStr.includes(monthNames[i]) || lowerDateStr.includes(monthAbbr[i])) {
            monthIndex = i;
            break;
          }
        }
          
        // Look for day number
        const dayMatch = lowerDateStr.match(/\b(\d{1,2})(st|nd|rd|th)?\b/);
        if (dayMatch) {
          day = parseInt(dayMatch[1]);
        }
          
        if (monthIndex >= 0 && day > 0) {
          // We found both month and day
          const year = now.getFullYear(); // Assume current year
          eventDate = new Date(year, monthIndex, day);
            
          // If the date is in the past by more than a few days, assume it's for next year
          if (eventDate < now && now - eventDate > 7 * 24 * 60 * 60 * 1000) {
            eventDate.setFullYear(year + 1);
          }
        } else {
          console.log(`Couldn't parse date: ${dateStr}`);
          return null;
        }
      }
    }
      
    console.log(`Parsed date: ${eventDate.toISOString()} from "${dateStr}"`);
      
    // Parse the time
    if (timeStr && timeStr.trim()) {
      const time = timeStr.trim();
      let hours = 0;
      let minutes = 0;
        
      const timeParts = time.match(/(\d{1,2}):(\d{2})\s*(am|pm)?/i);
      if (timeParts) {
        hours = parseInt(timeParts[1]);
        minutes = parseInt(timeParts[2]);
          
        // Handle AM/PM
        if (timeParts[3] && timeParts[3].toLowerCase() === 'pm' && hours < 12) {
          hours += 12;
        } else if (timeParts[3] && timeParts[3].toLowerCase() === 'am' && hours === 12) {
          hours = 0;
        }
      } else {
        // Try other formats like "3pm"
        const simpleTime = time.match(/(\d{1,2})\s*(am|pm)/i);
        if (simpleTime) {
          hours = parseInt(simpleTime[1]);
            
          // Handle AM/PM
          if (simpleTime[2] && simpleTime[2].toLowerCase() === 'pm' && hours < 12) {
            hours += 12;
          } else if (simpleTime[2] && simpleTime[2].toLowerCase() === 'am' && hours === 12) {
            hours = 0;
          }
        }
      }
        
      eventDate.setHours(hours, minutes, 0, 0);
      console.log(`Added time: ${hours}:${minutes} from "${timeStr}"`);
    } else {
      // Default to 7:00 AM for all-day events as requested
      eventDate.setHours(7, 0, 0, 0);
      console.log(`No time specified, defaulting to 7:00 AM`);
    }
      
    // Create end time (1 hour later for timed events, end of day for all-day)
    const endDate = new Date(eventDate);
    if (timeStr && timeStr.trim()) {
      endDate.setHours(endDate.getHours() + 1);
    } else {
      // For events without specified time, end time is 1 hour after start (8 AM)
      endDate.setHours(8, 0, 0, 0);
    }
      
    return {
      start: eventDate,
      end: endDate
    };
  } catch (error) {
    console.error(`Error parsing date "${dateStr}" and time "${timeStr}":`, error);
    return null;
  }
}

// Parse due date for Google Tasks (simpler than calendar events)
function parseDueDate(dateStr, timeStr) {
  try {
    const now = new Date();
    let dueDate;
      
    // Parse the date
    if (dateStr.toLowerCase() === 'today') {
      // For "today" items, set to the current time
      dueDate = new Date(now);
    } else if (dateStr.toLowerCase() === 'tomorrow') {
      dueDate = new Date(now);
      dueDate.setDate(dueDate.getDate() + 1);
    } else if (dateStr.match(/^\d{4}-\d{2}-\d{2}$/)) {
      // ISO format YYYY-MM-DD
      dueDate = new Date(dateStr + 'T00:00:00');
    } else {
      // Use the same logic as parseDateTime but simpler
      const dateParts = dateStr.split(/[\/\-\.]/);
      if (dateParts.length === 3) {
        let year, month, day;
          
        if (dateParts[0].length === 4 && !isNaN(dateParts[0])) {
          year = parseInt(dateParts[0]);
          month = parseInt(dateParts[1]) - 1;
          day = parseInt(dateParts[2]);
        } else {
          month = parseInt(dateParts[0]) - 1;
          day = parseInt(dateParts[1]);
          year = parseInt(dateParts[2]);
          if (year < 100) year += 2000;
        }
          
        dueDate = new Date(year, month, day);
      } else {
        // Intelligent parsing of date strings
        const monthNames = ["january", "february", "march", "april", "may", "june", "july", "august", "september", "october", "november", "december"];
        const monthAbbr = ["jan", "feb", "mar", "apr", "may", "jun", "jul", "aug", "sep", "oct", "nov", "dec"];
          
        let monthIndex = -1;
        let day = -1;
          
        const lowerDateStr = dateStr.toLowerCase();
          
        // Special case for "next x" expressions
        if (lowerDateStr.includes("next")) {
          const dayOfWeek = ["sunday", "monday", "tuesday", "wednesday", "thursday", "friday", "saturday"];
          const dayAbbr = ["sun", "mon", "tue", "wed", "thu", "fri", "sat"];
            
          for (let i = 0; i < dayOfWeek.length; i++) {
            if (lowerDateStr.includes(dayOfWeek[i]) || lowerDateStr.includes(dayAbbr[i])) {
              // Get the next occurrence of that day
              dueDate = new Date(now);
              const currentDay = dueDate.getDay();
              const daysToAdd = (i + 7 - currentDay) % 7;
              dueDate.setDate(dueDate.getDate() + (daysToAdd === 0 ? 7 : daysToAdd));
              break;
            }
          }
            
          // Handle "next week", "next month", etc.
          if (!dueDate) {
            dueDate = new Date(now);
            if (lowerDateStr.includes("week")) {
              dueDate.setDate(dueDate.getDate() + 7);
            } else if (lowerDateStr.includes("month")) {
              dueDate.setMonth(dueDate.getMonth() + 1);
            } else if (lowerDateStr.includes("year")) {
              dueDate.setFullYear(dueDate.getFullYear() + 1);
            }
          }
        } else {
          // Look for month name
          for (let i = 0; i < monthNames.length; i++) {
            if (lowerDateStr.includes(monthNames[i]) || lowerDateStr.includes(monthAbbr[i])) {
              monthIndex = i;
              break;
            }
          }
            
          // Look for day number
          const dayMatch = lowerDateStr.match(/\b(\d{1,2})(st|nd|rd|th)?\b/);
          if (dayMatch) {
            day = parseInt(dayMatch[1]);
          }
            
          if (monthIndex >= 0 && day > 0) {
            const year = now.getFullYear();
            dueDate = new Date(year, monthIndex, day);
              
            // If the date is in the past, assume it's for next year
            if (dueDate < now) {
              dueDate.setFullYear(year + 1);
            }
          }
        }
      }
    }
      
    // If we failed to parse a date, default to today
    if (!dueDate) {
      console.log(`Failed to parse date "${dateStr}", defaulting to today`);
      dueDate = new Date(now);
    }
      
    // Add time if specified (important for notifications)
    if (timeStr && timeStr.trim()) {
      const time = timeStr.trim();
      let hours = 0;
      let minutes = 0;
        
      const timeParts = time.match(/(\d{1,2}):(\d{2})\s*(am|pm)?/i);
      if (timeParts) {
        hours = parseInt(timeParts[1]);
        minutes = parseInt(timeParts[2]);
          
        if (timeParts[3] && timeParts[3].toLowerCase() === 'pm' && hours < 12) {
          hours += 12;
        } else if (timeParts[3] && timeParts[3].toLowerCase() === 'am' && hours === 12) {
          hours = 0;
        }
      } else {
        const simpleTime = time.match(/(\d{1,2})\s*(am|pm)/i);
        if (simpleTime) {
          hours = parseInt(simpleTime[1]);
            
          if (simpleTime[2] && simpleTime[2].toLowerCase() === 'pm' && hours < 12) {
            hours += 12;
          } else if (simpleTime[2] && simpleTime[2].toLowerCase() === 'am' && hours === 12) {
            hours = 0;
          }
        }
      }
        
      dueDate.setHours(hours, minutes, 0, 0);
    } else if (dateStr.toLowerCase() === 'today') {
      // For "today" tasks WITHOUT a specified time, keep the current time
      // (we don't modify the time)
    } else {
      // For non-today tasks without specific time, set to 9am
      dueDate.setHours(9, 0, 0, 0);
    }
      
    console.log(`Parsed task due date: ${dueDate.toISOString()} from "${dateStr}" and "${timeStr}"`);
    return dueDate;
  } catch (error) {
    console.error(`Error parsing due date: ${error}`);
    return new Date(); // Default to today
  }
}
// Send an email notification for new events and tasks (when not using approval workflow)
function sendEmailNotification(events, log) {
  if (!events || events.length === 0) return;
    
  // Get the user's email (the person who owns the script)
  const email = Session.getActiveUser().getEmail();
    
  // Create the email subject
  const subject = `New Life Logs Events/Tasks Added: ${events.length} items`;
    
  // Create the email body with HTML formatting
  let body = `<h2>New Events/Tasks from Life Logs</h2>`;
  body += `<p>The following ${events.length} items were automatically added from your recent Life Logs:</p>`;
    
  // Add a table of events
  body += `<table border="1" cellpadding="5" style="border-collapse: collapse;">`;
  body += `<tr style="background-color: #f2f2f2;"><th>Type</th><th>Title</th><th>Date</th><th>Time</th><th>Details</th><th>Score</th></tr>`;
    
  // Add each event to the table
  for (const event of events) {
    const type = event.isReminder ? "Task" : "Calendar Event";
    const bgColor = event.isReminder ? "#fff0f0" : "#f0f0ff"; // Light red for tasks, light blue for events
    const score = event.score || "N/A";
      
    body += `<tr style="background-color: ${bgColor};">`;
    body += `<td><strong>${type}</strong></td>`;
    body += `<td>${event.title}</td>`;
    body += `<td>${event.date}</td>`;
    body += `<td>${event.time || "No time specified"}</td>`;
    body += `<td>${event.details || ""}</td>`;
    body += `<td>${score}</td>`;
    body += `</tr>`;
  }
    
  body += `</table>`;
    
  // Add information about the source log
  body += `<p><strong>Source:</strong> "${log.title}" (${new Date(log.startTime).toLocaleString()})</p>`;
    
  // Add a footer with links to view tasks/calendar
  body += `<p style="margin-top: 20px;">`;
  body += `<a href="https://calendar.google.com/" style="margin-right: 15px;">View Calendar</a> | `;
  body += `<a href="https://tasks.google.com/">View Tasks</a>`;
  body += `</p>`;
    
  // Send the email
  try {
    MailApp.sendEmail({
      to: email,
      subject: subject,
      htmlBody: body
    });
    console.log(`Email notification sent to ${email}`);
  } catch (error) {
    console.error("Error sending email notification:", error);
  }
}

// Initialize the scoring system
function initializeScoring() {
  // Get or create the user preferences sheet
  const sheet = getUserPreferencesSheet();
  const data = sheet.getDataRange().getValues();
    
  // Check if we already have preferences set up
  if (data.length <= 1) {
    // No preferences yet, set up defaults
    console.log("Setting up default scoring preferences...");
    sheet.appendRow(["scoreThreshold", DEFAULT_SCORE_THRESHOLD.toString()]);
    sheet.appendRow(["weightActionVerbs", "15"]);
    sheet.appendRow(["weightIntentPhrases", "15"]);
    sheet.appendRow(["weightTemporalIndicators", "15"]);
    sheet.appendRow(["weightPriorityMarkers", "15"]);
    sheet.appendRow(["weightDirectCommand", "20"]);
    sheet.appendRow(["weightSpecificDetails", "10"]);
    sheet.appendRow(["weightRepetition", "15"]);
    sheet.appendRow(["weightVoiceEmphasis", "10"]);
    sheet.appendRow(["weightAttentionPhrase", "25"]);
    sheet.appendRow(["weightUncertaintyMarkers", "-15"]);
    sheet.appendRow(["weightCasualConversation", "-20"]);
    sheet.appendRow(["learningRate", "0.05"]);
  }
}

// Get scoring history sheet
function getScoringHistorySheet() {
  const spreadsheet = getApprovalSpreadsheet();
  let sheet = spreadsheet.getSheetByName(SCORING_HISTORY_SHEET_NAME);
    
  if (!sheet) {
    sheet = spreadsheet.insertSheet(SCORING_HISTORY_SHEET_NAME);
    sheet.appendRow([
      "Timestamp",   
      "LogID",   
      "EventTitle",   
      "EventDate",   
      "EventType",   
      "TotalScore",   
      "ActionVerbs",   
      "IntentPhrases",   
      "TemporalIndicators",   
      "PriorityMarkers",   
      "DirectCommand",   
      "SpecificDetails",   
      "Repetition",   
      "VoiceEmphasis",   
      "AttentionPhrase",   
      "UncertaintyMarkers",   
      "CasualConversation",   
      "PassedThreshold",   
      "UserFeedback"
    ]);
      
    // Format headers
    sheet.getRange(1, 1, 1, 19).setBackground("#f3f3f3").setFontWeight("bold");
    sheet.setFrozenRows(1);
      
    console.log(`Created ${SCORING_HISTORY_SHEET_NAME} sheet for scoring system`);
  }
    
  return sheet;
}

// Get user preferences sheet
function getUserPreferencesSheet() {
  const spreadsheet = getApprovalSpreadsheet();
  let sheet = spreadsheet.getSheetByName(USER_PREFERENCES_SHEET_NAME);
    
  if (!sheet) {
    sheet = spreadsheet.insertSheet(USER_PREFERENCES_SHEET_NAME);
    sheet.appendRow(["Preference", "Value"]);
      
    // Format headers
    sheet.getRange(1, 1, 1, 2).setBackground("#f3f3f3").setFontWeight("bold");
    sheet.setFrozenRows(1);
      
    console.log(`Created ${USER_PREFERENCES_SHEET_NAME} sheet for user preferences`);
  }
    
  return sheet;
}

// Get a preference value from the user preferences sheet
function getPreference(preferenceName, defaultValue) {
  const sheet = getUserPreferencesSheet();
  const data = sheet.getDataRange().getValues();
    
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === preferenceName) {
      return data[i][1];
    }
  }
    
  return defaultValue;
}

// Get current score threshold
function getCurrentScoreThreshold() {
  const thresholdStr = getPreference("scoreThreshold", DEFAULT_SCORE_THRESHOLD.toString());
  return parseInt(thresholdStr);
}

// Get weight for a specific scoring factor
function getWeightForFactor(factorName, defaultWeight) {
  const weightStr = getPreference(`weight${factorName}`, defaultWeight.toString());
  return parseInt(weightStr);
}
// Record an event score in the history
function recordEventScore(event, scoringResult, logId) {
  const sheet = getScoringHistorySheet();
    
  const row = [
    new Date().toISOString(),
    logId,
    event.title,
    event.date,
    event.isReminder ? "Task" : "Event",
    scoringResult.totalScore,
    scoringResult.factors.actionVerbs ? "true" : "false",
    scoringResult.factors.intentPhrases ? "true" : "false",
    scoringResult.factors.temporalIndicators ? "true" : "false",
    scoringResult.factors.priorityMarkers ? "true" : "false",
    scoringResult.factors.directCommand ? "true" : "false",
    scoringResult.factors.specificDetails ? "true" : "false",
    scoringResult.factors.repetition ? "true" : "false",
    scoringResult.factors.voiceEmphasis ? "true" : "false",
    scoringResult.factors.attentionPhrase ? "true" : "false",
    scoringResult.factors.uncertaintyMarkers ? "true" : "false",
    scoringResult.factors.casualConversation ? "true" : "false",
    scoringResult.totalScore >= getCurrentScoreThreshold() ? "true" : "false",
    "" // User feedback will be filled in later
  ];
    
  sheet.appendRow(row);
}

// Score an event based on the TASK DETECTION FRAMEWORK
function scoreEvent(event, logContent) {
  let score = 40; // Base score
  const factors = {
    actionVerbs: false,
    intentPhrases: false,
    temporalIndicators: false,
    priorityMarkers: false,
    directCommand: false,
    specificDetails: false,
    repetition: false,
    voiceEmphasis: false,
    attentionPhrase: false,
    uncertaintyMarkers: false,
    casualConversation: false
  };
    
  // Convert the event title and details to lowercase for easier checking
  const title = (event.title || "").toLowerCase();
  const details = (event.details || "").toLowerCase();
  const combinedText = `${title} ${details}`.toLowerCase();
    
  // 1. ACTION VERBS: Check for explicit task verbs
  const actionVerbs = ["buy", "get", "pick up", "call", "schedule", "remember", "send", "pay", "make",
                      "finish", "complete", "attend", "bring", "return", "download", "upload", "submit", "renew"];
  for (const verb of actionVerbs) {
    if (combinedText.includes(verb)) {
      factors.actionVerbs = true;
      score += getWeightForFactor("ActionVerbs", 15);
      break;
    }
  }
    
  // 2. INTENT PHRASES: Check for clear intent markers
  const intentPhrases = ["i need to", "i should", "i have to", "remember to", "don't forget to",
                        "make sure to", "it's important to", "reminder to"];
  for (const phrase of intentPhrases) {
    if (combinedText.includes(phrase)) {
      factors.intentPhrases = true;
      score += getWeightForFactor("IntentPhrases", 15);
      break;
    }
  }
    
  // 3. TEMPORAL INDICATORS: Check for time references
  const temporalIndicators = ["today", "tomorrow", "tonight", "this weekend", "next week", "on monday",
                             "at 3pm", "before dinner", "after work", "by eod"];
  for (const indicator of temporalIndicators) {
    if (combinedText.includes(indicator)) {
      factors.temporalIndicators = true;
      score += getWeightForFactor("TemporalIndicators", 15);
      break;
    }
  }
    
  // 4. PRIORITY MARKERS: Check for urgency indicators
  const priorityMarkers = ["important", "urgent", "critical", "asap", "immediately", "soon", "deadline", "priority"];
  for (const marker of priorityMarkers) {
    if (combinedText.includes(marker)) {
      factors.priorityMarkers = true;
      score += getWeightForFactor("PriorityMarkers", 15);
      break;
    }
  }
    
  // 5. DIRECT COMMAND: Check for clear, direct commands
  if (title.match(/^(get|buy|call|pick|bring|make|send|pay|attend)/i)) {
    factors.directCommand = true;
    score += getWeightForFactor("DirectCommand", 20);
  }
    
  // 6. SPECIFIC DETAILS: Check for specific details
  if (combinedText.match(/(from|at|in) [a-z0-9\s]+ (store|shop|market|mall|location|place)/i) ||
      combinedText.match(/\d+ (gallon|oz|ounce|pound|liter|ml|gram)/i)) {
    factors.specificDetails = true;
    score += getWeightForFactor("SpecificDetails", 10);
  }
    
  // 7. REPETITION: Check for repetition of key phrases
  const repeatedActions = [];
  for (const verb of actionVerbs) {
    if ((title.match(new RegExp(verb, "g")) || []).length > 1 ||
        (combinedText.match(new RegExp(verb, "g")) || []).length > 1) {
      repeatedActions.push(verb);
    }
  }
    
  if (repeatedActions.length > 0) {
    factors.repetition = true;
    score += getWeightForFactor("Repetition", 15);
  }
    
  // 8. VOICE STRESS/EMPHASIS: This is hard to detect from text, but we can look for ALL CAPS or exclamation marks
  if (event.title.match(/[A-Z]{2,}/) || event.title.includes("!") ||
      (event.details && (event.details.match(/[A-Z]{2,}/) || event.details.includes("!")))) {
    factors.voiceEmphasis = true;
    score += getWeightForFactor("VoiceEmphasis", 10);
  }
    
  // 9. ATTENTION PHRASE: Check for attention phrases
  const attentionPhrases = ["hey", "listen", "note that", "remember that", "by the way", "don't forget that"];
  for (const phrase of attentionPhrases) {
    if (combinedText.includes(phrase)) {
      factors.attentionPhrase = true;
      score += getWeightForFactor("AttentionPhrase", 25);
      break;
    }
  }
    
  // 10. UNCERTAINTY MARKERS: Check for uncertainty markers
  const uncertaintyMarkers = ["maybe", "might", "possibly", "perhaps", "probably", "i guess", "i think"];
  for (const marker of uncertaintyMarkers) {
    if (combinedText.includes(marker)) {
      factors.uncertaintyMarkers = true;
      score += getWeightForFactor("UncertaintyMarkers", -15);
      break;
    }
  }
    
  // 11. CASUAL CONVERSATION: Check for signs of casual conversation
  const casualConversationMarkers = ["just saying", "you know", "by the way", "anyway", "as i was saying"];
  for (const marker of casualConversationMarkers) {
    if (combinedText.includes(marker)) {
      factors.casualConversation = true;
      score += getWeightForFactor("CasualConversation", -20);
      break;
    }
  }
    
  // Check for false positives using additional filtering
  if (combinedText.includes("i need to say") ||
      combinedText.includes("i have to admit") ||
      combinedText.includes("i should mention")) {
    // These are rhetorical phrases, not actual tasks
    score -= 30;
  }
    
  console.log(`Scored event "${event.title}": ${score} points`);
    
  return {
    totalScore: score,
    factors: factors
  };
}

// Update the approval sheet to include scoring information
function updateApprovalSheetWithScoring() {
  const sheet = getApprovalSheet();
  const headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
    
  // Check if we already have a Score column
  let scoreColumnIndex = headers.indexOf("Score");
  if (scoreColumnIndex === -1) {
    // Add Score column
    sheet.insertColumnAfter(sheet.getLastColumn());
    sheet.getRange(1, sheet.getLastColumn()).setValue("Score");
      
    // Add conditional formatting for score
    const scoreRange = sheet.getRange(2, sheet.getLastColumn(), sheet.getMaxRows(), 1);
    const rules = sheet.getConditionalFormatRules();
      
    // High score - green
    const highScoreRule = SpreadsheetApp.newConditionalFormatRule()
      .whenNumberGreaterThanOrEqualTo(90)
      .setBackground("#E6FFE6")
      .setRanges([scoreRange])
      .build();
    rules.push(highScoreRule);
      
    // Medium score - yellow
    const mediumScoreRule = SpreadsheetApp.newConditionalFormatRule()
      .whenNumberBetween(70, 90)
      .setBackground("#FFFDE7")
      .setRanges([scoreRange])
      .build();
    rules.push(mediumScoreRule);
      
    // Low score - light red
    const lowScoreRule = SpreadsheetApp.newConditionalFormatRule()
      .whenNumberLessThan(70)
      .setBackground("#FFEBEE")
      .setRanges([scoreRange])
      .build();
    rules.push(lowScoreRule);
      
    sheet.setConditionalFormatRules(rules);
      
    console.log("Added Score column to approval sheet");
  }
    
  // Also check for Feedback column
  let feedbackColumnIndex = headers.indexOf("Feedback");
  if (feedbackColumnIndex === -1) {
    // Add Feedback column
    sheet.insertColumnAfter(sheet.getLastColumn());
    sheet.getRange(1, sheet.getLastColumn()).setValue("Feedback");
      
    // Add data validation for feedback
    const feedbackRange = sheet.getRange(2, sheet.getLastColumn(), sheet.getMaxRows(), 1);
    const rule = SpreadsheetApp.newDataValidation()
      .requireValueInList(['correct', 'should_not_have_included', 'missed_important', ''], true)
      .build();
    feedbackRange.setDataValidation(rule);
      
    console.log("Added Feedback column to approval sheet");
  }
}

// Process user feedback to adjust learning
function processFeedbackForLearning(itemId, feedback) {
  if (!LEARNING_ENABLED || !feedback || feedback === '') {
    return;
  }
    
  console.log(`Processing feedback "${feedback}" for item ${itemId}`);
    
  // Get the scoring history sheet
  const scoringSheet = getScoringHistorySheet();
  const scoringData = scoringSheet.getDataRange().getValues();
  const scoringHeaders = scoringData[0];
    
  // Find the corresponding entry in the scoring history
  const titleIndex = scoringHeaders.indexOf("EventTitle");
  const scoreIndex = scoringHeaders.indexOf("TotalScore");
  const feedbackIndex = scoringHeaders.indexOf("UserFeedback");
    
  // Get the approval sheet to find the item
  const approvalSheet = getApprovalSheet();
  const approvalData = approvalSheet.getDataRange().getValues();
  const approvalHeaders = approvalData[0];
    
  const idIndex = approvalHeaders.indexOf("ID");
  const eventTitleIndex = approvalHeaders.indexOf("Title");
    
  // Find the item in the approval sheet
  let itemTitle = null;
  for (let i = 1; i < approvalData.length; i++) {
    if (approvalData[i][idIndex] === itemId) {
      itemTitle = approvalData[i][eventTitleIndex];
      break;
    }
  }
    
  if (!itemTitle) {
    console.log(`Could not find item ${itemId} in approval sheet`);
    return;
  }
    
  // Find the item in the scoring history
  for (let i = 1; i < scoringData.length; i++) {
    if (scoringData[i][titleIndex] === itemTitle) {
      // Update the feedback
      scoringSheet.getRange(i + 1, feedbackIndex + 1).setValue(feedback);
      console.log(`Updated feedback for "${itemTitle}" in scoring history`);
      return;
    }
  }
    
  console.log(`Could not find item "${itemTitle}" in scoring history`);
}
// Adjust score threshold based on feedback
function adjustScoreThresholdBasedOnFeedback() {
  if (!LEARNING_ENABLED) {
    return;
  }
    
  console.log("Analyzing feedback to adjust score threshold...");
    
  // Get the scoring history
  const sheet = getScoringHistorySheet();
  const data = sheet.getDataRange().getValues();
    
  if (data.length <= 1) {
    console.log("No scoring history yet, skipping threshold adjustment");
    return;
  }
    
  const headers = data[0];
  const scoreIndex = headers.indexOf("TotalScore");
  const thresholdIndex = headers.indexOf("PassedThreshold");
  const feedbackIndex = headers.indexOf("UserFeedback");
    
  // Count false positives (should not have included) and false negatives (missed important)
  let falsePositives = 0;
  let falseNegatives = 0;
  let totalWithFeedback = 0;
    
  for (let i = 1; i < data.length; i++) {
    const feedback = data[i][feedbackIndex];
    if (feedback) {
      totalWithFeedback++;
        
      const passedThreshold = data[i][thresholdIndex] === "true";
      const score = parseFloat(data[i][scoreIndex]);
        
      if (passedThreshold && feedback === "should_not_have_included") {
        falsePositives++;
      } else if (!passedThreshold && feedback === "missed_important") {
        falseNegatives++;
      }
    }
  }
    
  // Only adjust if we have enough feedback
  if (totalWithFeedback < 5) {
    console.log(`Only ${totalWithFeedback} items with feedback, need at least 5 to adjust threshold`);
    return;
  }
    
  // Get current threshold and learning rate
  const currentThreshold = getCurrentScoreThreshold();
  const learningRateStr = getPreference("learningRate", "0.05");
  const learningRate = parseFloat(learningRateStr);
    
  // Calculate adjustment based on false positive/negative ratio
  let adjustment = 0;
  if (falsePositives > falseNegatives) {
    // Too many false positives, increase threshold
    adjustment = Math.ceil((falsePositives - falseNegatives) * learningRate * 10);
  } else if (falseNegatives > falsePositives) {
    // Too many false negatives, decrease threshold
    adjustment = Math.floor((falsePositives - falseNegatives) * learningRate * 10);
  }
    
  // Apply adjustment if needed
  if (adjustment !== 0) {
    const newThreshold = Math.max(50, Math.min(95, currentThreshold + adjustment));
      
    // Update preference
    const prefsSheet = getUserPreferencesSheet();
    const prefsData = prefsSheet.getDataRange().getValues();
      
    for (let i = 1; i < prefsData.length; i++) {
      if (prefsData[i][0] === "scoreThreshold") {
        prefsSheet.getRange(i + 1, 2).setValue(newThreshold.toString());
        console.log(`Adjusted score threshold from ${currentThreshold} to ${newThreshold} based on feedback`);
        break;
      }
    }
  } else {
    console.log("No threshold adjustment needed based on current feedback");
  }
}

// Get recent life logs from the Limitless API
function getRecentLifeLogs() {
  const now = new Date();
  const startTime = new Date(now.getTime() - (HOURS_LOOKBACK * 60 * 60 * 1000));
    
  const options = {
    method: 'GET',
    headers: {
      'X-API-Key': LIMITLESS_API_KEY
    },
    muteHttpExceptions: true
  };
    
  try {
    // Get logs from the last HOURS_LOOKBACK hours
    const response = UrlFetchApp.fetch(`https://api.limitless.ai/v1/lifelogs?limit=20`, options);
    const responseData = JSON.parse(response.getContentText());
      
    if (responseData && responseData.data && responseData.data.lifelogs) {
      // Filter logs that are newer than our lookback time
      return responseData.data.lifelogs.filter(log => {
        const logDate = new Date(log.startTime);
        return logDate >= startTime;
      });
    }
    return [];
  } catch (error) {
    console.error("Error fetching life logs:", error);
    return [];
  }
}

// Use OpenAI to extract events and reminders from the log content
function extractEventsWithOpenAI(logContent) {
  const prompt = `
TASK DETECTION FRAMEWORK

1. PRIMARY INTENT INDICATORS (must satisfy at least 2):
  - ACTION VERBS: Contains explicit task verbs (buy, get, pick up, call, schedule, remember, send, pay, make, finish, complete, attend, bring, return, download, upload, submit, renew)
  - INTENT PHRASES: Contains clear intent markers ("I need to", "I should", "I have to", "Remember to", "Don't forget to", "Make sure to", "It's important to", "Reminder to")
  - TEMPORAL INDICATORS: Contains time references (today, tomorrow, tonight, this weekend, next week, on Monday, at 3PM, before dinner, after work, by EOD)
  - PRIORITY MARKERS: Contains urgency indicators (important, urgent, critical, ASAP, immediately, soon, deadline, priority)

2. CONTEXTUAL VALIDATION:
  - Examine 8 seconds of speech before and after the potential reminder
  - Reject if part of hypothetical scenarios ("If I were to...", "I might need to...")
  - Reject if clearly referring to someone else's tasks (unless prefaced with "Remind me to tell...")
  - Reject if negated ("I don't need to", "I already did", "I shouldn't")
  - Reject if part of a question ("Should I buy milk?")
  - Reject if clearly past tense reflection
  - Verify the statement isn't part of media consumption (podcast, video, audiobook)

3. CONFIDENCE SCORING ALGORITHM:
  - Base score: 40 points
  - Each primary indicator satisfied: +15 points
  - Clear, direct command ("Get milk"): +20 points
  - Contains specific details (brands, amounts, locations): +10 points
  - Contains repetition ("Remember to get milk, I need to get milk"): +15 points
  - Voice stress/emphasis detected on key parts: +10 points
  - Preceded by attention phrase ("Hey Dave, remember..."): +25 points
  - Contains uncertainty markers ("maybe", "might", "possibly"): -15 points
  - Detected as casual conversation: -20 points
  - THRESHOLD: Create reminder if score ≥ 80

4. DATA EXTRACTION:
  - TASK: Isolate the specific action item (what needs to be done)
  - TIMEFRAME: Extract any deadline or scheduling information
  - LOCATION: Identify any place-related information
  - PEOPLE: Note any individuals involved
  - PRIORITY: Assess urgency based on language and tone
  - DETAILS: Capture quantities, specifications, or other relevant information
  - DEFAULT DATE: If no specific date/time is mentioned in the task, automatically set the due date to today

5. FALSE POSITIVE FILTERS:
  - Evaluate if statement refers to habitual behavior rather than specific task
  - Check if statement is expressing preference rather than intention
  - Reject common phrases that sound like tasks but aren't ("I need to say", "I have to admit")
  - Filter out rhetorical "reminders" ("I need to remember how good this feels")
  - Ignore statements about general facts ("I need to drink more water")
  - Distinguish between desires and actual intentions ("I would love to visit Paris")

OUTPUT FORMAT:
Respond ONLY with a valid JSON array of events. If no events/reminders are found, return an empty array [].

Example response:
[
 {
   "title": "Dental Checkup",
   "date": "2023-11-14",
   "time": "15:30",
   "isReminder": false,
   "details": "Scheduled dental appointment"
 },
 {
   "title": "Buy milk",
   "date": "today",
   "time": "",
   "isReminder": true,
   "details": "Need to get milk from the store"
 },
 {
   "title": "Follow up about: Eye Exam",
   "date": "today",
   "time": "",
   "isReminder": true,
   "details": "Need to confirm date and time for upcoming eye exam"
 }
]

TRANSCRIPT TO ANALYZE:
${logContent}

IMPORTANT: Based on this transcript, what events or reminders should be added to a calendar? Respond with a JSON array only.
`;

  const options = {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${OPENAI_API_KEY}`
    },
    payload: JSON.stringify({
      model: "gpt-3.5-turbo",
      messages: [
        {
          role: "system",
          content: "You are a helpful assistant that extracts calendar events and reminders from text. Only respond with valid JSON."
        },
        {
          role: "user",
          content: prompt
        }
      ],
      temperature: 0.1,
      max_tokens: 2048
    }),
    muteHttpExceptions: true
  };

  try {
    const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
    const responseCode = response.getResponseCode();
    console.log(`OpenAI API response code: ${responseCode}`);
      
    const responseText = response.getContentText();
    console.log("OpenAI response received. Length: " + responseText.length);
      
    // Check for API errors
    if (responseCode !== 200) {
      console.error("OpenAI API Error:", responseText);
      return [];
    }
      
    const responseData = JSON.parse(responseText);
      
    if (responseData && responseData.choices && responseData.choices.length > 0) {
      const text = responseData.choices[0].message.content;
      console.log("Raw OpenAI response:", text);
        
      try {
        // Try to parse the JSON directly
        const parsedEvents = JSON.parse(text);
        console.log(`Successfully parsed ${parsedEvents.length} events`);
        return parsedEvents;
      } catch (e) {
        // If direct parsing fails, try to extract JSON from the text
        const jsonMatch = text.match(/\[\s*\{.*\}\s*\]/s);
        if (jsonMatch) {
          console.log("JSON pattern found in response");
          try {
            const parsedEvents = JSON.parse(jsonMatch[0]);
            console.log(`Successfully parsed ${parsedEvents.length} events from pattern`);
            return parsedEvents;
          } catch (e2) {
            console.error("Failed to parse JSON from pattern:", e2);
            return [];
          }
        } else {
          console.log("No JSON array found in OpenAI response");
          return [];
        }
      }
    }
      
    return [];
  } catch (error) {
    console.error("Error calling OpenAI API:", error);
    console.error("Error details:", error.toString());
    return [];
  }
}
// Function to test the script with a sample log
function testWithSampleLog() {
  const sampleLog = {
    id: "test-log",
    title: "Test Log",
    startTime: new Date().toISOString(),
    markdown: `
      Speaker 1: We have you booked in for the 14th of November at 3:30 PM for your dental checkup.
        
      Speaker 2: Great, thanks for confirming. I'll make sure to be there.
        
      Speaker 1: Don't forget to get milk today, we're completely out and I need it for breakfast tomorrow.
        
      Speaker 2: Okay, I'll pick some up from the store on my way home.
        
      Speaker 1: Also, remember we have dinner with the Johnsons next Friday at 7:00 PM.
    `
  };
    
  console.log("Testing with sample log...");
    
  // Make sure we have access to Tasks API
  try {
    // Enable the Tasks API
    enableTasksService();
      
    // Initialize scoring system
    initializeScoring();
      
    // First, verify the OpenAI connection
    console.log("First, testing API connection...");
    if (!testOpenAIConnection()) {
      console.log("API connection test failed. Please check your OpenAI API key.");
      return;
    }
      
    console.log("Now extracting events from log...");
    const events = extractEventsWithOpenAI(sampleLog.markdown);
      
    if (events && events.length > 0) {
      console.log(`Found ${events.length} events in sample log:`, events);
        
      // Score each event
      const scoredEvents = [];
      for (const event of events) {
        const scoringResult = scoreEvent(event, sampleLog.markdown);
        console.log(`Event "${event.title}" scored ${scoringResult.totalScore}`);
          
        // Record the score in history for learning
        recordEventScore(event, scoringResult, sampleLog.id);
          
        // Only process events that meet the threshold
        if (scoringResult.totalScore >= getCurrentScoreThreshold()) {
          scoredEvents.push({
            ...event,
            score: scoringResult.totalScore,
            scoringFactors: scoringResult.factors
          });
        }
      }
        
      console.log(`After scoring, ${scoredEvents.length} events met the threshold.`);
        
      // Process the events
      console.log("Testing adding events to sheet for approval...");
      processEvents(scoredEvents, sampleLog);
      console.log("Test completed successfully!");
        
      // Get and print the spreadsheet URL for easy access
      try {
        const url = getApprovalSpreadsheet().getUrl();
        console.log("=============================================");
        console.log(`Approval spreadsheet URL: ${url}`);
        console.log("Open this URL to see and approve/reject items");
        console.log("=============================================");
      } catch (e) {
        console.error("Could not get spreadsheet URL:", e);
      }
    } else {
      console.log("No events found in sample log or API error occurred.");
    }
  } catch (e) {
    console.error("Error testing:", e);
    console.log("Please enable the Google Tasks API manually in Resources > Advanced Google services...");
  }
}

// Enable Tasks API service
function enableTasksService() {
  // Check if Tasks API is already enabled
  try {
    Tasks.Tasklists.list();
    console.log("Tasks API is already enabled.");
  } catch (e) {
    console.log("Tasks API is not enabled. You need to enable it manually.");
    console.log("Go to Resources > Advanced Google services... and enable the Tasks API.");
    throw new Error("Tasks API not enabled");
  }
}

// Simple function to test just the OpenAI API connection
function testOpenAIConnection() {
  console.log("Testing OpenAI API connection...");
    
  const options = {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${OPENAI_API_KEY}`
    },
    payload: JSON.stringify({
      model: "gpt-3.5-turbo",
      messages: [
        {
          role: "user",
          content: "Say 'API connection successful'"
        }
      ],
      max_tokens: 50
    }),
    muteHttpExceptions: true
  };
    
  try {
    const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
    const responseCode = response.getResponseCode();
    console.log(`OpenAI API response code: ${responseCode}`);
      
    const responseText = response.getContentText();
    console.log("Response:", responseText);
      
    if (responseCode === 200) {
      console.log("✅ OpenAI API connection is working!");
      return true;
    } else {
      console.log("❌ OpenAI API connection failed");
      return false;
    }
  } catch (error) {
    console.error("❌ Error testing OpenAI API connection:", error);
    return false;
  }
}

// Function to create triggers for automatic execution
function setupTriggers() {
  // Clear any existing triggers
  const triggers = ScriptApp.getProjectTriggers();
  for (const trigger of triggers) {
    ScriptApp.deleteTrigger(trigger);
  }
    
  // Create trigger to run processLifeLogs every 5 minutes
  ScriptApp.newTrigger('processLifeLogs')
    .timeBased()
    .everyMinutes(5)
    .create();
    
  // Create trigger to run processPendingApprovals every hour
  ScriptApp.newTrigger('processPendingApprovals')
    .timeBased()
    .everyHours(1)
    .create();
    
  console.log("Triggers set up successfully!");
}

// Function to clear all pending items (for testing)
function clearPendingItems() {
  try {
    const sheet = getApprovalSheet();
    const lastRow = sheet.getLastRow();
      
    if (lastRow > 1) {
      // Clear all data except headers
      sheet.deleteRows(2, lastRow - 1);
    }
      
    console.log("All pending items cleared!");
  } catch (e) {
    console.error("Error clearing pending items:", e);
  }
}

// Function to view approval spreadsheet
function getApprovalSpreadsheetUrl() {
  try {
    const spreadsheet = getApprovalSpreadsheet();
    const url = spreadsheet.getUrl();
    console.log("Approval spreadsheet URL:", url);
    return url;
  } catch (e) {
    console.error("Error getting approval spreadsheet:", e);
    return null;
  }
}

// Function to reset approval process (for testing)
function resetApprovalSystem() {
  // Clear the stored spreadsheet ID
  PropertiesService.getScriptProperties().deleteProperty('APPROVAL_SPREADSHEET_ID');
  console.log("Approval system reset - will create a new spreadsheet on next run");
}

// Function to reset processed logs tracking (to force reprocessing)
function resetProcessedLogsTracking() {
  try {
    // Get the processed logs sheet
    const sheet = getProcessedLogsSheet();
    const lastRow = sheet.getLastRow();
      
    if (lastRow > 1) {
      // Clear all data except headers
      sheet.deleteRows(2, lastRow - 1);
    }
      
    console.log("Processed logs tracking reset - logs will be reprocessed on next run");
  } catch (e) {
    console.error("Error resetting processed logs tracking:", e);
  }
}

// Function to reset scoring history (for testing)
function resetScoringHistory() {
  try {
    const spreadsheet = getApprovalSpreadsheet();
    let sheet = spreadsheet.getSheetByName(SCORING_HISTORY_SHEET_NAME);
      
    if (sheet) {
      const lastRow = sheet.getLastRow();
      if (lastRow > 1) {
        // Clear all data except headers
        sheet.deleteRows(2, lastRow - 1);
      }
      console.log("Scoring history reset");
    } else {
      console.log("No scoring history sheet found");
    }
  } catch (e) {
    console.error("Error resetting scoring history:", e);
  }
}

// Function to manually approve an item by ID
function manuallyApproveItem(itemId) {
  try {
    const sheet = getApprovalSheet();
    const data = sheet.getDataRange().getValues();
    const headers = data[0];
      
    const idIndex = headers.indexOf("ID");
    const statusIndex = headers.indexOf("Status");
      
    for (let i = 1; i < data.length; i++) {
      if (data[i][idIndex] === itemId) {
        // Update status to "approved"
        sheet.getRange(i + 1, statusIndex + 1).setValue("approved");
        console.log(`Item ${itemId} marked as approved`);
        return true;
      }
    }
      
    console.log(`Item ${itemId} not found`);
    return false;
  } catch (e) {
    console.error("Error manually approving item:", e);
    return false;
  }
}

// Function to manually update user feedback for an item
function provideFeedback(itemId, feedback) {
  try {
    const sheet = getApprovalSheet();
    const data = sheet.getDataRange().getValues();
    const headers = data[0];
      
    const idIndex = headers.indexOf("ID");
    const feedbackIndex = headers.indexOf("Feedback");
      
    if (feedbackIndex === -1) {
      console.log("Feedback column not found, add it first");
      updateApprovalSheetWithScoring(); // This adds the feedback column
      return provideFeedback(itemId, feedback); // Retry after adding column
    }
      
    for (let i = 1; i < data.length; i++) {
      if (data[i][idIndex] === itemId) {
        // Update feedback
        sheet.getRange(i + 1, feedbackIndex + 1).setValue(feedback);
        console.log(`Feedback for item ${itemId} set to "${feedback}"`);
          
        // Process feedback for learning if enabled
        if (LEARNING_ENABLED) {
          processFeedbackForLearning(itemId, feedback);
        }
          
        return true;
      }
    }
      
    console.log(`Item ${itemId} not found`);
    return false;
  } catch (e) {
    console.error("Error providing feedback:", e);
    return false;
  }
}

// Function to view current score threshold
function viewScoreThreshold() {
  const threshold = getCurrentScoreThreshold();
  console.log(`Current score threshold: ${threshold}`);
  return threshold;
}

// Function to manually adjust score threshold
function setScoreThreshold(newThreshold) {
  try {
    // Validate input
    const threshold = parseInt(newThreshold);
    if (isNaN(threshold) || threshold < 0 || threshold > 100) {
      console.error("Invalid threshold value. Must be a number between 0 and 100.");
      return false;
    }
      
    // Get the preferences sheet
    const sheet = getUserPreferencesSheet();
    const data = sheet.getDataRange().getValues();
      
    // Find the threshold preference
    let found = false;
    for (let i = 1; i < data.length; i++) {
      if (data[i][0] === "scoreThreshold") {
        sheet.getRange(i + 1, 2).setValue(threshold.toString());
        found = true;
        break;
      }
    }
      
    // If not found, add it
    if (!found) {
      sheet.appendRow(["scoreThreshold", threshold.toString()]);
    }
      
    console.log(`Score threshold updated to ${threshold}`);
    return true;
  } catch (e) {
    console.error("Error setting score threshold:", e);
    return false;
  }
}

// Function to view scoring statistics
function viewScoringStats() {
  try {
    const sheet = getScoringHistorySheet();
    const data = sheet.getDataRange().getValues();
      
    if (data.length <= 1) {
      console.log("No scoring history available");
      return null;
    }
      
    const headers = data[0];
    const scoreIndex = headers.indexOf("TotalScore");
    const typeIndex = headers.indexOf("EventType");
    const feedbackIndex = headers.indexOf("UserFeedback");
    const passedThresholdIndex = headers.indexOf("PassedThreshold");
      
    // Calculate statistics
    let totalItems = data.length - 1;
    let totalScore = 0;
    let maxScore = 0;
    let minScore = 1000;
    let tasksCount = 0;
    let eventsCount = 0;
    let passedThresholdCount = 0;
    let feedbackCount = 0;
    let correctFeedbackCount = 0;
    let incorrectFeedbackCount = 0;
      
    for (let i = 1; i < data.length; i++) {
      const score = parseFloat(data[i][scoreIndex]);
      totalScore += score;
        
      if (score > maxScore) maxScore = score;
      if (score < minScore) minScore = score;
        
      if (data[i][typeIndex] === "Task") tasksCount++;
      else eventsCount++;
        
      if (data[i][passedThresholdIndex] === "true") passedThresholdCount++;
        
      if (data[i][feedbackIndex]) {
        feedbackCount++;
        if (data[i][feedbackIndex] === "correct") correctFeedbackCount++;
        else if (data[i][feedbackIndex] === "should_not_have_included" || 
                 data[i][feedbackIndex] === "missed_important") {
          incorrectFeedbackCount++;
        }
      }
    }
      
    const avgScore = totalScore / totalItems;
      
    // Return statistics
    const stats = {
      totalItems: totalItems,
      averageScore: avgScore.toFixed(2),
      maxScore: maxScore,
      minScore: minScore,
      tasks: tasksCount,
      events: eventsCount,
      passedThreshold: passedThresholdCount,
      withFeedback: feedbackCount,
      correctFeedback: correctFeedbackCount,
      incorrectFeedback: incorrectFeedbackCount,
      currentThreshold: getCurrentScoreThreshold()
    };
      
    console.log("Scoring Statistics:", stats);
    return stats;
  } catch (e) {
    console.error("Error calculating scoring statistics:", e);
    return null;
  }
}
```
5. Replace the API keys at the top of the script with your own
6. Save the project
7. Enable the necessary Google Services:
   - Go to **Resources > Advanced Google Services**
   - Enable the **Google Calendar API**
   - Enable the **Google Tasks API**
8. Run the `testWithSampleLog` function to test the setup
9. Run the `setupTriggers` function to set up automatic processing

## Configuration

At the top of the script, you can customize various settings:

```javascript
// Example configuration (replace with actual code during setup)
const HOURS_LOOKBACK = 24; // How many hours back to check for logs
const CALENDAR_NAME = "Life Logs Appointments"; // Name of the Google Calendar
const DEFAULT_REMINDER_MINUTES = 30; // Default reminder time
const ENABLE_APPROVAL_WORKFLOW = true; // Enable/disable approval workflow
```

## Scoring System

The script uses a sophisticated scoring system to determine which potential events should be added to your calendar. Each event starts with a base score and gains or loses points based on various factors:

| Factor | Weight | Description |
|--------|--------|-------------|
| Action Verbs | +15 | Contains action words like "buy," "call" |
| Intent Phrases | +15 | Contains phrases like "I need to," "don't forget" |
| Temporal Indicators | +15 | Contains time references like "tomorrow," "at 3pm" |
| Priority Markers | +15 | Contains urgency words like "important," "deadline" |
| Direct Command | +20 | Starts with an action verb |
| Specific Details | +10 | Contains specific details about the task |
| Repetition | +15 | Action is mentioned multiple times |
| Voice Emphasis | +10 | Contains ALL CAPS or exclamation marks |
| Attention Phrase | +25 | Contains attention-grabbing phrases |
| Uncertainty Markers | -15 | Contains uncertain language like "maybe," "might" |
| Casual Conversation | -20 | Contains casual conversation markers |

Events that score above the threshold (default: 80) are considered valid tasks or appointments.

## Advanced Usage

### Testing

- `testWithSampleLog()`: Tests the entire workflow with a sample log
- `testOpenAIConnection()`: Tests your OpenAI API connection

### Maintenance

- `clearPendingItems()`: Clears all pending items from the approval sheet
- `resetApprovalSystem()`: Resets the approval system to create a new spreadsheet
- `resetProcessedLogsTracking()`: Force reprocessing of previously processed logs
- `resetScoringHistory()`: Clears the scoring history

### Customization

- `setScoreThreshold(threshold)`: Adjust the scoring threshold
- `viewScoreThreshold()`: View the current threshold
- `viewScoringStats()`: View statistics about the scoring system

## Learning System

The script includes a machine learning component that adjusts the scoring threshold based on your feedback:

1. When you indicate that an item was incorrectly included, the threshold increases
2. When you indicate that an important item was missed, the threshold decreases
3. The learning rate controls how quickly the system adapts to your feedback

## Troubleshooting

- **API Connection Issues**: Run `testOpenAIConnection()` to check your API keys
- **Missing Events**: Check your score threshold with `viewScoreThreshold()` and consider lowering it
- **Too Many False Positives**: Consider increasing the score threshold
- **Tasks API Error**: Make sure you've enabled the Google Tasks API in Advanced Google Services

## Contributing

Feel free to submit issues or pull requests to improve the functionality.

## License

[MIT License](LICENSE)

## Acknowledgments

- [Limitless AI](https://limitless.ai) for the Life Logs API
- [OpenAI](https://openai.com) for the natural language processing capabilities
